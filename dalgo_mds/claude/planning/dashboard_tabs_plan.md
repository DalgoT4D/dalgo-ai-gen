# Dashboard Tabs Feature - Implementation Plan

## Feature Overview

**Goal:** Enable users to organize dashboard components into multiple tabs, with filters that work across all tabs (global filtering).

**Current State:** Dashboards currently display all components (charts, text, filters) in a single flat grid layout without any grouping mechanism.

**Target State:** Users can create, rename, reorder, and delete tabs within a dashboard. Components belong to specific tabs, and filters can apply globally across all tabs.

---

## High-Level Architecture

### Design Principles

1. **Backward Compatibility:** Existing dashboards without tabs continue to work seamlessly
2. **Global Filters:** Filters apply to all charts across all tabs (simplest UX for data filtering)
3. **Minimal Model Changes:** Use existing JSON fields (`layout_config`, `components`) to store tab metadata
4. **Consistent UX:** Follow existing patterns in the codebase (Radix UI Tabs component is already available)

### System Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    Dashboard View/Edit                           │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              FILTERS (Global - applies to all tabs)       │   │
│  │  [Filter 1] [Filter 2] [Filter 3]        [Apply] [Clear] │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  TABS BAR                                                 │   │
│  │  [Overview] [Sales Analysis] [Customer Metrics] [+ Add]  │   │
│  └──────────────────────────────────────────────────────────┘   │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │              TAB CONTENT (Grid Layout)                    │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │   │
│  │  │   Chart 1   │  │   Chart 2   │  │   Chart 3   │       │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │   │
│  │  ┌─────────────────────────────────────────────────┐     │   │
│  │  │              Chart 4 (wide)                      │     │   │
│  │  └─────────────────────────────────────────────────┘     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Data Model Design

### Option 1: Store Tabs in Existing JSON Fields (Recommended)

No new database model needed. Extend the existing `components` and `layout_config` JSON fields.

#### Updated `components` JSON Structure

```json
{
  "__tabs__": {
    "enabled": true,
    "tabs": [
      {
        "id": "tab-1",
        "label": "Overview",
        "order": 0
      },
      {
        "id": "tab-2",
        "label": "Sales Analysis",
        "order": 1
      },
      {
        "id": "tab-3",
        "label": "Customer Metrics",
        "order": 2
      }
    ],
    "defaultTab": "tab-1"
  },
  "chart-1": {
    "type": "chart",
    "config": { "chartId": 123 },
    "tabId": "tab-1"  // NEW: Tab association
  },
  "chart-2": {
    "type": "chart",
    "config": { "chartId": 456 },
    "tabId": "tab-2"
  },
  "text-1": {
    "type": "text",
    "config": { "content": "..." },
    "tabId": "tab-1"
  }
}
```

#### Updated `layout_config` Structure

```json
[
  {
    "i": "chart-1",
    "x": 0,
    "y": 0,
    "w": 6,
    "h": 4,
    "minW": 2,
    "minH": 2,
    "tabId": "tab-1"  // NEW: Tab association (redundant but useful for filtering)
  },
  {
    "i": "chart-2",
    "x": 0,
    "y": 0,
    "w": 12,
    "h": 6,
    "minW": 4,
    "minH": 3,
    "tabId": "tab-2"
  }
]
```

### Backward Compatibility

- Components without a `tabId` are shown in a default "Main" tab
- Dashboards without `__tabs__` configuration show all components in a single view (current behavior)
- Migration is automatic on first edit (if user adds tabs)

---

## API Design

### No New Endpoints Required

Tab management is handled through the existing dashboard update endpoint:

```
PUT /api/dashboards/{id}/
```

#### Request Body (with tabs)

```json
{
  "title": "My Dashboard",
  "layout_config": [...],  // Includes tabId per layout item
  "components": {
    "__tabs__": {...},     // Tab configuration
    "chart-1": {...},      // Component with tabId
    ...
  }
}
```

### Validation Rules (Backend)

1. Tab IDs must be unique within a dashboard
2. Tab labels must not be empty
3. At least one tab must exist if tabs are enabled
4. All components must reference valid tab IDs
5. Default tab must exist in the tabs array

---

## Frontend Implementation

### Files to Modify

#### 1. Type Definitions

**File:** `webapp_v2/types/dashboard-filters.ts` (or create new `webapp_v2/types/dashboard-tabs.ts`)

```typescript
// Dashboard Tab Types

export interface DashboardTab {
  id: string;
  label: string;
  order: number;
}

export interface DashboardTabsConfig {
  enabled: boolean;
  tabs: DashboardTab[];
  defaultTab: string;
}

// Extended component interface
export interface DashboardComponentWithTab {
  id: string;
  type: 'chart' | 'text' | 'heading' | 'filter';
  config: any;
  tabId?: string;  // Optional for backward compatibility
}

// Extended layout item
export interface DashboardLayoutItemWithTab {
  i: string;
  x: number;
  y: number;
  w: number;
  h: number;
  minW?: number;
  minH?: number;
  maxW?: number;
  maxH?: number;
  tabId?: string;  // Optional for backward compatibility
}
```

#### 2. Dashboard Builder V2 (Edit Mode)

**File:** `webapp_v2/components/dashboard/dashboard-builder-v2.tsx`

**Changes Required:**

1. Add state for active tab and tabs configuration
2. Add tab bar UI with Radix Tabs component
3. Filter layout items by active tab when rendering
4. Add tab management UI (add, rename, delete, reorder tabs)
5. Track which tab a component belongs to when adding/moving components

**Key Code Additions:**

```typescript
// New state
const [tabsConfig, setTabsConfig] = useState<DashboardTabsConfig>(() => {
  const existing = initialComponents['__tabs__'] as DashboardTabsConfig | undefined;
  return existing?.enabled ? existing : {
    enabled: false,
    tabs: [{ id: 'default', label: 'Main', order: 0 }],
    defaultTab: 'default'
  };
});
const [activeTab, setActiveTab] = useState<string>(tabsConfig.defaultTab);

// Filter layout for current tab
const tabLayout = useMemo(() => {
  if (!tabsConfig.enabled) return state.layout;
  return state.layout.filter(item => {
    const component = state.components[item.i];
    return (component?.tabId || 'default') === activeTab;
  });
}, [state.layout, state.components, activeTab, tabsConfig.enabled]);
```

#### 3. Dashboard Native View (View Mode)

**File:** `webapp_v2/components/dashboard/dashboard-native-view.tsx`

**Changes Required:**

1. Parse tabs configuration from dashboard data
2. Render tab bar using existing Radix Tabs component
3. Filter visible components based on active tab
4. Preserve filter state across tab switches (already works - filters are at dashboard level)

**Key Code Additions:**

```typescript
// Parse tabs config
const tabsConfig: DashboardTabsConfig | null = useMemo(() => {
  const tabs = dashboard?.components?.['__tabs__'];
  return tabs?.enabled ? tabs : null;
}, [dashboard?.components]);

// Active tab state
const [activeTab, setActiveTab] = useState<string>(() => {
  return tabsConfig?.defaultTab || 'default';
});

// Filter layout items for current tab
const visibleLayout = useMemo(() => {
  if (!tabsConfig) return dashboard?.layout_config || [];
  return (dashboard?.layout_config || []).filter(item => {
    const component = dashboard?.components?.[item.i];
    return (component?.tabId || 'default') === activeTab;
  });
}, [dashboard, activeTab, tabsConfig]);
```

#### 4. New Component: Tab Manager

**File:** `webapp_v2/components/dashboard/dashboard-tabs-manager.tsx` (NEW)

```typescript
'use client';

import { useState } from 'react';
import { DndContext, closestCenter, DragEndEvent } from '@dnd-kit/core';
import { SortableContext, horizontalListSortingStrategy, arrayMove } from '@dnd-kit/sortable';
import { Tabs, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Plus, X, GripHorizontal, Edit2, Check } from 'lucide-react';
import { cn } from '@/lib/utils';
import type { DashboardTab, DashboardTabsConfig } from '@/types/dashboard-tabs';

interface DashboardTabsManagerProps {
  tabsConfig: DashboardTabsConfig;
  activeTab: string;
  onActiveTabChange: (tabId: string) => void;
  onTabsConfigChange: (config: DashboardTabsConfig) => void;
  isEditMode: boolean;
}

export function DashboardTabsManager({
  tabsConfig,
  activeTab,
  onActiveTabChange,
  onTabsConfigChange,
  isEditMode,
}: DashboardTabsManagerProps) {
  const [editingTabId, setEditingTabId] = useState<string | null>(null);
  const [editingLabel, setEditingLabel] = useState('');

  const handleAddTab = () => {
    const newTab: DashboardTab = {
      id: `tab-${Date.now()}`,
      label: `Tab ${tabsConfig.tabs.length + 1}`,
      order: tabsConfig.tabs.length,
    };
    onTabsConfigChange({
      ...tabsConfig,
      enabled: true,
      tabs: [...tabsConfig.tabs, newTab],
    });
    onActiveTabChange(newTab.id);
  };

  const handleDeleteTab = (tabId: string) => {
    if (tabsConfig.tabs.length <= 1) return; // Keep at least one tab

    const newTabs = tabsConfig.tabs
      .filter(t => t.id !== tabId)
      .map((t, i) => ({ ...t, order: i }));

    const newDefaultTab = tabId === tabsConfig.defaultTab
      ? newTabs[0].id
      : tabsConfig.defaultTab;

    onTabsConfigChange({
      ...tabsConfig,
      tabs: newTabs,
      defaultTab: newDefaultTab,
    });

    if (activeTab === tabId) {
      onActiveTabChange(newTabs[0].id);
    }
  };

  const handleRenameTab = (tabId: string, newLabel: string) => {
    onTabsConfigChange({
      ...tabsConfig,
      tabs: tabsConfig.tabs.map(t =>
        t.id === tabId ? { ...t, label: newLabel } : t
      ),
    });
    setEditingTabId(null);
  };

  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event;
    if (over && active.id !== over.id) {
      const oldIndex = tabsConfig.tabs.findIndex(t => t.id === active.id);
      const newIndex = tabsConfig.tabs.findIndex(t => t.id === over.id);
      const reorderedTabs = arrayMove(tabsConfig.tabs, oldIndex, newIndex)
        .map((t, i) => ({ ...t, order: i }));
      onTabsConfigChange({ ...tabsConfig, tabs: reorderedTabs });
    }
  };

  // View mode - simple tabs
  if (!isEditMode) {
    return (
      <Tabs value={activeTab} onValueChange={onActiveTabChange}>
        <TabsList>
          {tabsConfig.tabs
            .sort((a, b) => a.order - b.order)
            .map(tab => (
              <TabsTrigger key={tab.id} value={tab.id}>
                {tab.label}
              </TabsTrigger>
            ))}
        </TabsList>
      </Tabs>
    );
  }

  // Edit mode - tabs with management controls
  return (
    <div className="flex items-center gap-2 border-b border-gray-200 pb-2 mb-4">
      <DndContext collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
        <SortableContext
          items={tabsConfig.tabs.map(t => t.id)}
          strategy={horizontalListSortingStrategy}
        >
          <Tabs value={activeTab} onValueChange={onActiveTabChange}>
            <TabsList>
              {tabsConfig.tabs
                .sort((a, b) => a.order - b.order)
                .map(tab => (
                  <SortableTab
                    key={tab.id}
                    tab={tab}
                    isActive={tab.id === activeTab}
                    isEditing={editingTabId === tab.id}
                    editingLabel={editingLabel}
                    canDelete={tabsConfig.tabs.length > 1}
                    onSelect={() => onActiveTabChange(tab.id)}
                    onStartEdit={() => {
                      setEditingTabId(tab.id);
                      setEditingLabel(tab.label);
                    }}
                    onEditChange={setEditingLabel}
                    onSaveEdit={() => handleRenameTab(tab.id, editingLabel)}
                    onCancelEdit={() => setEditingTabId(null)}
                    onDelete={() => handleDeleteTab(tab.id)}
                  />
                ))}
            </TabsList>
          </Tabs>
        </SortableContext>
      </DndContext>

      <Button
        variant="ghost"
        size="sm"
        onClick={handleAddTab}
        className="h-8 px-2"
      >
        <Plus className="w-4 h-4 mr-1" />
        Add Tab
      </Button>
    </div>
  );
}
```

#### 5. Component Assignment UI

When adding a component (chart, text) in edit mode, it should be automatically assigned to the currently active tab.

**Modification in `dashboard-builder-v2.tsx`:**

```typescript
const addChartToCanvas = useCallback((chartData: any) => {
  const componentId = `chart-${Date.now()}`;
  const newComponent = {
    type: DashboardComponentType.CHART,
    config: { chartId: chartData.id },
    tabId: activeTab,  // Assign to current tab
  };

  const newLayoutItem = {
    i: componentId,
    x: 0,
    y: findLowestY(tabLayout),  // Use filtered layout
    w: 6,
    h: 4,
    minW: 2,
    minH: 2,
    tabId: activeTab,  // Assign to current tab
  };

  // ... rest of logic
}, [activeTab, tabLayout]);
```

---

## Filter Behavior with Tabs

### Global Filters (Recommended Approach)

Filters remain at the dashboard level and apply to all charts across all tabs:

1. **Filter Position:** Filters are displayed in the sidebar/top bar outside the tab content area
2. **Filter State:** `AppliedFilters` state is maintained at the dashboard level (current behavior)
3. **Cross-Tab Application:** When user switches tabs, filters are already applied to charts in the new tab
4. **No Changes Needed:** Current filter implementation already works globally - charts receive `dashboardFilters` prop regardless of tab

### Implementation Notes

The current implementation already supports this:

```typescript
// In dashboard-native-view.tsx
<ChartElementView
  chartId={component.config?.chartId}
  dashboardFilters={selectedFilters}        // Same filters for all charts
  dashboardFilterConfigs={dashboardFilters} // Same filter configs
  // ...
/>
```

When rendering tab content, the same `selectedFilters` state is passed to all charts regardless of which tab they're on.

---

## Implementation Phases

### Phase 1: Backend Validation (Optional)

Add validation in `dashboard_native_api.py` for the new JSON structure:

```python
def validate_tabs_config(components: dict) -> bool:
    """Validate tabs configuration in components dict."""
    tabs_config = components.get('__tabs__')
    if not tabs_config or not tabs_config.get('enabled'):
        return True  # No tabs or tabs disabled - valid

    tabs = tabs_config.get('tabs', [])
    if not tabs:
        return False  # Tabs enabled but no tabs defined

    tab_ids = {t['id'] for t in tabs}
    default_tab = tabs_config.get('defaultTab')

    if default_tab and default_tab not in tab_ids:
        return False  # Default tab doesn't exist

    # Validate all components reference valid tabs
    for key, component in components.items():
        if key == '__tabs__':
            continue
        tab_id = component.get('tabId')
        if tab_id and tab_id not in tab_ids:
            return False  # Component references invalid tab

    return True
```

### Phase 2: Frontend - Type Definitions

1. Create `webapp_v2/types/dashboard-tabs.ts` with tab interfaces
2. Update existing dashboard types if needed

### Phase 3: Frontend - Tab Manager Component

1. Create `dashboard-tabs-manager.tsx` with view/edit modes
2. Implement add, rename, delete, reorder functionality
3. Use existing Radix UI Tabs and dnd-kit for drag-and-drop

### Phase 4: Frontend - Dashboard Builder Integration

1. Add tab state management to `dashboard-builder-v2.tsx`
2. Filter layout items by active tab
3. Auto-assign new components to active tab
4. Show "Enable Tabs" toggle in toolbar
5. Update save logic to include tabs configuration

### Phase 5: Frontend - Dashboard View Integration

1. Update `dashboard-native-view.tsx` to render tabs
2. Filter components by active tab
3. Persist active tab in URL or local state (optional)

### Phase 6: Testing

1. Unit tests for tab manager component
2. Integration tests for tab switching with filters
3. E2E tests for full tab workflow (create, rename, delete, switch)
4. Backward compatibility tests (dashboards without tabs)

---

## Testing Strategy

### Unit Tests

**File:** `webapp_v2/__tests__/components/dashboard/dashboard-tabs-manager.test.tsx`

```typescript
describe('DashboardTabsManager', () => {
  it('renders tabs in correct order', () => {});
  it('handles tab selection', () => {});
  it('adds new tab', () => {});
  it('renames tab', () => {});
  it('deletes tab and selects next', () => {});
  it('prevents deleting last tab', () => {});
  it('reorders tabs via drag and drop', () => {});
  it('shows edit controls only in edit mode', () => {});
});
```

### Integration Tests

**File:** `webapp_v2/__tests__/components/dashboard/dashboard-with-tabs.test.tsx`

```typescript
describe('Dashboard with Tabs', () => {
  it('filters layout items by active tab', () => {});
  it('applies filters to all tabs', () => {});
  it('assigns new component to active tab', () => {});
  it('saves tab configuration correctly', () => {});
  it('loads dashboard without tabs (backward compatibility)', () => {});
});
```

### E2E Tests

**File:** `webapp_v2/e2e/dashboards/tabs.spec.ts`

```typescript
import { test, expect } from '@playwright/test';

test.describe('Dashboard Tabs', () => {
  test('create dashboard with tabs', async ({ page }) => {
    // Navigate to create dashboard
    // Enable tabs
    // Add multiple tabs
    // Add charts to different tabs
    // Save and verify persistence
  });

  test('filter applies across tabs', async ({ page }) => {
    // Open dashboard with tabs
    // Apply filter on Tab 1
    // Switch to Tab 2
    // Verify filter is still applied to charts
  });

  test('edit existing dashboard tabs', async ({ page }) => {
    // Open existing dashboard for edit
    // Rename tab
    // Reorder tabs
    // Delete tab
    // Verify changes persist
  });
});
```

### Test Cases Checklist

- [ ] Create dashboard with no tabs (current behavior works)
- [ ] Enable tabs on new dashboard
- [ ] Add/rename/delete/reorder tabs
- [ ] Add components to specific tabs
- [ ] Move components between tabs (if supported)
- [ ] Switch tabs in view mode
- [ ] Apply filters and switch tabs (filters persist)
- [ ] Save dashboard with tabs
- [ ] Load dashboard with tabs
- [ ] Load dashboard without tabs (backward compatibility)
- [ ] Public dashboard with tabs
- [ ] Mobile/responsive tabs display

---

## Migration Strategy

### Automatic Migration (On Edit)

When a user edits an existing dashboard for the first time after this feature is deployed:

1. If dashboard has no `__tabs__` config, continue showing single view
2. If user clicks "Enable Tabs", create default tab and assign all existing components to it
3. Save new structure on next save

### No Data Migration Required

Existing dashboards work without modification because:
- Components without `tabId` show in default view
- Absence of `__tabs__` config means tabs are disabled
- All existing functionality preserved

---

## File Changes Summary

### Backend (DDP_backend)

| File | Change Type | Description |
|------|-------------|-------------|
| `ddpui/api/dashboard_native_api.py` | Modify | Add tabs config validation (optional) |
| `ddpui/schemas/dashboard_schema.py` | No Change | Existing schema accepts any JSON in components |

### Frontend (webapp_v2)

| File | Change Type | Description |
|------|-------------|-------------|
| `types/dashboard-tabs.ts` | New | Tab type definitions |
| `components/dashboard/dashboard-tabs-manager.tsx` | New | Tab bar component with edit controls |
| `components/dashboard/dashboard-builder-v2.tsx` | Modify | Add tab state, filter by tab, assign components to tabs |
| `components/dashboard/dashboard-native-view.tsx` | Modify | Render tabs, filter by active tab |
| `__tests__/components/dashboard/` | New | Unit/integration tests |
| `e2e/dashboards/tabs.spec.ts` | New | E2E tests |

---

## External Resources

### Documentation

- **Radix UI Tabs:** https://www.radix-ui.com/primitives/docs/components/tabs
- **dnd-kit Sortable:** https://docs.dndkit.com/presets/sortable
- **React Grid Layout:** https://github.com/react-grid-layout/react-grid-layout

### Similar Implementations

- **Apache Superset Dashboard Tabs:** Superset uses a similar approach where tabs contain multiple charts
- **Metabase Dashboard Tabs:** Recent feature addition with similar UX patterns

---

## Success Criteria

1. **Functional:**
   - [ ] Users can create dashboards with multiple tabs
   - [ ] Users can add/rename/delete/reorder tabs
   - [ ] Components can be added to specific tabs
   - [ ] Filters work across all tabs
   - [ ] Tab state persists on save

2. **Backward Compatible:**
   - [ ] Existing dashboards without tabs continue working
   - [ ] No migration script required
   - [ ] API responses remain compatible

3. **Performance:**
   - [ ] Tab switching is instant (no API calls)
   - [ ] Only active tab's charts render (optional optimization)

4. **Tested:**
   - [ ] Unit tests for tab manager
   - [ ] Integration tests for tab + filter interaction
   - [ ] E2E tests for complete workflow
   - [ ] Manual testing on mobile/tablet

---

## Open Questions

1. **Move components between tabs?** Should users be able to drag a component from one tab to another in edit mode?
   - Recommendation: Yes, implement via context menu "Move to Tab" option

2. **Tab-specific filters?** Should some filters only apply to specific tabs?
   - Recommendation: No, keep filters global for simplicity. Can be added later if needed.

3. **URL sync?** Should the active tab be reflected in the URL (e.g., `/dashboards/123?tab=sales`)?
   - Recommendation: Yes, for shareability. Use URL search params.

4. **Default tab setting?** Should users be able to set which tab opens by default?
   - Recommendation: Yes, store `defaultTab` in tabs config.

---

## Estimated Effort

| Phase | Description | Estimated Time |
|-------|-------------|----------------|
| Phase 1 | Backend validation | 2-3 hours |
| Phase 2 | Type definitions | 1 hour |
| Phase 3 | Tab manager component | 4-6 hours |
| Phase 4 | Dashboard builder integration | 6-8 hours |
| Phase 5 | Dashboard view integration | 4-6 hours |
| Phase 6 | Testing | 4-6 hours |
| **Total** | | **~24-32 hours** |

---

## Quality Checklist

- [x] All necessary context included
- [x] Validation gates are executable by AI
- [x] References existing patterns (Radix Tabs, dnd-kit)
- [x] Clear implementation path for all services
- [x] Error handling documented (validation)
- [x] Backward compatibility addressed
- [x] Testing strategy defined
