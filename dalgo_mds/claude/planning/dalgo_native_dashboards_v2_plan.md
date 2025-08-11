# Dalgo Native Dashboards v2 Implementation Plan

## Executive Summary

This document provides a comprehensive plan for implementing native dashboards v2 for the Dalgo platform. The feature will allow users to create dynamic dashboards by combining existing charts with drag-and-drop functionality, filters, and real-time collaboration features. The implementation leverages existing patterns in the codebase while introducing new components for dashboard management.

**Confidence Score: 9/10** - High confidence due to existing infrastructure and clear patterns to follow.

## Feature Overview

### Core Requirements
1. **Dashboard Management**: Create, edit, delete, and view native dashboards
2. **Grid Layout System**: Flexible grid-based layout with drag-and-drop
3. **Component Types**: Charts, text blocks, and headings
4. **Filtering System**: Dashboard-level filters (categorical and numerical)
5. **Auto-save**: Google Docs-style automatic saving every 5 seconds
6. **Locking Mechanism**: Prevent concurrent editing
7. **Full-screen Mode**: Enhanced viewing experience
8. **Integration**: Coexist with existing Superset dashboards
9. **Undo/Redo**: Max 20 undo operations for dashboard actions

## Architecture Design

### High-Level Flow

```
Frontend (webapp_v2)                    Backend (DDP_backend)
     │                                          │
     ├─ Dashboard List ──────────────────────> GET /api/dashboards/
     │                                          │
     ├─ Dashboard Builder                       │
     │   ├─ Drag & Drop Components             │
     │   ├─ Auto-save (5s debounce) ────────> PUT /api/dashboards/{id}/
     │   ├─ Lock Dashboard ──────────────────> POST /api/dashboards/{id}/lock/
     │   └─ Undo/Redo (local state)           │
     │                                          │
     ├─ Dashboard Viewer                        │
     │   ├─ Render Components                   │
     │   └─ Apply Filters ───────────────────> POST /api/dashboards/{id}/filter-data/
     │                                          │
     ├─ Chart Selection Modal                   │
     │   ├─ Search Charts ───────────────────> GET /api/charts/?search=...
     │   └─ Preview Charts                      │
     │                                          │
     └─ Filter Configuration ─────────────────> GET /api/dashboards/filter-options/
```

## Backend Implementation

### 1. Database Models with Enums

#### Enums (`ddpui/models/dashboard.py`)
```python
from enum import Enum
from django.db import models
from django.contrib.postgres.fields import ArrayField
from ddpui.models.org import Org
from ddpui.models.org_user import OrgUser

class DashboardType(str, Enum):
    """Dashboard type enum"""
    NATIVE = "native"
    SUPERSET = "superset"
    
    @classmethod
    def choices(cls):
        return [(key.value, key.name) for key in cls]

class DashboardComponentType(str, Enum):
    """Dashboard component types"""
    CHART = "chart"
    TEXT = "text"
    HEADING = "heading"
    
    @classmethod
    def choices(cls):
        return [(key.value, key.name) for key in cls]

class DashboardFilterType(str, Enum):
    """Dashboard filter types"""
    VALUE = "value"
    NUMERICAL = "numerical"
    
    @classmethod
    def choices(cls):
        return [(key.value, key.name) for key in cls]
```

#### Dashboard Model
```python
class Dashboard(models.Model):
    """Native dashboard configuration model"""
    
    id = models.BigAutoField(primary_key=True)
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True, null=True)
    dashboard_type = models.CharField(
        max_length=20, 
        choices=DashboardType.choices(), 
        default=DashboardType.NATIVE.value
    )
    
    # Grid configuration
    grid_columns = models.IntegerField(default=12)
    
    # Layout configuration stored as JSON
    layout_config = models.JSONField(
        default=dict, 
        help_text="Grid layout positions and sizes"
    )
    
    # Components configuration
    components = models.JSONField(
        default=dict,
        help_text="Dashboard components configuration"
    )
    
    # Publishing status
    is_published = models.BooleanField(default=False)
    published_at = models.DateTimeField(null=True, blank=True)
    
    # Metadata
    created_by = models.ForeignKey(
        OrgUser, 
        on_delete=models.CASCADE, 
        db_column="created_by"
    )
    org = models.ForeignKey(Org, on_delete=models.CASCADE)
    last_modified_by = models.ForeignKey(
        OrgUser,
        on_delete=models.CASCADE,
        db_column="last_modified_by",
        null=True,
        related_name="dashboards_modified",
    )
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return f"{self.title} ({self.dashboard_type})"
```

#### DashboardFilter Model
```python
class DashboardFilter(models.Model):
    """Dashboard filter configuration"""
    
    id = models.BigAutoField(primary_key=True)
    dashboard = models.ForeignKey(
        Dashboard, 
        on_delete=models.CASCADE, 
        related_name="filters"
    )
    
    # Filter configuration
    filter_type = models.CharField(
        max_length=20,
        choices=DashboardFilterType.choices()
    )
    schema_name = models.CharField(max_length=255)
    table_name = models.CharField(max_length=255)
    column_name = models.CharField(max_length=255)
    
    # Filter settings
    settings = models.JSONField(
        default=dict,
        help_text="Filter-specific settings"
    )
    
    # UI positioning
    order = models.IntegerField(default=0)
    
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
```

#### DashboardLock Model (following TaskLock pattern)
```python
class DashboardLock(models.Model):
    """Locking implementation for Dashboard editing"""
    
    dashboard = models.OneToOneField(
        Dashboard, 
        on_delete=models.CASCADE, 
        related_name="lock"
    )
    locked_at = models.DateTimeField(auto_now_add=True)
    locked_by = models.ForeignKey(OrgUser, on_delete=models.CASCADE)
    lock_token = models.CharField(max_length=36, unique=True)
    expires_at = models.DateTimeField()
    
    class Meta:
        db_table = "dashboard_lock"
```

### 2. API Implementation

#### Dashboard API (`ddpui/api/dashboard_native_api.py`)
```python
from ninja import Router, Schema
from typing import Optional, List
from datetime import datetime, timedelta
import uuid

dashboard_native_router = Router()

# Schemas
class DashboardCreate(Schema):
    title: str
    description: Optional[str] = None
    grid_columns: int = 12

class DashboardUpdate(Schema):
    title: Optional[str] = None
    description: Optional[str] = None
    layout_config: Optional[dict] = None
    components: Optional[dict] = None
    is_published: Optional[bool] = None

class DashboardResponse(Schema):
    id: int
    title: str
    dashboard_type: str
    is_locked: bool
    locked_by: Optional[str] = None
    # ... other fields

# Endpoints
@dashboard_native_router.get("/", response=List[DashboardResponse])
@has_permission(["can_view_dashboards"])
def list_dashboards(request, dashboard_type: Optional[str] = None):
    """List all dashboards with optional type filter"""
    # Implementation

@dashboard_native_router.post("/", response=DashboardResponse)
@has_permission(["can_create_dashboards"])
def create_dashboard(request, payload: DashboardCreate):
    """Create a new dashboard"""
    # Implementation

@dashboard_native_router.put("/{dashboard_id}/", response=DashboardResponse)
@has_permission(["can_edit_dashboards"])
def update_dashboard(request, dashboard_id: int, payload: DashboardUpdate):
    """Update dashboard with auto-save support"""
    # Check lock status
    # Update dashboard
    # Return updated dashboard

@dashboard_native_router.post("/{dashboard_id}/lock/")
@has_permission(["can_edit_dashboards"])
def lock_dashboard(request, dashboard_id: int):
    """Lock dashboard for editing"""
    # Create or refresh lock
    # Return lock token

@dashboard_native_router.delete("/{dashboard_id}/lock/")
@has_permission(["can_edit_dashboards"])
def unlock_dashboard(request, dashboard_id: int):
    """Unlock dashboard"""
    # Release lock
```

### 3. Service Layer

#### Dashboard Service (`ddpui/services/dashboard_service.py`)
```python
from ddpui.models.dashboard import DashboardFilterType, DashboardComponentType

class DashboardService:
    @staticmethod
    def apply_filters(dashboard_id: int, filters: dict) -> dict:
        """Apply filters to all chart components in dashboard"""
        # Get dashboard configuration
        # For each chart component:
        #   - Apply filters to chart query
        #   - Return filtered data
        
    @staticmethod
    def check_lock_status(dashboard_id: int) -> dict:
        """Check if dashboard is locked for editing"""
        # Check DashboardLock table
        # Return lock status and user info
        
    @staticmethod
    def generate_filter_options(schema: str, table: str, column: str) -> list:
        """Generate filter options for categorical filters"""
        # Query distinct values from warehouse
        # Cache in Redis
        # Return options
```

## Frontend Implementation

### 1. Undo/Redo Implementation

```typescript
// webapp_v2/hooks/useUndoRedo.ts
interface HistoryState<T> {
  past: T[];
  present: T;
  future: T[];
}

export function useUndoRedo<T>(initialState: T, maxHistory: number = 20) {
  const [history, setHistory] = useState<HistoryState<T>>({
    past: [],
    present: initialState,
    future: [],
  });

  const setState = useCallback((newState: T) => {
    setHistory((prev) => ({
      past: [...prev.past.slice(-maxHistory + 1), prev.present],
      present: newState,
      future: [],
    }));
  }, [maxHistory]);

  const undo = useCallback(() => {
    setHistory((prev) => {
      if (prev.past.length === 0) return prev;
      
      const previous = prev.past[prev.past.length - 1];
      const newPast = prev.past.slice(0, prev.past.length - 1);
      
      return {
        past: newPast,
        present: previous,
        future: [prev.present, ...prev.future],
      };
    });
  }, []);

  const redo = useCallback(() => {
    setHistory((prev) => {
      if (prev.future.length === 0) return prev;
      
      const next = prev.future[0];
      const newFuture = prev.future.slice(1);
      
      return {
        past: [...prev.past, prev.present],
        present: next,
        future: newFuture,
      };
    });
  }, []);

  return {
    state: history.present,
    setState,
    undo,
    redo,
    canUndo: history.past.length > 0,
    canRedo: history.future.length > 0,
  };
}
```

### 2. Chart Selection Modal

```typescript
// webapp_v2/components/dashboard/chart-selector-modal.tsx
import { useState } from 'react';
import { Dialog, DialogContent, DialogHeader } from '@/components/ui/dialog';
import { Input } from '@/components/ui/input';
import { ScrollArea } from '@/components/ui/scroll-area';
import { useCharts } from '@/hooks/api/useCharts';
import { MiniChart } from '@/components/charts/MiniChart';

interface ChartSelectorModalProps {
  open: boolean;
  onClose: () => void;
  onSelect: (chartId: number) => void;
}

export function ChartSelectorModal({ open, onClose, onSelect }: ChartSelectorModalProps) {
  const [search, setSearch] = useState('');
  const { data: charts, isLoading } = useCharts({ search });

  const handleSelect = (chartId: number) => {
    onSelect(chartId);
    onClose();
  };

  return (
    <Dialog open={open} onOpenChange={onClose}>
      <DialogContent className="max-w-4xl max-h-[80vh]">
        <DialogHeader>
          <h2 className="text-xl font-semibold">Select a Chart</h2>
        </DialogHeader>
        
        <div className="space-y-4">
          <Input
            placeholder="Search charts..."
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            className="w-full"
          />
          
          <ScrollArea className="h-[500px]">
            {isLoading ? (
              <div className="text-center py-8">Loading charts...</div>
            ) : (
              <div className="grid grid-cols-2 gap-4">
                {charts?.map((chart) => (
                  <div
                    key={chart.id}
                    className="border rounded-lg p-4 cursor-pointer hover:bg-gray-50 transition-colors"
                    onClick={() => handleSelect(chart.id)}
                  >
                    <h3 className="font-medium mb-2">{chart.title}</h3>
                    <div className="h-32">
                      <MiniChart chartId={chart.id} />
                    </div>
                    <p className="text-sm text-gray-500 mt-2">
                      {chart.chart_type} • {chart.schema_name}.{chart.table_name}
                    </p>
                  </div>
                ))}
              </div>
            )}
          </ScrollArea>
        </div>
      </DialogContent>
    </Dialog>
  );
}
```

### 3. Enhanced Dashboard Builder with Undo/Redo

```typescript
// webapp_v2/components/dashboard/dashboard-builder-v2.tsx
import GridLayout from 'react-grid-layout';
import 'react-grid-layout/css/styles.css';
import 'react-resizable/css/styles.css';
import { useUndoRedo } from '@/hooks/useUndoRedo';
import { ChartSelectorModal } from './chart-selector-modal';

interface DashboardState {
  layout: DashboardLayout[];
  components: DashboardComponent[];
}

export function DashboardBuilderV2() {
  const {
    state,
    setState,
    undo,
    redo,
    canUndo,
    canRedo
  } = useUndoRedo<DashboardState>({
    layout: [],
    components: []
  }, 20);

  const [isDragging, setIsDragging] = useState(false);
  const [showChartSelector, setShowChartSelector] = useState(false);
  const [pendingChartPosition, setPendingChartPosition] = useState<{x: number, y: number} | null>(null);
  
  // Auto-save with debounce
  const debouncedSave = useMemo(
    () => debounce(saveDashboard, 5000),
    []
  );
  
  // Lock management
  useEffect(() => {
    lockDashboard();
    return () => unlockDashboard();
  }, []);

  // Keyboard shortcuts for undo/redo
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if ((e.metaKey || e.ctrlKey) && e.key === 'z' && !e.shiftKey) {
        e.preventDefault();
        if (canUndo) undo();
      } else if ((e.metaKey || e.ctrlKey) && (e.key === 'y' || (e.key === 'z' && e.shiftKey))) {
        e.preventDefault();
        if (canRedo) redo();
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    return () => window.removeEventListener('keydown', handleKeyDown);
  }, [undo, redo, canUndo, canRedo]);

  const handleLayoutChange = (newLayout: GridLayout.Layout[]) => {
    setState({
      ...state,
      layout: newLayout
    });
    debouncedSave(state);
  };

  const handleAddChart = () => {
    setShowChartSelector(true);
  };

  const handleChartSelected = async (chartId: number) => {
    // Fetch chart details
    const chartDetails = await fetchChart(chartId);
    
    // Add to dashboard
    const newComponent: DashboardComponent = {
      id: `chart-${Date.now()}`,
      type: DashboardComponentType.CHART,
      config: {
        chartId,
        ...chartDetails
      }
    };

    const newLayoutItem: DashboardLayout = {
      i: newComponent.id,
      x: 0,
      y: 0,
      w: 4,
      h: 8,
      minW: 2,
      maxW: 12,
      minH: 4
    };

    setState({
      layout: [...state.layout, newLayoutItem],
      components: [...state.components, newComponent]
    });
  };

  return (
    <div className="dashboard-builder">
      <div className="toolbar flex gap-2 p-4 border-b">
        <Button onClick={handleAddChart}>
          <Plus className="w-4 h-4 mr-2" />
          Add Chart
        </Button>
        
        <Button onClick={undo} disabled={!canUndo}>
          <Undo className="w-4 h-4" />
        </Button>
        
        <Button onClick={redo} disabled={!canRedo}>
          <Redo className="w-4 h-4" />
        </Button>
        
        <div className="ml-auto flex items-center gap-2">
          {debouncedSave.pending && (
            <span className="text-sm text-gray-500">Saving...</span>
          )}
        </div>
      </div>

      <GridLayout
        className="layout"
        layout={state.layout}
        cols={12}
        rowHeight={30}
        width={1200}
        onLayoutChange={handleLayoutChange}
        draggableHandle=".drag-handle"
        compactType={null}
        preventCollision={false}
      >
        {renderComponents(state.components)}
      </GridLayout>

      <ChartSelectorModal
        open={showChartSelector}
        onClose={() => setShowChartSelector(false)}
        onSelect={handleChartSelected}
      />
    </div>
  );
}
```

### 4. Component Palette Enhancement

```typescript
// webapp_v2/components/dashboard/component-palette.tsx
export function ComponentPalette({ onAddComponent }: ComponentPaletteProps) {
  const [showChartSelector, setShowChartSelector] = useState(false);

  const components = [
    {
      type: DashboardComponentType.CHART,
      icon: BarChart3,
      label: 'Chart',
      description: 'Add existing chart',
      action: () => setShowChartSelector(true)
    },
    {
      type: DashboardComponentType.TEXT,
      icon: Type,
      label: 'Text',
      description: 'Add text block',
      action: () => onAddComponent(DashboardComponentType.TEXT)
    },
    {
      type: DashboardComponentType.HEADING,
      icon: Heading,
      label: 'Heading',
      description: 'Add heading',
      action: () => onAddComponent(DashboardComponentType.HEADING)
    }
  ];

  return (
    <>
      <div className="component-palette p-4 border-r">
        <h3 className="font-semibold mb-4">Components</h3>
        <div className="space-y-2">
          {components.map((comp) => (
            <div
              key={comp.type}
              className="p-3 border rounded cursor-pointer hover:bg-gray-50"
              onClick={comp.action}
            >
              <comp.icon className="w-5 h-5 mb-1" />
              <div className="font-medium">{comp.label}</div>
              <div className="text-xs text-gray-500">{comp.description}</div>
            </div>
          ))}
        </div>
      </div>

      <ChartSelectorModal
        open={showChartSelector}
        onClose={() => setShowChartSelector(false)}
        onSelect={(chartId) => {
          onAddComponent(DashboardComponentType.CHART, { chartId });
          setShowChartSelector(false);
        }}
      />
    </>
  );
}
```

## Implementation Steps

### Phase 1: Backend Foundation (2-3 days)
1. Create Django models with proper enums and migrations
2. Implement basic CRUD API endpoints
3. Set up dashboard locking mechanism
4. Create filter options API

### Phase 2: Frontend Grid System (3-4 days)
1. Install and configure react-grid-layout
2. Create dashboard builder component with undo/redo
3. Implement drag-and-drop functionality
4. Add component palette with chart selector

### Phase 3: Chart Selection Flow (2 days)
1. Create chart selector modal with search
2. Implement chart preview in selector
3. Add chart to dashboard flow
4. Handle chart data fetching

### Phase 4: Filter System (2-3 days)
1. Create filter configuration UI
2. Implement value and numerical filters
3. Connect filters to chart data queries
4. Add filter preview in builder

### Phase 5: Auto-save & Locking (2 days)
1. Implement auto-save with debouncing
2. Add save status indicators
3. Create lock management UI
4. Handle lock expiration

### Phase 6: Integration & Polish (2-3 days)
1. Integrate with existing dashboard list
2. Add full-screen mode
3. Polish undo/redo functionality
4. Add responsive design

### Phase 7: Testing & Refinement (2 days)
1. Write unit tests for backend APIs
2. Create component tests for frontend
3. Integration testing
4. Performance optimization

## Testing Strategy

### Backend Tests
```python
# ddpui/tests/api_tests/test_dashboard_native_api.py
class TestDashboardNativeAPI(TestCase):
    def test_create_dashboard(self):
        # Test dashboard creation
    
    def test_dashboard_locking(self):
        # Test concurrent editing prevention
    
    def test_auto_save(self):
        # Test auto-save functionality
    
    def test_filter_application(self):
        # Test filter data queries
    
    def test_enum_usage(self):
        # Test that enums work correctly
```

### Frontend Tests
```typescript
// webapp_v2/components/dashboard/__tests__/dashboard-builder.test.tsx
describe('DashboardBuilder', () => {
  it('should handle drag and drop', () => {
    // Test grid layout changes
  });
  
  it('should auto-save after 5 seconds', () => {
    // Test debounced save
  });
  
  it('should show lock indicator', () => {
    // Test concurrent editing UI
  });
  
  it('should handle undo/redo operations', () => {
    // Test undo/redo functionality with 20 item limit
  });
  
  it('should open chart selector and add chart', () => {
    // Test chart selection flow
  });
});
```

## Key Implementation Details

### Undo/Redo System
- Implemented using a custom React hook that maintains history state
- Stores up to 20 previous states as specified
- Keyboard shortcuts: Cmd/Ctrl+Z for undo, Cmd/Ctrl+Y or Cmd/Ctrl+Shift+Z for redo
- Only tracks significant changes (layout changes, component additions/removals)

### Chart Selection Flow
1. User clicks "Add Chart" button
2. Modal opens showing all available charts with search
3. Charts are displayed with mini previews using existing MiniChart component
4. User selects a chart
5. Chart is added to the dashboard at default position
6. User can then drag to reposition

### Enum Usage Benefits
- Type safety in both Python and TypeScript
- Consistent values across backend and frontend
- Easy to extend with new types
- Follows existing codebase patterns

## Success Criteria

1. ✅ Users can create and edit native dashboards
2. ✅ Drag-and-drop grid layout works smoothly
3. ✅ Chart selection modal provides good UX
4. ✅ Undo/redo works with 20 operation limit
5. ✅ Filters apply to all chart components
6. ✅ Auto-save prevents data loss
7. ✅ Lock prevents concurrent editing
8. ✅ All tests pass
9. ✅ Performance meets requirements (<2s load time)

## References

### Existing Code Patterns
- Enum Pattern: `DDP_backend/ddpui/models/tasks.py:130` (TaskLockStatus)
- Chart Model: `DDP_backend/ddpui/models/visualization.py`
- TaskLock Pattern: `DDP_backend/ddpui/models/tasks.py:139`
- Superset Integration: `webapp_v2/components/dashboard/superset-embed.tsx`
- Chart Service: `DDP_backend/ddpui/core/charts/charts_service.py`

### External Documentation
- React Grid Layout: https://github.com/react-grid-layout/react-grid-layout
- React Grid Layout Examples: https://react-grid-layout.github.io/react-grid-layout/examples/0-showcase.html
- Auto-save Patterns: https://www.npmjs.com/package/react-autosave
- Dashboard Best Practices: https://www.ilert.com/blog/building-interactive-dashboards-why-react-grid-layout-was-our-best-choice

## Conclusion

This implementation plan provides a comprehensive approach to building native dashboards v2 for Dalgo. The plan leverages existing patterns in the codebase while introducing modern dashboard building capabilities. With the existing infrastructure and clear implementation path, this feature can be successfully implemented with high confidence.