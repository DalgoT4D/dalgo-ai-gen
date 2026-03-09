# Drill-Down Table Enhancement Plan

**Date Created:** 2025-12-17
**Author:** Planning Document
**Status:** Draft for Review

---

## Executive Summary

This document outlines a comprehensive plan to enhance the table chart drill-down functionality in Dalgo. The enhancement builds upon recently completed multi-dimensional table support and introduces a more flexible, user-centric approach to hierarchical data navigation.

### Current vs. Proposed Architecture

| Aspect | Current Implementation | Proposed Enhancement |
|--------|----------------------|---------------------|
| **Dimensions** | Single `dimension_col` or limited dimensions array | Unlimited dimensions array with user-defined order |
| **Metrics** | Optional (recently implemented) | Optional (maintained) |
| **Hierarchy Definition** | Fixed `DrillDownConfig.hierarchy` array with predefined levels | Toggle-based: user enables drill-down on selected dimensions |
| **Hierarchy Order** | Explicitly configured in hierarchy array | Derived from dimension order (first enabled dimension → second → third...) |
| **Column Management** | Fixed display columns via `table_columns` | Drag-and-drop column rearrangement with display order control |
| **Configuration Complexity** | Separate dimension, drill-down, and aggregation configs | Unified configuration in dimension array with drill-down toggles |

### Key Benefits

1. **Simplified UX**: No separate drill-down hierarchy configuration needed
2. **Unlimited Flexibility**: Add as many dimensions as needed for analysis
3. **Intuitive Hierarchy**: Dimension order = drill-down order
4. **Better Column Control**: Rearrange any columns including dimensions and metrics
5. **Backward Compatible**: Existing charts continue working with migration path

---

## 1. Current Implementation Analysis

### 1.1 Frontend Architecture

#### SimpleDrillDownConfiguration.tsx
**Current Pattern:**
- Displays table columns selection with checkboxes
- Separate drill-down column dropdown (single column selection)
- `max_drill_levels` input (1-5 range)
- Aggregation columns selection for numeric columns

**Data Flow:**
```typescript
// Configuration stored in formData
{
  table_columns: string[],              // Display columns
  extra_config: {
    drill_down_column: string,          // Single column for drill-down
    max_drill_levels: number,           // 1-5
    aggregation_columns: string[]       // Numeric columns to sum
  }
}
```

**Limitations:**
- Only one column can be drilled at a time
- Hierarchy is implicit (same column at each level)
- Fixed 5-level maximum
- No clear relationship between dimensions and drill-down columns

#### TableChart.tsx
**Current Drill-Down Logic:**
```typescript
// Lines 94-102
const isDrillDownConfigured =
  drill_down_config?.enabled && drill_down_config?.hierarchy?.length > 0;

const currentDrillLevel = drillDownPath.length;
const drillDownColumn = drill_down_config?.hierarchy?.[currentDrillLevel]?.column;
```

**Breadcrumb Navigation:**
- Uses `DrillDownBreadcrumb` component with path and config
- Shows "Home" button and clickable path steps
- Displays current level indicator

**Clickable Cell Logic (Lines 348-359):**
```typescript
const isDrillDownCell = Boolean(
  isDrillDownConfigured === true &&
  columnMatches === true &&
  hasNextLevel === true &&
  onDrillDown !== undefined
);
```

### 1.2 Backend Architecture

#### Chart Schema (chart_schema.py)
**Current Payload Fields:**
```python
class ChartDataPayload(Schema):
    dimensions: Optional[List[str]] = None  # Multiple dimensions (new)
    dimension_col: Optional[str] = None     # Legacy single dimension

    # Drill-down fields
    drill_down_level: Optional[int] = 0
    drill_down_path: Optional[List[dict]] = None
    # [{"level": 0, "column": "state", "value": "Maharashtra"}]
```

#### Charts API (charts_api.py)
**Drill-Down Processing (Lines 1147-1186):**
```python
drill_down_config = extra_config.get("drill_down_config", {})
drill_down_enabled = drill_down_config.get("enabled", False)

if drill_down_enabled and (payload.drill_down_level > 0 or payload.drill_down_path):
    # Convert drill-down path to filters
    drill_down_filters = []
    if payload.drill_down_path:
        for step in payload.drill_down_path:
            drill_down_filters.append({
                "column": step["column"],
                "operator": "equals",
                "value": step["value"]
            })

    # Determine current level configuration
    hierarchy = drill_down_config.get("hierarchy", [])
    if payload.drill_down_level < len(hierarchy):
        level_config = hierarchy[payload.drill_down_level]
```

**Key Insight:** Backend already supports multiple dimensions (`dimensions: List[str]`), and drill-down is filter-based. The enhancement is primarily frontend-driven with minor backend schema updates.

### 1.3 Type Definitions

#### charts.ts
```typescript
export interface DrillDownLevel {
  level: number;
  column: string;
  display_name: string;
  table_name?: string;
  schema_name?: string;
  parent_column?: string;
  aggregation_columns?: string[];
}

export interface DrillDownConfig {
  enabled: boolean;
  hierarchy: DrillDownLevel[];
  clickable_columns?: ClickableColumn[];
}

export interface DrillDownPathStep {
  level: number;
  column: string;
  value: any;
  display_name: string;
}
```

---

## 2. Requirements Breakdown

### Requirement 1: Review Existing Drill-Down Implementation ✅
**Status:** Completed in Section 1
**Findings:**
- Current implementation uses fixed hierarchy with single drill-down column
- Backend supports filters and path tracking
- Frontend has breadcrumb navigation and clickable cells
- Multi-dimensional support exists but not integrated with drill-down

---

### Requirement 2: Unlimited Dimensions + Rename "Group By" to "Dimension"

#### 2.1 Technical Specifications

**Frontend Changes:**
1. **DimensionsSelector Component** (already exists)
   - Currently used for dimensions in table charts
   - Needs enhancement: Add drill-down toggle per dimension
   - Interface update:
   ```typescript
   export interface ChartDimension {
     column: string;
     alias?: string;
     enable_drill_down?: boolean;  // NEW: Toggle for this dimension
     display_order?: number;        // NEW: For column rearrangement
   }
   ```

2. **Remove "Group By" Terminology**
   - Find: `dimension_col`, `dimension column`, `Group by`
   - Replace with: `dimensions`, `Dimensions`, `Dimension`
   - Files affected:
     - `ChartDataConfigurationV3.tsx`
     - `SimpleDrillDownConfiguration.tsx` (to be replaced)
     - `chartAutoPrefill.ts`

**Backend Changes:**
1. **Schema Update** (chart_schema.py)
   ```python
   # Deprecate dimension_col in favor of dimensions
   # Add comments encouraging dimensions array usage
   dimension_col: Optional[str] = None  # DEPRECATED: Use dimensions instead
   dimensions: Optional[List[str]] = None  # Preferred for all charts
   ```

2. **Service Layer** (charts_service.py)
   - Already implemented: `build_table_query` uses dimensions array
   - Add logging for dimension_col deprecation warnings

#### 2.2 Implementation Steps

1. **Phase 2.1: Update DimensionsSelector**
   - Add drill-down toggle checkbox next to each dimension
   - Store `enable_drill_down` flag in ChartDimension objects
   - Visual indicator (icon) for drill-enabled dimensions

2. **Phase 2.2: Update Type Definitions**
   - Extend `ChartDimension` interface in `types/charts.ts`
   - Update `ChartBuilderFormData` and `ChartCreate` types

3. **Phase 2.3: Terminology Migration**
   - Global find/replace for UI text
   - Update component prop names (backward compatible)
   - Add migration notes in code comments

#### 2.3 Testing Strategy

**Unit Tests:**
```typescript
// DimensionsSelector.test.tsx
describe('DimensionsSelector - Unlimited Dimensions', () => {
  it('should allow adding more than 5 dimensions', () => {
    // Add 10 dimensions and verify all are stored
  });

  it('should preserve dimension order when adding/removing', () => {
    // Add dims A, B, C → Remove B → Add D → Verify order: A, C, D
  });

  it('should not auto-limit dimension count', () => {
    // Ensure no MAX_DIMENSIONS constraint exists
  });
});
```

**Backend Tests:**
```python
# test_charts_service_table.py
def test_table_with_10_dimensions():
    """Test table query building with 10+ dimensions"""
    payload = ChartDataPayload(
        chart_type="table",
        dimensions=["dim1", "dim2", ..., "dim10"],
        metrics=None,
    )
    result = build_table_query(payload, query_builder)
    assert len(result.select_cols) == 10
    assert len(result.group_cols) == 10
```

---

### Requirement 3: Metrics Optional for Table Creation ✅

**Status:** Already Implemented
**Details:**
- Backend validation updated (line 318-319 in charts_service.py)
- Frontend DimensionsSelector supports dimension-only tables
- Tests exist in `test_charts_service_table.py`

**Action Items:**
- ✅ Verify existing implementation
- ✅ Add integration tests for dimension-only drill-down
- Document this capability in user docs

---

### Requirement 4: Toggle-Based Hierarchy from Dimension Order

#### 4.1 Conceptual Model

**Current Model:**
```
User selects drill-down column → Separate hierarchy config → Fixed drill path
```

**New Model:**
```
User adds dimensions → Toggles drill-down on specific dimensions → Order defines hierarchy
```

**Example:**
```
Dimensions (in order):
1. Country       [✓ Enable Drill-Down]
2. State         [✓ Enable Drill-Down]
3. City          [✓ Enable Drill-Down]
4. Product       [ ] (no drill-down)
5. Category      [ ] (no drill-down)

Metrics:
1. Total Sales (sum of amount)
2. Avg Quantity (avg of quantity)

Result: Hierarchy is Country → State → City (Product and Category are normal columns)
```

#### 4.2 Technical Architecture

**Frontend State Management:**
```typescript
// ChartBuilder state
interface ChartBuilderFormData {
  dimensions: ChartDimension[];  // Ordered array
  metrics?: ChartMetric[];

  // Computed from dimensions array
  // drill_down_config?: DrillDownConfig;  // DEPRECATED
}

// Derived drill-down config (computed property)
function getDrillDownConfig(dimensions: ChartDimension[]): DrillDownConfig {
  const drillDimensions = dimensions.filter(d => d.enable_drill_down);

  if (drillDimensions.length === 0) {
    return { enabled: false, hierarchy: [] };
  }

  return {
    enabled: true,
    hierarchy: drillDimensions.map((dim, index) => ({
      level: index,
      column: dim.column,
      display_name: dim.alias || dim.column,
    })),
  };
}
```

**Backend Payload:**
```python
# extra_config now contains drill-down hierarchy derived from frontend
{
  "dimensions": ["country", "state", "city", "product"],
  "metrics": [...],
  "extra_config": {
    "drill_down_config": {
      "enabled": true,
      "hierarchy": [
        {"level": 0, "column": "country", "display_name": "Country"},
        {"level": 1, "column": "state", "display_name": "State"},
        {"level": 2, "column": "city", "display_name": "City"}
      ]
    },
    "dimension_drill_down_flags": {  # NEW: Store toggle states
      "country": true,
      "state": true,
      "city": true,
      "product": false
    }
  }
}
```

#### 4.3 UI/UX Design

**DimensionsSelector Enhanced:**
```
┌─────────────────────────────────────────────────────────┐
│ Dimensions                                              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. [⋮⋮] Country          [✓] Drill-Down  [🗑️]        │
│         Alias: [Country        ]                       │
│                                                         │
│  2. [⋮⋮] State            [✓] Drill-Down  [🗑️]        │
│         Alias: [State/Province ]                       │
│                                                         │
│  3. [⋮⋮] City             [✓] Drill-Down  [🗑️]        │
│         Alias: [City           ]                       │
│                                                         │
│  4. [⋮⋮] Product          [ ] Drill-Down  [🗑️]        │
│         Alias: [Product Name   ]                       │
│                                                         │
│  [+ Add Dimension]                                      │
│                                                         │
│  ℹ️  Drill-down hierarchy: Country → State → City      │
│     (Drag to reorder dimensions)                       │
└─────────────────────────────────────────────────────────┘
```

**Key UI Elements:**
1. **Drag Handle ([⋮⋮])**: Reorder dimensions via drag-and-drop
2. **Drill-Down Toggle**: Enable/disable drill-down for each dimension
3. **Hierarchy Preview**: Show computed drill path below dimension list
4. **Visual Indicator**: Highlight drill-enabled dimensions (e.g., blue border)

#### 4.4 Implementation Steps

**Phase 4.1: Update DimensionsSelector Component**
1. Add `enable_drill_down` toggle checkbox
2. Add drag-and-drop reordering (using `react-beautiful-dnd` or `dnd-kit`)
3. Compute and display hierarchy preview
4. Handle toggle state changes

**Phase 4.2: Update ChartBuilder Logic**
1. Compute `drill_down_config` from dimensions array before saving
2. Store `dimension_drill_down_flags` in `extra_config`
3. Remove old `SimpleDrillDownConfiguration` component
4. Update payload construction in ChartBuilder.tsx

**Phase 4.3: Update TableChart Rendering**
1. Use computed hierarchy from `drill_down_config`
2. Determine clickable column based on current drill level
3. Maintain backward compatibility with old drill_down_config format

**Phase 4.4: Backend Compatibility**
1. Accept both old and new formats in API
2. Migration function to convert old configs to new format
3. Return computed hierarchy in response

#### 4.5 Testing Strategy

**Unit Tests:**
```typescript
// DimensionsSelector.test.tsx
describe('Toggle-Based Hierarchy', () => {
  it('should compute hierarchy from enabled dimensions', () => {
    const dimensions = [
      { column: 'country', enable_drill_down: true },
      { column: 'state', enable_drill_down: true },
      { column: 'city', enable_drill_down: false },
      { column: 'product', enable_drill_down: true },
    ];
    const hierarchy = getDrillDownConfig(dimensions).hierarchy;
    expect(hierarchy).toHaveLength(3);
    expect(hierarchy[0].column).toBe('country');
    expect(hierarchy[1].column).toBe('state');
    expect(hierarchy[2].column).toBe('product'); // city skipped
  });

  it('should update hierarchy when toggle changes', () => {
    // Toggle drill-down on → verify hierarchy includes dimension
    // Toggle drill-down off → verify hierarchy excludes dimension
  });

  it('should preserve hierarchy order when dimensions reordered', () => {
    // Drag dimension[2] to position 0 → verify hierarchy updates
  });
});
```

**Integration Tests:**
```typescript
// ChartBuilder.integration.test.tsx
describe('Drill-Down Table Creation', () => {
  it('should create table with toggle-based hierarchy', async () => {
    // 1. Select table chart type
    // 2. Add 3 dimensions
    // 3. Enable drill-down on first 2
    // 4. Save chart
    // 5. Verify saved config has correct hierarchy
  });

  it('should drill down correctly in TableChart', async () => {
    // 1. Render table with hierarchy
    // 2. Click first dimension value
    // 3. Verify drill-down to level 1
    // 4. Verify only second dimension is clickable
    // 5. Click breadcrumb to navigate back
  });
});
```

**Backend Tests:**
```python
# test_charts_api.py
def test_toggle_based_drill_down(client):
    """Test API accepts toggle-based drill-down config"""
    payload = {
        "chart_type": "table",
        "dimensions": ["country", "state", "city"],
        "extra_config": {
            "drill_down_config": {
                "enabled": True,
                "hierarchy": [
                    {"level": 0, "column": "country", "display_name": "Country"},
                    {"level": 1, "column": "state", "display_name": "State"},
                ]
            },
            "dimension_drill_down_flags": {
                "country": True,
                "state": True,
                "city": False
            }
        }
    }
    response = client.post("/api/charts/chart-data-preview/", json=payload)
    assert response.status_code == 200
```

---

### Requirement 5: Column Rearrangement

#### 5.1 Technical Specifications

**Goals:**
1. Allow users to reorder any displayed column (dimensions + metrics)
2. Persist column order in chart configuration
3. Render table columns in user-defined order
4. Maintain drill-down functionality regardless of display order

**Key Insight:** Display order ≠ Hierarchy order
- **Hierarchy Order:** Determined by dimension order with drill-down enabled
- **Display Order:** Visual arrangement of columns in the table

**Example:**
```
Dimensions (dimension order):
1. Country [✓ Drill-Down]
2. State [✓ Drill-Down]
3. City [✓ Drill-Down]
4. Product [ ]

Metrics:
1. Total Sales
2. Avg Quantity

Display Order (column_display_order):
1. Product
2. Total Sales
3. Country
4. State
5. City
6. Avg Quantity

Result:
- Table displays columns: Product | Total Sales | Country | State | City | Avg Quantity
- Drill-down hierarchy: Country → State → City (unchanged by display order)
- Country column is clickable first, then State, then City
```

#### 5.2 Data Model

**Extended ChartCreate/ChartUpdate Schema:**
```typescript
export interface ChartCreate {
  extra_config: {
    // Existing fields...
    dimensions?: ChartDimension[];  // With enable_drill_down flags
    metrics?: ChartMetric[];

    // NEW: Display order configuration
    column_display_order?: string[];  // Array of column names in display order
    // Example: ["product", "total_sales", "country", "state", "city", "avg_quantity"]
  };
}
```

**Backend Schema Update:**
```python
# chart_schema.py
class ChartDataPayload(Schema):
    # Existing fields...

    # New field for column display order
    extra_config: Optional[dict] = None
    # extra_config["column_display_order"]: Optional[List[str]]
```

#### 5.3 UI Components

**New Component: ColumnOrderManager**
```typescript
interface ColumnOrderManagerProps {
  dimensions: ChartDimension[];
  metrics: ChartMetric[];
  columnDisplayOrder: string[];
  onChange: (newOrder: string[]) => void;
}

export function ColumnOrderManager({
  dimensions,
  metrics,
  columnDisplayOrder,
  onChange,
}: ColumnOrderManagerProps) {
  // Compute all available columns
  const allColumns = [
    ...dimensions.map(d => ({ id: d.column, label: d.alias || d.column, type: 'dimension' })),
    ...metrics.map(m => ({ id: m.alias || m.aggregation, label: m.alias, type: 'metric' })),
  ];

  // Render drag-and-drop list
  return (
    <DndContext onDragEnd={handleDragEnd}>
      <SortableContext items={columnDisplayOrder}>
        {columnDisplayOrder.map((columnId, index) => {
          const column = allColumns.find(c => c.id === columnId);
          return (
            <SortableColumnItem
              key={columnId}
              id={columnId}
              label={column?.label}
              type={column?.type}
              index={index}
            />
          );
        })}
      </SortableContext>
    </DndContext>
  );
}
```

**Integration into SimpleDrillDownConfiguration:**
```typescript
export function SimpleDrillDownConfiguration({
  formData,
  columns,
  onChange,
}: SimpleDrillDownConfigurationProps) {
  // ... existing code ...

  return (
    <div className="space-y-4">
      {/* Existing Display Columns Selection */}
      <Card>
        <CardHeader>
          <CardTitle>Display Columns</CardTitle>
        </CardHeader>
        <CardContent>
          {/* Keep checkbox selection for enabling/disabling columns */}
        </CardContent>
      </Card>

      {/* NEW: Column Order Manager */}
      <Card>
        <CardHeader>
          <CardTitle>Column Display Order</CardTitle>
        </CardHeader>
        <CardContent>
          <ColumnOrderManager
            dimensions={formData.dimensions || []}
            metrics={formData.metrics || []}
            columnDisplayOrder={formData.extra_config?.column_display_order || []}
            onChange={(newOrder) => onChange({
              extra_config: {
                ...formData.extra_config,
                column_display_order: newOrder,
              }
            })}
          />
        </CardContent>
      </Card>
    </div>
  );
}
```

#### 5.4 TableChart Rendering Logic

**Update TableChart.tsx:**
```typescript
// Lines 111-120 (existing column computation)
const columns = useMemo(() => {
  // NEW: Use column_display_order if available
  const displayOrder = config.column_display_order;

  if (displayOrder && displayOrder.length > 0) {
    return displayOrder;
  }

  // Fallback to existing logic
  if (table_columns && table_columns.length > 0) {
    return table_columns;
  }

  if (data.length > 0) {
    return Object.keys(data[0]);
  }

  return [];
}, [data, config.column_display_order, table_columns]);
```

**Drill-Down Column Detection (No Change Needed):**
```typescript
// Lines 101-102 (existing)
const currentDrillLevel = drillDownPath.length;
const drillDownColumn = drill_down_config?.hierarchy?.[currentDrillLevel]?.column;
```
- This remains unchanged because hierarchy is independent of display order
- A column can be clickable even if it's displayed in the middle of the table

#### 5.5 Implementation Steps

**Phase 5.1: Create ColumnOrderManager Component**
1. Install `@dnd-kit/core` and `@dnd-kit/sortable` (if not already installed)
2. Build drag-and-drop column list component
3. Add visual indicators for column type (dimension vs metric)
4. Handle reordering logic

**Phase 5.2: Integrate into ChartBuilder Flow**
1. Add `column_display_order` to ChartBuilderFormData type
2. Update SimpleDrillDownConfiguration to include ColumnOrderManager
3. Compute default column order when user first selects dimensions/metrics
4. Save column order in extra_config

**Phase 5.3: Update TableChart Rendering**
1. Modify column computation logic to use `column_display_order`
2. Ensure drill-down detection still works with reordered columns
3. Test with various column arrangements

**Phase 5.4: Backend Validation**
1. Add validation to ensure `column_display_order` contains valid column names
2. Return column order in API responses
3. Handle missing/invalid column names gracefully

#### 5.6 Testing Strategy

**Unit Tests:**
```typescript
// ColumnOrderManager.test.tsx
describe('ColumnOrderManager', () => {
  it('should display all columns in current order', () => {
    const order = ['product', 'total_sales', 'country'];
    render(<ColumnOrderManager columnDisplayOrder={order} {...props} />);
    // Verify visual order matches array order
  });

  it('should reorder columns on drag-and-drop', () => {
    const onChange = jest.fn();
    const { reorderColumn } = renderColumnOrderManager({ onChange });

    reorderColumn('country', 0); // Move country to first position

    expect(onChange).toHaveBeenCalledWith(['country', 'product', 'total_sales']);
  });

  it('should distinguish dimension and metric columns visually', () => {
    // Verify different styling/icons for dimensions vs metrics
  });
});
```

**Integration Tests:**
```typescript
// TableChart.columnOrder.test.tsx
describe('TableChart Column Ordering', () => {
  it('should render columns in custom display order', () => {
    const config = {
      column_display_order: ['metric1', 'dim1', 'metric2', 'dim2'],
      dimensions: ['dim1', 'dim2'],
      metrics: [{ alias: 'metric1' }, { alias: 'metric2' }],
    };

    render(<TableChart data={data} config={config} />);

    const headers = screen.getAllByRole('columnheader');
    expect(headers[0]).toHaveTextContent('metric1');
    expect(headers[1]).toHaveTextContent('dim1');
    expect(headers[2]).toHaveTextContent('metric2');
    expect(headers[3]).toHaveTextContent('dim2');
  });

  it('should maintain drill-down functionality with reordered columns', () => {
    // Arrange: dim1 is first in hierarchy but last in display order
    // Act: Click dim1 cell
    // Assert: Drill-down occurs correctly
  });
});
```

**E2E Tests:**
```typescript
// e2e/table-column-reorder.spec.ts
test('user can reorder table columns via drag-and-drop', async ({ page }) => {
  // 1. Navigate to chart builder
  // 2. Create table with 3 dimensions and 2 metrics
  // 3. Open column order manager
  // 4. Drag metric to first position
  // 5. Save chart
  // 6. View chart
  // 7. Verify table displays metric in first column
});
```

---

### Requirement 6: End-to-End Implementation with Unit Tests

#### 6.1 Implementation Phases

**Phase 1: Foundation (Week 1)**
- [ ] Update ChartDimension interface with drill-down flags and display order
- [ ] Enhance DimensionsSelector with unlimited dimensions support
- [ ] Update backend schema to support new fields
- [ ] Create migration function for old drill-down configs

**Phase 2: Toggle-Based Hierarchy (Week 2)**
- [ ] Add drill-down toggle to DimensionsSelector
- [ ] Implement hierarchy computation logic
- [ ] Update ChartBuilder payload construction
- [ ] Add hierarchy preview in UI
- [ ] Backend: Accept new drill-down config format

**Phase 3: Column Rearrangement (Week 3)**
- [ ] Create ColumnOrderManager component
- [ ] Integrate with SimpleDrillDownConfiguration
- [ ] Update TableChart to use display order
- [ ] Add visual indicators for column types

**Phase 4: Testing & Refinement (Week 4)**
- [ ] Write comprehensive unit tests (all components)
- [ ] Write integration tests (ChartBuilder → TableChart flow)
- [ ] Write E2E tests (full user journey)
- [ ] Performance testing with large dimension counts
- [ ] Accessibility testing (keyboard navigation, screen readers)
- [ ] Cross-browser testing

**Phase 5: Documentation & Migration (Week 5)**
- [ ] Update user documentation
- [ ] Create migration guide for existing charts
- [ ] Add tooltips and help text in UI
- [ ] Backend: Log deprecation warnings for old configs
- [ ] Create video tutorial for new features

#### 6.2 Testing Coverage Matrix

| Component | Unit Tests | Integration Tests | E2E Tests |
|-----------|-----------|-------------------|-----------|
| **DimensionsSelector** | ✓ Add/remove dimensions<br>✓ Unlimited count<br>✓ Toggle drill-down<br>✓ Reorder dimensions | ✓ Updates ChartBuilder state<br>✓ Computes hierarchy correctly | ✓ User adds 10 dimensions<br>✓ User toggles drill-down |
| **ColumnOrderManager** | ✓ Drag-and-drop reorder<br>✓ Column type indicators<br>✓ Validate column list | ✓ Updates extra_config<br>✓ Syncs with dimensions/metrics | ✓ User reorders columns<br>✓ Order persists after save |
| **ChartBuilder** | ✓ Payload construction<br>✓ Hierarchy computation<br>✓ Config validation | ✓ Save chart with new config<br>✓ Load existing chart | ✓ Full chart creation flow<br>✓ Edit existing chart |
| **TableChart** | ✓ Column rendering order<br>✓ Clickable cell detection<br>✓ Breadcrumb navigation | ✓ Drill-down state changes<br>✓ Filter application | ✓ User drills down 3 levels<br>✓ Navigate via breadcrumb |
| **Backend API** | ✓ Schema validation<br>✓ Query building with dimensions<br>✓ Drill-down filter logic | ✓ Full request/response cycle<br>✓ Pagination with drill-down | ✓ API handles complex config<br>✓ Performance with 10+ dims |

#### 6.3 Test Scenarios

**Scenario 1: Create Table with Unlimited Dimensions**
```
User Actions:
1. Click "Create Chart" → Select "Table"
2. Select schema and table
3. Add 8 dimensions: country, state, city, district, product, category, brand, sku
4. Enable drill-down on: country, state, city, district
5. Add 3 metrics: Total Sales (sum), Avg Price (avg), Count (count)
6. Rearrange columns: Product, Brand, Total Sales, Country, State, City, District, Avg Price, Count
7. Save chart

Expected Results:
- Chart saves successfully
- extra_config contains:
  - dimensions: [country, state, city, district, product, category, brand, sku]
  - drill_down_config.hierarchy: [country, state, city, district] (4 levels)
  - column_display_order: [product, brand, total_sales, country, state, city, district, avg_price, count]
- Table displays columns in specified order
- Country column is clickable (first drill-down level)
```

**Scenario 2: Drill-Down Navigation with Reordered Columns**
```
Given: Table from Scenario 1

User Actions:
1. Click "USA" in Country column (column is displayed in 4th position)
2. Verify: State column becomes clickable, breadcrumb shows "Root / USA"
3. Click "California" in State column
4. Verify: City column becomes clickable, breadcrumb shows "Root / USA / California"
5. Click "Los Angeles" in City column
6. Verify: District column becomes clickable, breadcrumb shows "Root / USA / California / Los Angeles"
7. Click breadcrumb "California" to go back to level 2
8. Verify: Back at California level, City column is clickable

Expected Results:
- Drill-down works correctly despite column reordering
- Breadcrumb navigation is accurate
- Data filters correctly at each level
- No console errors
```

**Scenario 3: Toggle Drill-Down On/Off**
```
Given: Table with 4 dimensions, all drill-down enabled

User Actions:
1. Edit chart
2. Disable drill-down on "City" dimension
3. Save chart
4. View chart and drill down: Country → State
5. Verify: District column is now clickable (City skipped in hierarchy)

Expected Results:
- Hierarchy recomputed: [country, state, district]
- City column is not clickable at any level
- Drill-down skips City and goes to District after State
```

**Scenario 4: Dimension-Only Table (No Metrics)**
```
User Actions:
1. Create table with 5 dimensions
2. Enable drill-down on 3 dimensions
3. Do NOT add any metrics
4. Save and view chart

Expected Results:
- Chart saves successfully (metrics optional)
- Table displays all 5 dimensions
- Drill-down works on enabled dimensions
- No errors about missing metrics
```

**Scenario 5: Backward Compatibility**
```
Given: Existing chart with old drill-down config format
{
  "extra_config": {
    "drill_down_column": "state",
    "max_drill_levels": 3,
    "aggregation_columns": ["amount"]
  }
}

User Actions:
1. Open chart in edit mode
2. View shows migration notice: "This chart uses legacy drill-down. Click to upgrade."
3. Click "Upgrade"
4. System converts to new format:
   - Adds "state" to dimensions if not present
   - Sets enable_drill_down: true on "state"
   - Creates drill_down_config with hierarchy
5. Save chart

Expected Results:
- Old config is migrated seamlessly
- Chart continues to work as before
- New features (unlimited dimensions, column reorder) now available
```

#### 6.4 Test Implementation Files

**Frontend Unit Tests:**
```
webapp_v2/components/charts/__tests__/
  ├── DimensionsSelector.unlimited.test.tsx
  ├── DimensionsSelector.drillDownToggle.test.tsx
  ├── ColumnOrderManager.test.tsx
  ├── ChartBuilder.hierarchy.test.tsx
  └── TableChart.columnOrder.test.tsx
```

**Frontend Integration Tests:**
```
webapp_v2/__tests__/integration/
  ├── table-drill-down-flow.test.tsx
  ├── dimension-metric-management.test.tsx
  └── column-reordering.test.tsx
```

**E2E Tests:**
```
webapp_v2/e2e/
  ├── table-unlimited-dimensions.spec.ts
  ├── table-drill-down-navigation.spec.ts
  ├── table-column-rearrangement.spec.ts
  └── table-backward-compatibility.spec.ts
```

**Backend Tests:**
```
DDP_backend/ddpui/tests/
  ├── core/test_charts_service_unlimited_dimensions.py
  ├── core/test_charts_service_drill_down.py
  ├── api/test_charts_api_drill_down.py
  └── api/test_charts_api_column_order.py
```

---

## 3. Technical Architecture Details

### 3.1 Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                          User Interface                             │
│                                                                     │
│  ┌──────────────────┐        ┌──────────────────────────────┐     │
│  │ DimensionsSelector│───────▶│ ColumnOrderManager           │     │
│  │ - Add dimensions │        │ - Reorder columns            │     │
│  │ - Toggle drill   │        │ - Visual order preview       │     │
│  │ - Drag to reorder│        └──────────────┬───────────────┘     │
│  └────────┬─────────┘                       │                     │
│           │                                 │                     │
│           ▼                                 ▼                     │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │           ChartBuilder State (React Hook Form)             │  │
│  │  - dimensions: ChartDimension[]                            │  │
│  │  - metrics: ChartMetric[]                                  │  │
│  │  - extra_config.column_display_order: string[]            │  │
│  └────────────────────────┬───────────────────────────────────┘  │
│                           │                                       │
└───────────────────────────┼───────────────────────────────────────┘
                            │
                            │ Compute drill_down_config
                            │ from dimensions array
                            ▼
                   ┌─────────────────────┐
                   │ ChartDataPayload    │
                   │ - dimensions: []    │
                   │ - metrics: []       │
                   │ - extra_config:     │
                   │   - drill_down...   │
                   │   - column_display..│
                   └──────────┬──────────┘
                              │
                              │ POST /api/charts/
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Backend API Layer                             │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  charts_api.py                                               │  │
│  │  - Validate payload                                          │  │
│  │  - Extract drill_down_config                                 │  │
│  │  - Convert drill_down_path to filters                        │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
│                           │                                         │
│                           ▼                                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  charts_service.py                                           │  │
│  │  - build_table_query(payload, query_builder)                │  │
│  │  - Add dimension columns to SELECT                          │  │
│  │  - Add metric aggregations                                  │  │
│  │  - Add GROUP BY for all dimensions                          │  │
│  │  - Apply drill-down filters (WHERE clauses)                 │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
│                           │                                         │
│                           ▼                                         │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  query_builder.py (AggQueryBuilder)                          │  │
│  │  - Build SQL query                                           │  │
│  │  - Execute against warehouse                                │  │
│  │  - Return results                                            │  │
│  └────────────────────────┬─────────────────────────────────────┘  │
│                           │                                         │
└───────────────────────────┼─────────────────────────────────────────┘
                            │
                            │ JSON response
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      Frontend Rendering                             │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  TableChart Component                                        │  │
│  │  1. Read column_display_order from config                   │  │
│  │  2. Render columns in specified order                       │  │
│  │  3. Determine clickable column from hierarchy[currentLevel] │  │
│  │  4. Apply drill-down handlers to clickable cells            │  │
│  │  5. Render breadcrumb navigation                            │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 State Management Architecture

**ChartBuilder Form State:**
```typescript
interface ChartBuilderFormData {
  // Chart metadata
  title: string;
  chart_type: 'table';
  schema_name: string;
  table_name: string;

  // Data configuration
  dimensions: ChartDimension[];  // Ordered array with drill-down flags
  metrics?: ChartMetric[];       // Optional for tables

  // Display and interaction config
  extra_config: {
    // Column display order (independent of dimension order)
    column_display_order?: string[];

    // Computed drill-down config (derived from dimensions)
    drill_down_config?: DrillDownConfig;

    // Store dimension drill-down flags for editing
    dimension_drill_down_flags?: Record<string, boolean>;

    // Existing fields
    filters?: ChartFilter[];
    sort?: ChartSort[];
    pagination?: ChartPagination;
  };
}

// Computed property (not stored)
function computeDrillDownConfig(dimensions: ChartDimension[]): DrillDownConfig {
  const drillableDims = dimensions.filter(d => d.enable_drill_down);

  if (drillableDims.length === 0) {
    return { enabled: false, hierarchy: [] };
  }

  return {
    enabled: true,
    hierarchy: drillableDims.map((dim, idx) => ({
      level: idx,
      column: dim.column,
      display_name: dim.alias || dim.column,
    })),
  };
}
```

**TableChart Runtime State:**
```typescript
interface TableChartState {
  // Drill-down navigation state
  drillDownPath: DrillDownPathStep[];  // Breadcrumb trail
  currentLevel: number;                // Current depth in hierarchy

  // Pagination state
  currentPage: number;
  pageSize: number;

  // Data state (managed by SWR)
  data: any[];
  isLoading: boolean;
  error: any;
}

// Drill-down state update on cell click
function handleDrillDownClick(column: string, value: any) {
  const currentLevelConfig = drill_down_config.hierarchy[currentLevel];

  // Add new step to path
  const newPath = [
    ...drillDownPath,
    {
      level: currentLevel,
      column: currentLevelConfig.column,
      value: value,
      display_name: currentLevelConfig.display_name,
    }
  ];

  // Update state
  setDrillDownPath(newPath);

  // Refetch data with new filters
  mutate();
}
```

### 3.3 API Contract

**POST /api/charts/chart-data-preview/**

**Request:**
```json
{
  "chart_type": "table",
  "schema_name": "public",
  "table_name": "sales",
  "dimensions": ["country", "state", "city", "product"],
  "metrics": [
    {"column": "amount", "aggregation": "sum", "alias": "Total Sales"},
    {"column": null, "aggregation": "count", "alias": "Count"}
  ],
  "extra_config": {
    "drill_down_config": {
      "enabled": true,
      "hierarchy": [
        {"level": 0, "column": "country", "display_name": "Country"},
        {"level": 1, "column": "state", "display_name": "State"},
        {"level": 2, "column": "city", "display_name": "City"}
      ]
    },
    "dimension_drill_down_flags": {
      "country": true,
      "state": true,
      "city": true,
      "product": false
    },
    "column_display_order": [
      "product",
      "total_sales",
      "country",
      "state",
      "city",
      "count"
    ],
    "filters": [],
    "sort": [{"column": "total_sales", "direction": "desc"}],
    "pagination": {"enabled": true, "page_size": 50}
  },
  "drill_down_level": 1,
  "drill_down_path": [
    {"level": 0, "column": "country", "value": "USA", "display_name": "Country"}
  ],
  "offset": 0,
  "limit": 50
}
```

**Response:**
```json
{
  "data": {
    "tableData": [
      {
        "country": "USA",
        "state": "California",
        "city": "Los Angeles",
        "product": "Widget A",
        "total_sales": 15000.00,
        "count": 120
      },
      // ... more rows
    ],
    "columns": ["product", "total_sales", "country", "state", "city", "count"]
  },
  "total_rows": 450,
  "echarts_config": {}
}
```

---

## 4. Migration Strategy

### 4.1 Backward Compatibility

**Old Config Format:**
```json
{
  "extra_config": {
    "drill_down_column": "state",
    "max_drill_levels": 3,
    "aggregation_columns": ["amount", "quantity"]
  }
}
```

**Migration Function:**
```typescript
function migrateDrillDownConfig(
  oldConfig: any,
  dimensions: ChartDimension[]
): {
  dimensions: ChartDimension[];
  extra_config: any;
} {
  const drillDownColumn = oldConfig.drill_down_column;
  const maxLevels = oldConfig.max_drill_levels || 3;

  // If drill_down_column exists but not in dimensions, add it
  let updatedDimensions = [...dimensions];
  const hasDrillColumn = dimensions.some(d => d.column === drillDownColumn);

  if (drillDownColumn && !hasDrillColumn) {
    updatedDimensions.push({
      column: drillDownColumn,
      alias: drillDownColumn,
      enable_drill_down: true,
    });
  } else if (drillDownColumn) {
    // Enable drill-down on existing dimension
    updatedDimensions = updatedDimensions.map(d =>
      d.column === drillDownColumn
        ? { ...d, enable_drill_down: true }
        : d
    );
  }

  // Build new drill_down_config
  const hierarchy = updatedDimensions
    .filter(d => d.enable_drill_down)
    .slice(0, maxLevels)
    .map((d, idx) => ({
      level: idx,
      column: d.column,
      display_name: d.alias || d.column,
      aggregation_columns: oldConfig.aggregation_columns || [],
    }));

  return {
    dimensions: updatedDimensions,
    extra_config: {
      ...oldConfig,
      drill_down_config: {
        enabled: hierarchy.length > 0,
        hierarchy,
      },
      dimension_drill_down_flags: Object.fromEntries(
        updatedDimensions.map(d => [d.column, d.enable_drill_down || false])
      ),
      // Remove old fields
      drill_down_column: undefined,
      max_drill_levels: undefined,
    },
  };
}
```

### 4.2 Migration UI

**Automatic Migration on Edit:**
```typescript
// In ChartBuilder.tsx when loading existing chart
useEffect(() => {
  if (existingChart && existingChart.extra_config) {
    const hasOldDrillDown = existingChart.extra_config.drill_down_column;
    const hasNewDrillDown = existingChart.extra_config.drill_down_config;

    if (hasOldDrillDown && !hasNewDrillDown) {
      // Show migration banner
      setShowMigrationBanner(true);

      // Auto-migrate on first save
      const migrated = migrateDrillDownConfig(
        existingChart.extra_config,
        existingChart.dimensions || []
      );

      // Update form with migrated config
      setFormData({
        ...existingChart,
        ...migrated,
      });
    }
  }
}, [existingChart]);
```

**Migration Banner Component:**
```tsx
{showMigrationBanner && (
  <Alert className="mb-4">
    <Info className="h-4 w-4" />
    <AlertDescription>
      This chart uses the legacy drill-down format. We've automatically upgraded it to
      the new format with more flexibility. You can now:
      <ul className="list-disc ml-6 mt-2">
        <li>Add unlimited dimensions</li>
        <li>Enable drill-down on any dimension</li>
        <li>Reorder columns freely</li>
      </ul>
      <Button variant="link" onClick={() => setShowMigrationBanner(false)}>
        Dismiss
      </Button>
    </AlertDescription>
  </Alert>
)}
```

### 4.3 Database Migration

**No schema changes needed** - all configuration is stored in JSON `extra_config` field.

**Optional: Data Migration Script**
```python
# scripts/migrate_drill_down_configs.py
from ddpui.models.org import Chart

def migrate_all_charts():
    """Migrate all charts with old drill-down format to new format"""
    charts = Chart.objects.filter(
        chart_type='table',
        extra_config__drill_down_column__isnull=False
    )

    migrated_count = 0
    for chart in charts:
        if migrate_chart_drill_down(chart):
            migrated_count += 1

    print(f"Migrated {migrated_count} charts")

def migrate_chart_drill_down(chart: Chart) -> bool:
    """Migrate a single chart's drill-down config"""
    old_config = chart.extra_config
    drill_down_column = old_config.get('drill_down_column')

    if not drill_down_column:
        return False  # Already migrated or no drill-down

    # Build new config
    dimensions = old_config.get('dimensions', [])
    if drill_down_column not in dimensions:
        dimensions.append(drill_down_column)

    new_config = {
        **old_config,
        'dimensions': dimensions,
        'drill_down_config': {
            'enabled': True,
            'hierarchy': [{
                'level': 0,
                'column': drill_down_column,
                'display_name': drill_down_column,
                'aggregation_columns': old_config.get('aggregation_columns', [])
            }]
        },
        'dimension_drill_down_flags': {
            drill_down_column: True
        }
    }

    # Remove old fields
    new_config.pop('drill_down_column', None)
    new_config.pop('max_drill_levels', None)

    chart.extra_config = new_config
    chart.save()

    return True
```

---

## 5. Performance Considerations

### 5.1 Frontend Performance

**Potential Issues:**
1. **Drag-and-drop with many columns**: Re-rendering entire list on drag
2. **Large dimension arrays**: Computing hierarchy on every state change
3. **Complex table rendering**: Many clickable cells with event handlers

**Optimizations:**

1. **Memoize Hierarchy Computation:**
```typescript
const drillDownConfig = useMemo(
  () => computeDrillDownConfig(dimensions),
  [dimensions] // Only recompute when dimensions change
);
```

2. **Virtualize Long Column Lists:**
```typescript
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={400}
  itemCount={dimensions.length}
  itemSize={60}
>
  {({ index, style }) => (
    <DimensionItem
      dimension={dimensions[index]}
      style={style}
    />
  )}
</FixedSizeList>
```

3. **Debounce Column Reordering:**
```typescript
const debouncedOnChange = useMemo(
  () => debounce(onChange, 300),
  [onChange]
);
```

### 5.2 Backend Performance

**Potential Issues:**
1. **Many GROUP BY columns**: Slow queries with 10+ dimensions
2. **Large result sets**: Paginating multi-dimensional tables
3. **Complex drill-down filters**: AND conditions on many columns

**Optimizations:**

1. **Index Dimension Columns:**
```sql
-- Recommend creating composite indexes on common dimension combinations
CREATE INDEX idx_sales_country_state_city
ON sales(country, state, city);
```

2. **Limit Result Set Size:**
```python
# Always apply LIMIT to queries
query_builder.limit(min(payload.limit, 1000))  # Cap at 1000 rows
```

3. **Cache Drill-Down Metadata:**
```python
# Cache column metadata to avoid repeated queries
@lru_cache(maxsize=128)
def get_table_columns(schema: str, table: str):
    # Fetch and cache column info
    pass
```

### 5.3 Monitoring

**Metrics to Track:**
- Average query execution time for multi-dimensional tables
- Number of dimensions per chart (distribution)
- Drill-down depth usage (how deep users go)
- Column reordering frequency

**Logging:**
```python
# Backend logging
logger.info(f"Table query with {len(dimensions)} dimensions, drill_level={drill_level}, execution_time={exec_time}ms")
```

```typescript
// Frontend analytics
analytics.track('table_chart_created', {
  dimension_count: dimensions.length,
  drill_down_enabled_count: dimensions.filter(d => d.enable_drill_down).length,
  metric_count: metrics.length,
});
```

---

## 6. User Experience Design

### 6.1 Progressive Disclosure

**Principle:** Don't overwhelm users with all options at once.

**Implementation:**
1. **Basic Mode**: Show only dimension selection and metric configuration
2. **Advanced Mode** (toggle): Show drill-down toggles, column reordering, advanced filters

```tsx
const [showAdvancedOptions, setShowAdvancedOptions] = useState(false);

return (
  <div>
    {/* Always visible: Basic dimension and metric selection */}
    <DimensionsSelector dimensions={dimensions} onChange={onChange} />
    <MetricsSelector metrics={metrics} onChange={onChange} />

    {/* Collapsible advanced options */}
    <Button onClick={() => setShowAdvancedOptions(!showAdvancedOptions)}>
      {showAdvancedOptions ? 'Hide' : 'Show'} Advanced Options
    </Button>

    {showAdvancedOptions && (
      <>
        <DrillDownConfiguration dimensions={dimensions} onChange={onChange} />
        <ColumnOrderManager dimensions={dimensions} metrics={metrics} onChange={onChange} />
      </>
    )}
  </div>
);
```

### 6.2 Visual Feedback

**Hierarchy Preview:**
```
┌───────────────────────────────────────────┐
│ Drill-Down Hierarchy Preview              │
├───────────────────────────────────────────┤
│                                           │
│  Level 1: Country  →  Level 2: State  →  │
│  Level 3: City                            │
│                                           │
│  Users will click Country values first,  │
│  then State, then City.                  │
└───────────────────────────────────────────┘
```

**Drag-and-Drop Indicators:**
- Drop zone highlights when dragging columns
- Ghost image shows column being dragged
- Visual separator line shows drop position

**Drill-Down Toggle States:**
- **Enabled**: Blue checkmark, dimension name in bold
- **Disabled**: Gray, normal font weight
- **Hover**: Show tooltip: "Enable to make this column drillable"

### 6.3 Error Prevention

**Validation Rules:**
1. Cannot enable drill-down on a dimension without any other drill-enabled dimensions
   - **Error:** "At least one other dimension must have drill-down enabled to create a hierarchy"

2. Cannot remove all dimensions if drill-down is enabled
   - **Error:** "Disable drill-down before removing all dimensions"

3. Column display order must include all selected dimensions and metrics
   - **Auto-fix:** If a dimension/metric is added and not in display order, append it

**Confirmation Dialogs:**
- Removing a dimension with drill-down enabled: "This dimension is part of the drill-down hierarchy. Remove it?"
- Disabling drill-down on all dimensions: "This will disable table drill-down. Continue?"

### 6.4 Help and Documentation

**Inline Help (Tooltips):**
- Dimension selector: "Add columns to group your data by. Order matters for drill-down!"
- Drill-down toggle: "Enable to make this column clickable for deeper analysis"
- Column order: "Drag columns to change how they appear in your table"

**Guided Tour (First-Time Users):**
1. Welcome to table builder
2. Add dimensions to group data
3. Enable drill-down on dimensions
4. Reorder columns (optional)
5. Preview and save

**Examples Library:**
- Pre-configured example: "Sales by Region (3-level drill-down)"
- Pre-configured example: "Product Performance Dashboard"
- Template gallery for common use cases

---

## 7. Success Criteria

### 7.1 Functional Requirements

- [ ] Users can add unlimited dimensions to table charts (tested up to 20 dimensions)
- [ ] Users can enable/disable drill-down on any dimension via toggle
- [ ] Drill-down hierarchy follows dimension order automatically
- [ ] Users can reorder columns via drag-and-drop
- [ ] Metrics remain optional for table creation
- [ ] Existing charts with old drill-down format migrate seamlessly
- [ ] All drill-down functionality works with reordered columns

### 7.2 Performance Requirements

- [ ] Table renders in <500ms with 10 dimensions and 1000 rows
- [ ] Drag-and-drop column reorder completes in <100ms
- [ ] Drill-down navigation (click → filter → render) completes in <1s
- [ ] Backend query execution <2s for 5 dimensions with drill-down filters

### 7.3 Quality Requirements

- [ ] 80%+ unit test coverage on new components
- [ ] 100% of integration test scenarios pass
- [ ] All E2E test scenarios pass in Chrome, Firefox, Safari
- [ ] Zero console errors or warnings in normal usage
- [ ] Accessibility: All interactive elements keyboard-navigable
- [ ] Accessibility: Screen reader announces drill-down hierarchy

### 7.4 User Experience Requirements

- [ ] 90%+ of internal beta testers successfully create drill-down table on first try
- [ ] Average time to create 3-level drill-down table <3 minutes
- [ ] No user-reported confusion about dimension vs. display order
- [ ] Positive feedback on drag-and-drop column reordering

---

## 8. Risks and Mitigation

### 8.1 Technical Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Performance degradation with 15+ dimensions** | High | Medium | Implement virtualization, add dimension count warning at 10+ |
| **Complex migration breaks existing charts** | High | Low | Extensive testing, rollback plan, staged rollout |
| **Drag-and-drop library compatibility issues** | Medium | Low | Use well-tested library (@dnd-kit), fallback to manual reorder |
| **Backend query timeout with complex GROUP BY** | High | Medium | Add query timeout limits, recommend dimension count limits |

### 8.2 User Experience Risks

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| **Users confused by dimension vs. column order** | Medium | High | Clear labeling, help tooltips, hierarchy preview |
| **Users accidentally create too many dimensions** | Low | Medium | Soft limit warning at 10 dimensions, performance tips |
| **Drill-down not intuitive with reordered columns** | Medium | Medium | Visual indicators on clickable cells, clear breadcrumb |

### 8.3 Rollback Plan

**If critical issues arise post-deployment:**

1. **Feature Flag**: Disable new drill-down UI via feature flag
   ```typescript
   const { NEW_DRILL_DOWN_UI } = useFeatureFlags();

   return NEW_DRILL_DOWN_UI
     ? <EnhancedDrillDownConfiguration />
     : <SimpleDrillDownConfiguration />;
   ```

2. **API Backward Compatibility**: Backend continues accepting old format
3. **Data Rollback**: Migration script has inverse function to revert configs
4. **User Communication**: In-app banner explaining temporary rollback

---

## 9. Deployment Plan

### 9.1 Phased Rollout

**Phase 1: Internal Testing (Week 1)**
- Deploy to staging environment
- Internal team testing with real data
- Bug fixes and refinements

**Phase 2: Beta Users (Week 2-3)**
- Deploy to 10% of production users (feature flag)
- Collect feedback via in-app survey
- Monitor performance metrics
- Iterate based on feedback

**Phase 3: General Availability (Week 4)**
- Deploy to 50% of users
- Monitor error rates and performance
- Increase to 100% if no issues

**Phase 4: Migration Push (Week 5+)**
- In-app notifications to users with old drill-down charts
- "Upgrade" button in chart edit mode
- Optional: Scheduled migration of all charts (with user consent)

### 9.2 Monitoring and Alerts

**Key Metrics:**
- Drill-down chart creation rate (before/after)
- Average dimension count per chart
- Error rate on chart save/load
- API response times for multi-dimensional queries

**Alerts:**
- Spike in 5xx errors on charts API
- Average query time >5s for table charts
- >5% increase in user-reported issues

### 9.3 Documentation Updates

**User Documentation:**
- [ ] Updated "Creating Table Charts" guide
- [ ] New "Multi-Dimensional Analysis" tutorial
- [ ] Video: "How to Set Up Drill-Down Tables"
- [ ] FAQ: "Dimension Order vs. Column Display Order"

**Developer Documentation:**
- [ ] API schema updates in OpenAPI spec
- [ ] Architecture decision record (ADR) for toggle-based hierarchy
- [ ] Migration guide for developers

---

## 10. Future Enhancements

### 10.1 Post-MVP Features

**Conditional Drill-Down:**
- Enable drill-down only when certain conditions are met
- Example: Only drill down on rows where `sales > 1000`

**Custom Drill-Down Actions:**
- Configure different actions per dimension (drill-down vs. open modal vs. external link)
- Example: Click country → drill-down, click product → open product details

**Drill-Down with Metric Changes:**
- Change metrics at each drill-down level
- Example: Level 1 shows total sales, Level 2 shows sales by category

**Saved Drill-Down Paths:**
- Save frequently used drill-down paths as bookmarks
- Example: "USA → California → Los Angeles" saved as "LA Analysis"

### 10.2 Advanced Analytics

**Drill-Down Path Analytics:**
- Track which drill-down paths users take most frequently
- Identify popular data exploration patterns
- Suggest common drill-down hierarchies based on usage

**Performance Insights:**
- Recommend optimal dimension order based on data cardinality
- Suggest indexes for frequently drilled dimensions
- Alert on slow-performing drill-down queries

---

## 11. Appendix

### 11.1 File Change Summary

**New Files:**
```
webapp_v2/components/charts/ColumnOrderManager.tsx
webapp_v2/components/charts/__tests__/ColumnOrderManager.test.tsx
webapp_v2/components/charts/__tests__/DimensionsSelector.unlimited.test.tsx
webapp_v2/components/charts/__tests__/DimensionsSelector.drillDownToggle.test.tsx
webapp_v2/__tests__/integration/table-drill-down-flow.test.tsx
webapp_v2/e2e/table-unlimited-dimensions.spec.ts
webapp_v2/e2e/table-drill-down-navigation.spec.ts
webapp_v2/e2e/table-column-rearrangement.spec.ts
DDP_backend/ddpui/tests/core/test_charts_service_unlimited_dimensions.py
DDP_backend/ddpui/tests/core/test_charts_service_drill_down.py
```

**Modified Files:**
```
webapp_v2/types/charts.ts
  - Update ChartDimension interface
  - Add column_display_order to ChartCreate

webapp_v2/components/charts/DimensionsSelector.tsx
  - Add drill-down toggle per dimension
  - Add drag-and-drop reordering
  - Add hierarchy preview

webapp_v2/components/charts/SimpleDrillDownConfiguration.tsx
  - Integrate ColumnOrderManager
  - Update UI for toggle-based hierarchy

webapp_v2/components/charts/ChartBuilder.tsx
  - Compute drill_down_config from dimensions
  - Include column_display_order in payload

webapp_v2/components/charts/TableChart.tsx
  - Use column_display_order for rendering
  - Maintain drill-down detection logic

DDP_backend/ddpui/schemas/chart_schema.py
  - Add comments on dimension_col deprecation

DDP_backend/ddpui/core/charts/charts_service.py
  - Log deprecation warnings for dimension_col usage
```

**Deprecated Files:**
```
(None - SimpleDrillDownConfiguration will be updated, not replaced)
```

### 11.2 Dependencies

**New Frontend Dependencies:**
```json
{
  "@dnd-kit/core": "^6.0.8",
  "@dnd-kit/sortable": "^7.0.2",
  "@dnd-kit/utilities": "^3.2.1"
}
```

**Backend:** No new dependencies required

### 11.3 Glossary

- **Dimension**: A column by which data is grouped (e.g., Country, Product)
- **Metric**: An aggregated value (e.g., SUM of sales, COUNT of orders)
- **Drill-Down**: Navigation from aggregate data to detailed data by clicking values
- **Hierarchy**: The ordered sequence of dimensions that defines drill-down path
- **Display Order**: Visual arrangement of columns in the rendered table
- **Dimension Order**: Sequence of dimensions in the configuration (defines hierarchy)
- **Breadcrumb**: Navigation trail showing current drill-down path
- **Toggle**: Checkbox to enable/disable drill-down on a dimension

---

## End of Document

**Next Steps:**
1. Review this plan with the development team
2. Estimate effort for each phase
3. Prioritize requirements if needed
4. Begin Phase 1 implementation (Foundation)

**Questions for Stakeholders:**
1. Should we enforce a maximum dimension count (e.g., 20)?
2. Do we need drill-down analytics before general release?
3. Should migration be automatic or user-triggered?
4. What's the target completion date for MVP?
