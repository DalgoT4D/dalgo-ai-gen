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
| Milestone 3 | Mar 25-27 (Tue-Thu) | 3 days | Backend Integration & Migration |
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

**Note:** Existing dashboards are migrated to tabs structure via backend script (see Milestone 3).

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

## Milestone 3 — Backend Integration & Migration

**Mar 25-27 (Tue-Thu) | Goal:** Add tabs field, migrate existing dashboards to tabs structure.

### What We're Building

| Feature | Description |
|---------|-------------|
| Tabs Field | New `tabs` JSONField on Dashboard model |
| Migration Script | Migrate all existing dashboards to tabs structure |
| Clear Root Level | Set root `layout_config` and `components` to empty (keep fields) |
| API Updates | Handle tabs structure in all endpoints |

### How It Works

1. Add new `tabs` JSONField to Dashboard model
2. Run migration script to move existing dashboard data into tabs
3. Clear root level `layout_config` and `components` (set to empty, not removed)
4. All dashboards now use `tabs` for data, root level fields are always empty

### Data Model

**Dashboard Model (after adding tabs)**

| Field | Type | Description |
|-------|------|-------------|
| id | BigInt PK | Primary key |
| title | VARCHAR(255) | Dashboard title |
| layout_config | JSONField | Always empty `[]` (kept for safety) |
| components | JSONField | Always empty `{}` (kept for safety) |
| tabs | JSONField | **NEW** — Array of tab objects (actual data here) |
| ... | ... | Other existing fields |

**Note:** `layout_config` and `components` at root level are kept but always empty. All actual data lives inside `tabs[].layout_config` and `tabs[].components`.

**Tabs JSON Structure**

The `tabs` field is a simple array of tab objects. `activeTabId` is managed in frontend state only (not persisted).

```json
[
  {
    "id": "tab-1",
    "title": "Untitled Tab 1",
    "layout_config": [...],
    "components": {...}
  },
  {
    "id": "tab-2",
    "title": "Untitled Tab 2",
    "layout_config": [...],
    "components": {...}
  }
]
```

**Field Descriptions**

| Field | Type | Description |
|-------|------|-------------|
| id | String | Unique tab identifier |
| title | String | Tab display name (max 50 chars) |
| layout_config | Array | Grid layout items for this tab |
| components | Object | Component configs for this tab |

**Note:** Tab order is determined by array index. `activeTabId` is managed in frontend state, defaulting to first tab on load.

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | /api/dashboards/:id/ | Returns dashboard with `tabs` array |
| PUT | /api/dashboards/:id/ | Accepts `tabs` in request body, validates structure |
| POST | /api/dashboards/ | Creates dashboard with default "Untitled Tab 1" in `tabs` |

### Validation Rules

| Rule | Validation |
|------|------------|
| Max tabs | No limit |
| Tab title | Required, max 50 characters |
| Tab ID | Required, unique within dashboard |
| At least one tab | Required (tabs array cannot be empty) |

### Migration Approach

After running the migration script, ALL dashboards will have the `tabs` structure. No backward compatibility logic needed in frontend.

| Dashboard State | `tabs` Field | `layout_config` / `components` |
|-----------------|--------------|-------------------------------|
| Before migration | `null` or `[]` | Contains data |
| After migration | Contains data | Empty `[]` / `{}` |

### Deliverables

- Add `tabs` field to Dashboard model
- Run migration script to convert existing dashboards
- Update GET/PUT API to handle tabs structure
- Frontend: save/load tabs on dashboard save/load

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

### Current Dashboard Schema (Before Tabs)

```python
# Backend Model: ddpui/models/dashboard.py
class Dashboard(models.Model):
    id = models.BigAutoField(primary_key=True)
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True, null=True)
    dashboard_type = models.CharField(max_length=20)  # "native" or "superset"
    grid_columns = models.IntegerField(default=12)
    target_screen_size = models.CharField(max_length=20)  # desktop, tablet, mobile, a4
    layout_config = models.JSONField(default=list)    # Grid positions
    components = models.JSONField(default=dict)       # Component configs
    filter_layout = models.CharField(max_length=20)   # vertical or horizontal
    is_published = models.BooleanField(default=False)
    is_public = models.BooleanField(default=False)
    is_org_default = models.BooleanField(default=False)
    # ... other fields (created_by, org, timestamps, etc.)
```

### New Dashboard Schema (After Tabs Feature)

After migration, `layout_config` and `components` at root level are **always empty**. All data lives inside `tabs`.

```python
class Dashboard(models.Model):
    id = models.BigAutoField(primary_key=True)
    title = models.CharField(max_length=255)
    description = models.TextField(blank=True, null=True)
    dashboard_type = models.CharField(max_length=20)
    grid_columns = models.IntegerField(default=12)
    target_screen_size = models.CharField(max_length=20)
    layout_config = models.JSONField(default=list)    # Always empty []
    components = models.JSONField(default=dict)       # Always empty {}
    tabs = models.JSONField(default=list)             # NEW: All data here
    filter_layout = models.CharField(max_length=20)
    is_published = models.BooleanField(default=False)
    is_public = models.BooleanField(default=False)
    is_org_default = models.BooleanField(default=False)
    # ... other fields
```

### Approaches Considered

**Approach 1: Data Inside Tabs (Selected)**
```json
{
  "id": 1,
  "title": "My Dashboard",
  "dashboard_type": "native",
  "grid_columns": 12,
  "target_screen_size": "desktop",
  "filter_layout": "vertical",
  "is_published": false,
  "layout_config": [],
  "components": {},
  "tabs": [
    {
      "id": "tab-1",
      "title": "Overview",
      "layout_config": [
        { "i": "chart-123", "x": 0, "y": 0, "w": 6, "h": 4 }
      ],
      "components": {
        "chart-123": { "type": "chart", "config": { "chartId": 1 } }
      }
    }
  ]
}
```

**Approach 2: Children Array (Superset-style)**
```json
{
  "id": 1,
  "title": "My Dashboard",
  "dashboard_type": "native",
  "grid_columns": 12,
  "target_screen_size": "desktop",
  "filter_layout": "vertical",
  "is_published": false,
  "layout_config": [
    { "i": "chart-123", "tabId": "tab-1", "x": 0, "y": 0, "w": 6, "h": 4 },
    { "i": "chart-123", "tabId": "tab-2", "x": 6, "y": 0, "w": 4, "h": 6 }
  ],
  "components": {
    "chart-123": { "type": "chart", "config": { "chartId": 1 } }
  },
  "tabs": [
    { "id": "tab-1", "title": "Tab 1", "children": ["chart-123"] },
    { "id": "tab-2", "title": "Tab 2", "children": ["chart-123"] }
  ]
}
```

**Note:** Tab order is determined by array index. `activeTabId` is managed in frontend state only (not persisted). Default to first tab on load.

### Why We Chose Approach 1

1. **Direct Access** — Each tab contains its own `layout_config` and `components`, allowing O(1) direct access without filtering through shared arrays.

2. **Tab-Wise Isolation** — Each tab is self-contained. Modifying one tab doesn't affect others, making operations simpler and debugging easier.

3. **Flexible Chart Positioning** — Same chart can have different positions and sizes in different tabs since each tab maintains independent layout coordinates.

4. **Aligned with React Grid Layout** — Structure maps directly to React Grid Layout's `x, y, w, h` format without requiring transformation on every render.

### Migration Strategy (Backend Script)

Run a Django migration script to convert all existing dashboards to tabs structure.

**Step 1: Add tabs field to Dashboard model**
```python
# Migration: Add tabs field
class Migration(migrations.Migration):
    operations = [
        migrations.AddField(
            model_name='dashboard',
            name='tabs',
            field=models.JSONField(default=list),
        ),
    ]
```

**Step 2: Move existing data into tabs, clear root level**
```python
import time

def migrate_dashboards_to_tabs(apps, schema_editor):
    Dashboard = apps.get_model('ddpui', 'Dashboard')

    for dashboard in Dashboard.objects.all():
        if not dashboard.tabs:  # Only migrate if tabs is empty
            # Move root level data into first tab
            dashboard.tabs = [{
                'id': f'tab-{int(time.time() * 1000)}',  # Timestamp-based ID
                'title': 'Untitled Tab 1',
                'layout_config': dashboard.layout_config or [],
                'components': dashboard.components or {}
            }]
            # Clear root level (keep fields but set to empty)
            dashboard.layout_config = []
            dashboard.components = {}
            dashboard.save()

class Migration(migrations.Migration):
    operations = [
        migrations.RunPython(migrate_dashboards_to_tabs),
    ]
```

**Tab ID Generation (consistent with component IDs):**

| Context | Method | Example |
|---------|--------|---------|
| Backend (migration) | `f'tab-{int(time.time() * 1000)}'` | `"tab-1710901234567"` |
| Frontend (new tab) | `` `tab-${Date.now()}` `` | `"tab-1710901234567"` |

Same pattern as existing component IDs: `chart-${Date.now()}`

**What this does:**
1. For each existing dashboard, copy `layout_config` and `components` into a new tab called "Untitled Tab 1"
2. Clear root level fields (set to empty `[]` and `{}`)
3. Root level fields are **kept in database** (not removed) but always empty going forward


### API Response (After Migration)

After migration, API returns dashboard with empty root level fields and data inside `tabs`:

```json
{
  "id": 1,
  "title": "My Dashboard",
  "dashboard_type": "native",
  "grid_columns": 12,
  "layout_config": [],
  "components": {},
  "tabs": [
    {
      "id": "tab-1-1",
      "title": "Untitled Tab 1",
      "layout_config": [...],
      "components": {...}
    }
  ]
}
```

Root level `layout_config` and `components` are always empty. Frontend uses only `tabs[].layout_config` and `tabs[].components`.

### Frontend Active Tab State

`activeTabId` is managed in React state only (not persisted in database):

```typescript
// Default to first tab on load
const [activeTabId, setActiveTabId] = useState(dashboard.tabs[0]?.id);

// Switch tab
const handleTabChange = (tabId: string) => {
  setActiveTabId(tabId);
};
```

---

## Deliverables by Date

| Date | Milestone | Deliverable |
|------|-----------|-------------|
| Mar 20 (Thu) | M1 – Complete | Tab Bar UI, add/remove/rename tabs, tab switching |
| Mar 24 (Mon) | M2 – Complete | Charts added inside tabs, duplicate chart support |
| Mar 27 (Thu) | M3 – Complete | Backend `tabs` field, migration script, clear root level fields |
| Apr 1 (Tue) | M4 – Complete | Preview mode logic, full testing, ready for deployment |

---

## Key Rules Summary

| Rule | Details |
|------|---------|
| Default tab | New dashboards start with "Untitled Tab 1" |
| Charts location | Always inside tabs (not directly on dashboard) |
| Max tabs | No limit |
| Duplicate charts | Allowed in different tabs |
| Existing dashboards | Migrated via backend script to tabs structure |
| Preview (1 tab) | Tab bar hidden, normal dashboard look |
| Preview (2 or more tabs) | Tab bar visible, click to switch |

---

## Key Risks & Mitigations

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| Migration script failure | Low | Test on staging first; backup database before migration |
| Performance with many charts | Low | Same rendering as current dashboards |
| User confusion with new UI | Medium | Single tab = hidden tab bar; familiar experience |
| Data loss on tab delete | Medium | Confirm dialog before deleting tab with charts |

---

**Dalgo Engineering Team • March 2026**
