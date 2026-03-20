# DALGO DASHBOARD TABS
## Technical Specification
**Version 1.0 • March 2026**

---

## What We're Building

A tabs system for dashboards that allows users to organize charts into multiple tabs — replacing cluttered single-view dashboards with organized, tabbed views.

---

## Delivery Timeline

| Milestone | Dates | Duration | Focus |
|-----------|-------|----------|-------|
| Milestone 1 | Mar 19-20 (Wed-Thu) | 2 days | Tab Bar UI & Tab Management |
| Milestone 2 | Mar 21, 24 (Fri, Mon) | 2 days | Add Charts Inside Tabs |
| Milestone 3 | Mar 25-27 (Tue-Thu) | 3 days | Backend Integration & Backward Compatibility |
| Milestone 4 | Mar 28, 31, Apr 1 (Fri, Mon, Tue) | 3 days | Preview Mode Logic & Testing |

**Total: 10 working days • Start: March 19 • End: April 1**
**Note:** Weekends (Sat-Sun) excluded

---

## Milestone 1 — Tab Bar UI & Tab Management

**Mar 19-20 (Wed-Thu) | Goal:** Users can see tabs, switch between them, add new tabs, rename, and remove tabs.

### What We're Building

| Feature | Description |
|---------|-------------|
| Tab Bar UI | Horizontal bar showing all tabs with active tab highlighted |
| Default Tab | New dashboards start with "Untitled Tab 1", new tabs are "Untitled Tab 2", "Untitled Tab 3", etc. |
| Add Tab | Click "+" to add a new tab (no limit) |
| Remove Tab | Click "x" to remove a tab (cannot remove last tab) |
| Rename Tab | Double-click tab name to rename |
| Switch Tab | Click on tab to switch view |

### How It Works

When a user creates a new dashboard, it starts with a single "Untitled Tab 1". The tab bar appears at the top of the dashboard canvas. Users can add unlimited tabs (named "Untitled Tab 2", "Untitled Tab 3", etc.). Each tab maintains its own layout and charts independently. Switching tabs instantly shows that tab's content.

When an existing dashboard (without tabs) is opened in edit mode, all existing charts are automatically displayed inside a default "Untitled Tab 1" — providing a unified editing experience.

### UI Specifications

| Element | Behavior |
|---------|----------|
| Tab Bar | Horizontal, below dashboard title, above chart area |
| Active Tab | Highlighted with bottom border or background color |
| Add Button (+) | Appears after last tab, visible only in edit mode |
| Remove Button (x) | Appears on each tab (except when only 1 tab), visible only in edit mode |
| Max Tabs | No limit |
| Tab Name | Max 50 characters |

### Deliverables

- Tab Bar component with static tabs
- Tab switching functionality
- Add tab, remove tab, rename tab
- Testing & bug fixes

---

## Milestone 2 — Add Charts Inside Tabs

**Mar 21, 24 (Fri, Mon) | Goal:** Charts are added inside the active tab, not directly on the dashboard.

### What We're Building

| Feature | Description |
|---------|-------------|
| Chart Addition Flow | User selects tab → clicks "Add Chart" → chart appears in active tab |
| Per-Tab Layout | Each tab has its own grid layout and components |
| Duplicate Charts | Same chart can exist in multiple tabs |

### How It Works

When a user clicks "Add Chart", the system checks which tab is currently active and adds the chart to that tab's layout. Each tab maintains its own `layout_config` and `components` structure. Users can add the same chart to multiple tabs if needed — each instance is independent.

### Data Flow

| Step | Action |
|------|--------|
| 1 | User selects a tab (clicks on it) |
| 2 | User clicks "Add Chart" button |
| 3 | Chart selector modal opens |
| 4 | User selects a chart |
| 5 | Chart is added to active tab's `layout_config` and `components` |
| 6 | Tab content re-renders with new chart |

### Deliverables

- Update "Add Chart" flow to target active tab
- Update "Add Text" flow to target active tab
- Duplicate chart support
- Per-tab layout management

---

## Milestone 3 — Backend Integration & Backward Compatibility

**Mar 25-27 (Tue-Thu) | Goal:** Persist tabs to database. Existing dashboards continue to work without changes.

### What We're Building

| Feature | Description |
|---------|-------------|
| Tabs Field | New `tabs` JSONField on Dashboard model |
| Save/Load Tabs | API updates to handle tabs structure |
| Backward Compatibility | Existing dashboards (tabs = null) render normally |
| Auto-Migration on Edit | Existing dashboards get wrapped in "Untitled Tab" when edited |

### How It Works

A new `tabs` JSONField is added to the Dashboard model. When saving a dashboard with tabs, the entire tabs structure is stored in this field. When loading, if `tabs` is null, the dashboard renders using the existing `layout_config` and `components` (backward compatible). If `tabs` has data, it renders using the tabs structure.

### Data Model

**Dashboard Model (updated)**

| Field | Type | Description |
|-------|------|-------------|
| id | BigInt PK | Primary key |
| title | VARCHAR(255) | Dashboard title |
| layout_config | JSONField | Grid layout (used by old dashboards) |
| components | JSONField | Component configs (used by old dashboards) |
| tabs | JSONField, nullable | **NEW** — Tabs structure. NULL = old dashboard |

**Tabs JSON Structure**

```json
{
  "tabs": [
    {
      "id": "tab-1",
      "title": "Untitled Tab 1",
      "order": 0,
      "layout_config": [...],
      "components": {...}
    },
    {
      "id": "tab-2",
      "title": "Untitled Tab 2",
      "order": 1,
      "layout_config": [...],
      "components": {...}
    }
  ],
  "activeTabId": "tab-1"
}
```

**Field Descriptions**

| Field | Type | Description |
|-------|------|-------------|
| tabs | Array | List of tab objects |
| tabs[].id | String | Unique tab identifier |
| tabs[].title | String | Tab display name (max 50 chars) |
| tabs[].order | Integer | Tab position (0-indexed) |
| tabs[].layout_config | Array | Grid layout items for this tab |
| tabs[].components | Object | Component configs for this tab |
| activeTabId | String | Currently active tab ID |

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/dashboards/:id/ | Returns dashboard with `tabs` field (null or data) |
| PUT | /api/dashboards/:id/ | Accepts `tabs` in request body, validates structure |
| POST | /api/dashboards/ | Creates dashboard with default "Untitled Tab 1" in `tabs` |

### Validation Rules

| Rule | Validation |
|------|------------|
| Max tabs | No limit |
| Tab title | Required, max 50 characters |
| Tab ID | Required, unique within dashboard |
| At least one tab | Required when `tabs` is not null |

### Backward Compatibility

| Dashboard State | `tabs` Field | Rendering |
|-----------------|--------------|-----------|
| Old (existing) | `null` | Use `layout_config` & `components` directly |
| New (with tabs) | Has data | Use `tabs` structure |
| Old opened in edit | `null` | Frontend wraps in "Untitled Tab 1" |
| Old saved after edit | Has data | Converted to tabs structure |

### Deliverables

- Add `tabs` field to Dashboard model, migration
- Update GET/PUT API to handle tabs
- Frontend: save/load tabs on dashboard save/load
- Backward compatibility for existing dashboards

---

## Milestone 4 — Preview Mode Logic & Testing

**Mar 28, 31, Apr 1 (Fri, Mon, Tue) | Goal:** Preview mode hides tab bar when only 1 tab. Full end-to-end testing.

### What We're Building

| Feature | Description |
|---------|-------------|
| Single Tab Preview | When 1 tab exists, hide tab bar, show charts directly |
| Multi-Tab Preview | When 2 or more tabs exist, show tab bar with tab switching |
| No Edit Controls | No add/remove/rename buttons visible in preview |

### How It Works

In preview (view) mode, the system checks how many tabs exist. If only 1 tab, the tab bar is hidden completely and charts render as a normal dashboard. If 2 or more tabs exist, the tab bar appears (without edit controls) and users can click to switch between tabs.

### Preview Mode Behavior

| Tabs Count | Tab Bar | Edit Controls | User Experience |
|------------|---------|---------------|-----------------|
| 1 tab | Hidden | Hidden | Normal dashboard view |
| 2 or more tabs | Visible | Hidden | Tab switching only |

### Deliverables

- Preview mode: conditional tab bar rendering
- End-to-end testing: new & existing dashboards
- Bug fixes, UX polish

---

## Architecture Decision: Data Structure Analysis

This section documents the technical analysis performed to select the optimal data structure for the tabs feature.

### Approaches Considered

**Approach 1: Data Inside Tabs**
```json
{
  "layout_config": [],
  "components": {},
  "tabs": {
    "tabs": [
      {
        "id": "tab-1",
        "title": "Tab 1",
        "order": 0,
        "layout_config": [{ "i": "chart-123", "x": 0, "y": 0, "w": 6, "h": 4 }],
        "components": { "chart-123": { "type": "chart", "config": { "chartId": 1 } } }
      }
    ],
    "activeTabId": "tab-1"
  }
}
```

**Approach 2: Children Array (Superset-style)**
```json
{
  "layout_config": [{ "i": "chart-123", "x": 0, "y": 0, "w": 6, "h": 4 }],
  "components": { "chart-123": { "type": "chart", "config": { "chartId": 1 } } },
  "tabs": {
    "tabs": [
      { "id": "tab-1", "title": "Tab 1", "order": 0, "children": ["chart-123"] }
    ],
    "activeTabId": "tab-1"
  }
}
```

### Comparison Matrix

#### 1. Time Complexity

| Operation | Approach 1 (Data in Tabs) | Approach 2 (Children Array) |
|-----------|---------------------------|----------------------------|
| Get active tab components | O(1) - Direct access | O(n) - Filter by children IDs |
| Add chart to tab | O(1) - Direct insert | O(1) - Insert to root + push to children |
| Delete chart from tab | O(1) - Direct delete | O(n) - Find in root + remove from children |
| Move chart between tabs | O(n) - Copy + delete | O(1) - Move ID between children arrays |
| Render tab | O(k) where k = charts in tab | O(n) - Filter n total charts |
| Check chart exists in tab | O(k) where k = charts in tab | O(c) where c = children length |

**Winner: Approach 1** — Faster for most read operations (rendering is most frequent)

#### 2. Space Complexity

| Factor | Approach 1 | Approach 2 |
|--------|------------|------------|
| Same chart in multiple tabs | Duplicated data | Single reference |
| Storage per chart | Full object per tab | Full object once + ID refs |
| API payload size | Larger if duplicates | Smaller, normalized |

**Example: Same chart in 3 tabs**
- Approach 1: 3 × (layout + component) ≈ 3KB
- Approach 2: 1 × (layout + component) + 3 × ID ≈ 1.1KB

**Winner: Approach 2** — Normalized, no duplication

#### 3. Data Integrity & Consistency

| Factor | Approach 1 | Approach 2 |
|--------|------------|------------|
| Single source of truth | No - data in multiple tabs | Yes - root level only |
| Update chart config | Must update in ALL tabs | Update once at root |
| Orphan data risk | Low | Possible (ID in children but deleted from root) |
| Sync issues | Can have inconsistent copies | None (single copy) |

**Winner: Approach 2** — Single source of truth

#### 4. Backend & Database

| Factor | Approach 1 | Approach 2 |
|--------|------------|------------|
| Migration complexity | High - transform all dashboards | Low - add tabs field only |
| JSONField query | Complex nested queries | Simpler flat queries |
| Validation | Each tab needs validation | Single validation at root |
| Backward compatibility | Needs transformation layer | Native (tabs: null = show all) |

**Winner: Approach 2** — Simpler backend changes

#### 5. Frontend Rendering Performance

| Factor | Approach 1 | Approach 2 |
|--------|------------|------------|
| Tab switch | Direct render (fast) | Filter then render |
| React re-renders | Isolated per tab | May trigger parent re-renders |
| Memoization | Easy per tab | Need careful filtering memo |

**Approach 1 - Tab Switch Code:**
```typescript
const activeTab = tabs.find(t => t.id === activeTabId);
return <Grid layout={activeTab.layout_config} />  // Direct access
```

**Approach 2 - Tab Switch Code:**
```typescript
const activeTab = tabs.find(t => t.id === activeTabId);
const filteredLayout = layout_config.filter(l => activeTab.children.includes(l.i));
return <Grid layout={filteredLayout} />  // Requires filtering
```

**Winner: Approach 1** — Faster rendering, no filtering overhead

#### 6. Code Maintainability

| Factor | Approach 1 | Approach 2 |
|--------|------------|------------|
| Mental model | "Tab owns its data" | "Root owns data, tabs group" |
| Code paths | Single path per tab | Always filter |
| Testing | Isolated tab tests | Need integration tests |
| Debugging | Clear data location | Need to trace references |

**Winner: Approach 1** — Simpler mental model

#### 7. Scalability

| Scenario | Approach 1 | Approach 2 |
|----------|------------|------------|
| 100 charts, 10 tabs | 100 items split across tabs | Filter 100 items per tab switch |
| Large dashboards | Each tab loads only its data | Must load all, then filter |
| Lazy loading potential | Yes - load tab on demand | Harder - data at root |

**Winner: Approach 1** — Better for large dashboards

#### 8. Migration & Rollback Safety

| Factor | Approach 1 | Approach 2 |
|--------|------------|------------|
| Initial migration | Transform existing data | Add tabs field, keep existing |
| Rollback safety | Hard - need reverse transform | Easy - remove tabs field |
| Production risk | Higher | Lower |
| Gradual rollout | Complex | Simple (feature flag) |

**Winner: Approach 2** — Safer deployment

#### 9. Critical Feature: Same Chart in Multiple Tabs

| Requirement | Approach 1 | Approach 2 |
|-------------|------------|------------|
| Same chart, different positions | ✅ Supported | ❌ Not possible |
| Same chart, different sizes | ✅ Supported | ❌ Not possible |
| Independent tab layouts | ✅ Fully independent | ❌ Shared layout |

**Critical Finding:** If the same chart appears in Tab 1 at position (0,0) and Tab 2 at position (6,0), Approach 2 **cannot handle this** because `layout_config` is shared at root level.

### Final Summary

| Category | Winner |
|----------|--------|
| Time Complexity (Read) | Approach 1 |
| Space Complexity | Approach 2 |
| Data Integrity | Approach 2 |
| Backend Simplicity | Approach 2 |
| Frontend Performance | Approach 1 |
| Code Maintainability | Approach 1 |
| Scalability | Approach 1 |
| Migration Safety | Approach 2 |
| **Same chart, different positions** | **Only Approach 1** |

### Decision: Approach 1 (Data Inside Tabs)

**Rationale:**
1. **Flexibility**: Same chart can have different positions/sizes in different tabs
2. **Performance**: O(1) access to active tab data, no filtering required
3. **Scalability**: Large dashboards benefit from tab-isolated data
4. **Simpler Code**: No filtering logic on every render
5. **Future-proof**: Each tab is independent, easy to add tab-specific features

**Migration Strategy (Frontend):**
```typescript
// When loading existing dashboard (tabs: null)
if (!dashboard.tabs) {
  dashboard.tabs = {
    tabs: [{
      id: generateTabId(),
      title: "Untitled Tab 1",
      order: 0,
      layout_config: dashboard.layout_config,
      components: dashboard.components
    }],
    activeTabId: "tab-1"
  };
  // Clear root level after migration
  dashboard.layout_config = null;
  dashboard.components = null;
}
```

### API Response Handling

After migration, dashboards with tabs will have empty/null `layout_config` and `components` at root level. To maintain a clean API response, the backend serializer should exclude these fields when `tabs` exists.

**Problem:** Showing empty fields in API response is not clean for production.
```json
{
  "id": 1,
  "title": "My Dashboard",
  "layout_config": null,
  "components": null,
  "tabs": { ... }
}
```

**Solution:** Backend excludes empty root fields when tabs exist.

**Django Serializer Implementation:**
```python
class DashboardSerializer(serializers.ModelSerializer):
    class Meta:
        model = Dashboard
        fields = ['id', 'title', 'layout_config', 'components', 'tabs', ...]

    def to_representation(self, instance):
        data = super().to_representation(instance)
        # If tabs exist, exclude empty root-level layout fields from response
        if data.get('tabs'):
            data.pop('layout_config', None)
            data.pop('components', None)
        return data
```

**Clean API Response (with tabs):**
```json
{
  "id": 1,
  "title": "My Dashboard",
  "tabs": {
    "tabs": [
      {
        "id": "tab-1",
        "title": "Untitled Tab 1",
        "order": 0,
        "layout_config": [...],
        "components": {...}
      }
    ],
    "activeTabId": "tab-1"
  }
}
```

**API Response (legacy dashboard without tabs):**
```json
{
  "id": 2,
  "title": "Old Dashboard",
  "layout_config": [...],
  "components": {...},
  "tabs": null
}
```

| Dashboard Type | `layout_config` in response | `components` in response | `tabs` in response |
|----------------|-----------------------------|--------------------------|--------------------|
| Legacy (no tabs) | ✅ Included | ✅ Included | `null` |
| New (with tabs) | ❌ Excluded | ❌ Excluded | ✅ Included |

This ensures clean API responses while maintaining backward compatibility for legacy dashboards.

---

## Deliverables by Date

| Date | Milestone | Deliverable |
|------|-----------|-------------|
| Mar 20 (Thu) | M1 – Complete | Tab Bar UI, add/remove/rename tabs, tab switching |
| Mar 24 (Mon) | M2 – Complete | Charts added inside tabs, duplicate chart support |
| Mar 27 (Thu) | M3 – Complete | Backend `tabs` field, API updates, backward compatibility |
| Apr 1 (Tue) | M4 – Complete | Preview mode logic, full testing, ready for deployment |

---

## Key Rules Summary

| Rule | Details |
|------|---------|
| Default tab | New dashboards start with "Untitled Tab 1" |
| Charts location | Always inside tabs (not directly on dashboard) |
| Max tabs | No limit |
| Duplicate charts | Allowed in different tabs |
| Existing dashboards | No migration needed, continue to work |
| Edit mode (existing) | Existing charts shown in "Untitled Tab 1" |
| Preview (1 tab) | Tab bar hidden, normal dashboard look |
| Preview (2 or more tabs) | Tab bar visible, click to switch |

---

## Key Risks & Mitigations

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Existing dashboard breakage | Low | Backward compat with null check; extensive testing |
| Performance with many charts | Low | Same rendering as current dashboards |
| User confusion with new UI | Medium | Single tab = hidden tab bar; familiar experience |
| Data loss on tab delete | Medium | Confirm dialog before deleting tab with charts |

---

**Dalgo Engineering Team • March 2026**
