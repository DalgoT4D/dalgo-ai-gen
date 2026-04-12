# DALGO DASHBOARD TABS
## Technical Specification
**Version 1.0 • March 2026**

---

## Progress Status

> **Instructions for Claude:** Read this section first to understand current progress. Continue from the current milestone.

| Milestone | Status | Notes |
|-----------|--------|-------|
| M1 — Tab Bar UI | 🟢 Complete | All components created, TabBar integrated |
| M2 — Backend API | 🟢 Complete | Schema, service, schema migration, API updated |
| M3 — Add Charts | 🟢 Complete | State sync, save/load, tab handlers, reusable utils |
| M4 — Migration & Testing | 🟡 In Progress | Preview Mode done, Data Migration remaining |

**Current Focus:** Milestone 4 — Data Migration Script

**Last Updated:** March 26, 2026

**Status Legend:** 🔴 Not Started | 🟡 In Progress | 🟢 Complete

---

## What We're Building

A tabs system for dashboards that allows users to organize charts into multiple tabs — replacing cluttered single-view dashboards with organized, tabbed views.

---

## Delivery Timeline

| Milestone | Dates | Duration | Focus |
|-----------|-------|----------|-------|
| Milestone 1 | Mar 19-20 (Wed-Thu) | 2 days | Tab Bar UI & Tab Management (Frontend) |
| Milestone 2 | Mar 21, 24 (Fri, Mon) | 2 days | Backend Schema & API Updates |
| Milestone 3 | Mar 25-27 (Tue-Thu) | 3 days | Add Charts Inside Tabs (Frontend) |
| Milestone 4 | Mar 28, 31, Apr 1 (Fri, Mon, Tue) | 3 days | Migration, Preview Mode & Testing |

**Total: 10 working days • Start: March 19 • End: April 1**
**Note:** Weekends (Sat-Sun) excluded

### Milestone Dependencies

```
M1 (Tab Bar UI) → M2 (Backend API) → M3 (Add Charts) → M4 (Migration & Testing)
      ↓                  ↓                  ↓                    ↓
 Frontend only     API accepts tabs    Can save tabs      Migrate existing
 (local state)     (persistence)       (full flow)        dashboards
```

---

## Milestone 1 — Tab Bar UI & Tab Management

**Mar 19-20 (Wed-Thu) | Goal:** Users can see tabs, switch between them, add new tabs, rename, and remove tabs.

> **Note:** M1 works in frontend local state only. Data is NOT persisted to backend yet (that requires M2).

---

### What We're Building

**Tab Bar UI** — horizontal bar below dashboard title showing all tabs; active tab highlighted with bottom border

**Add Tab** — click "+" button to create new tab with auto-generated name ("Untitled Tab 2", "Untitled Tab 3", etc.); no limit on number of tabs

**Remove Tab** — click "x" button to delete a tab; shows confirmation dialog with warning if tab contains charts; last tab cannot be deleted

**Rename Tab** — double-click tab name to edit inline; max 50 characters; Enter to save, Escape to cancel

**Switch Tab** — click any tab to display that tab's layout and charts; each tab maintains independent content

**Edit vs Preview Mode** — add/remove/rename controls visible only in edit mode; in preview mode with single tab, tab bar hidden completely for clean dashboard view

---

### How It Works

When a user creates a new dashboard, it starts with a single tab called "Untitled Tab 1". The tab bar appears below the dashboard title and above the chart canvas. Each tab maintains its own layout and charts independently. Clicking a tab instantly switches the view to display that tab's content.

In edit mode, users can add unlimited tabs using the "+" button, rename tabs by double-clicking, and remove tabs via the "x" button with a confirmation dialog. The last remaining tab cannot be deleted. Tab names are limited to 50 characters.

In preview or view mode, all editing controls are hidden. If only one tab exists, the tab bar is hidden completely — the dashboard looks exactly as it did before tabs were introduced. If multiple tabs exist, users can click to switch between them but cannot modify anything.

---

### File Details

#### NEW: `webapp_v2/components/dashboard/tabs/tab-utils.ts`

**Purpose:** Utility functions for tab operations

**What it exports:**
```typescript
TAB_TITLE_MAX_LENGTH = 50        // Max characters for tab title
generateTabId()                   // Returns "tab-{timestamp}"
createNewTab(tabNumber)           // Creates empty tab with title "Untitled Tab {n}"
getDefaultTabsConfig()            // Returns array with single default tab
getNextTabNumber(tabs)            // Calculates next tab number for naming
```

---

#### NEW: `webapp_v2/components/dashboard/tabs/DeleteTabDialog.tsx`

**Purpose:** Confirmation dialog before deleting a tab

**What it renders:**
```
┌─────────────────────────────────────┐
│  Delete "Tab Name"?                 │
│                                     │
│  This tab contains charts that      │
│  will be permanently deleted.       │
│                                     │
│            [Cancel]  [Delete]       │
└─────────────────────────────────────┘
```

**Props:**
- `open: boolean` — Dialog visibility
- `tabTitle: string` — Name of tab being deleted
- `hasContent: boolean` — Shows warning if tab has charts
- `onConfirm: () => void` — Called when delete confirmed

---

#### NEW: `webapp_v2/components/dashboard/tabs/TabBar.tsx`

**Purpose:** Horizontal tab bar with all tab operations

**What it renders:**
```
┌──────────────────────────────────────────────────────────┐
│ [Tab 1 ✕] [Tab 2 ✕] [Tab 3 ✕] [+]                        │
│  ▔▔▔▔▔▔                                                  │
│  (active)                                                │
└──────────────────────────────────────────────────────────┘
```

**Props:**
- `tabs: DashboardTab[]` — Array of tabs
- `activeTabId: string` — Currently active tab
- `isEditMode: boolean` — Show/hide edit controls
- `isPreviewMode: boolean` — Hide bar if single tab
- `onTabChange(tabId)` — Switch tab
- `onTabAdd()` — Add new tab
- `onTabRemove(tabId)` — Remove tab
- `onTabRename(tabId, title)` — Rename tab

**Behaviors:**
- Click tab → Switch to that tab
- Double-click tab → Edit name (edit mode only)
- Click "+" → Add new tab (edit mode only)
- Click "✕" → Show delete dialog (edit mode only)
- Single tab → Hide delete button

---

#### NEW: `webapp_v2/components/dashboard/tabs/index.ts`

**Purpose:** Barrel export for clean imports

```typescript
export { TabBar } from './TabBar';
export { DeleteTabDialog } from './DeleteTabDialog';
export * from './tab-utils';
```

---

#### MODIFY: `webapp_v2/types/dashboard.ts`

**Purpose:** Add TypeScript interface for tabs

**Add this interface:**
```typescript
export interface DashboardTab {
  id: string;                           // "tab-1710901234567"
  title: string;                        // "Untitled Tab 1" (max 50 chars)
  layout_config: DashboardLayoutItem[]; // Grid positions
  components: Record<string, DashboardComponentConfig>; // Component configs
}
```

**Update Dashboard interface:**
```typescript
export interface Dashboard {
  // ... existing fields ...
  tabs: DashboardTab[];  // ADD this field
}
```

---

#### MODIFY: `webapp_v2/components/dashboard/dashboard-builder-v2.tsx`

**Purpose:** Add tab state management and render TabBar

**Changes to make:**

1. **Add imports:**
```typescript
import { TabBar } from './tabs';
import { createNewTab, getNextTabNumber, generateTabId } from './tabs/tab-utils';
```

2. **Add state:**
```typescript
const [tabs, setTabs] = useState<DashboardTab[]>([...]);
const [activeTabId, setActiveTabId] = useState<string>('...');
const activeTab = tabs.find(t => t.id === activeTabId) || tabs[0];
```

3. **Add handlers:**
```typescript
handleTabChange(tabId)   // Switch active tab
handleTabAdd()           // Add new tab, switch to it
handleTabRemove(tabId)   // Remove tab, switch if needed
handleTabRename(tabId, title)  // Update tab title
```

4. **Update render:**
- Add `<TabBar />` below header, above canvas
- Use `activeTab.layout_config` instead of `dashboard.layout_config`
- Use `activeTab.components` instead of `dashboard.components`

---

### Manual Testing Checklist

| Test | Expected |
|------|----------|
| Open dashboard builder | Tab bar shows "Untitled Tab 1" |
| Click "+" | New "Untitled Tab 2" added, switched to it |
| Click Tab 1 | Switches back to Tab 1 |
| Double-click tab name | Input field appears |
| Type new name + Enter | Tab renamed |
| Press Escape while editing | Edit cancelled |
| Click "x" on tab | Delete dialog appears |
| Confirm delete | Tab removed |
| Single tab remaining | No "x" button visible |
| isEditMode=false | No +, x, or rename available |

---

### M1 Progress Checklist

#### New Files Created
- [x] `webapp_v2/components/dashboard/tabs/tab-utils.ts`
- [x] `webapp_v2/components/dashboard/tabs/DeleteTabDialog.tsx`
- [x] `webapp_v2/components/dashboard/tabs/TabBar.tsx`
- [x] `webapp_v2/components/dashboard/tabs/__tests__/TabBar.test.tsx`
- [x] `webapp_v2/components/dashboard/tabs/__tests__/DeleteTabDialog.test.tsx`
- [x] `webapp_v2/components/dashboard/tabs/__tests__/tab-utils.test.ts`

#### Files Modified
- [x] `webapp_v2/types/dashboard.ts` — Added DashboardTab interface
- [x] `webapp_v2/components/dashboard/dashboard-builder-v2.tsx` — Integrated TabBar

#### Testing
- [x] Unit tests passing
- [x] Manual testing complete

**✅ M1 Complete**

---

## Milestone 2 — Backend Schema & API Updates

**Mar 21, 24 (Fri, Mon) | Goal:** Backend accepts and returns `tabs` field so frontend can persist data.

---

### What We're Building

| Feature | Description |
|---------|-------------|
| Tabs Field | New `tabs` JSONField on Dashboard model |
| API Updates | GET/PUT/POST endpoints handle tabs structure |
| Validation | Ensure at least 1 tab, unique IDs, title max 50 chars |
| Default Tab | New dashboards created with "Untitled Tab 1" |

### How It Works

1. Add new `tabs` JSONField to Dashboard model
2. Update API schemas to accept/return tabs
3. Update DashboardService to handle tabs in create/update
4. New dashboards automatically get default tab
5. Frontend can now save and load tabs via API

### Data Model

**Dashboard Model (after adding tabs)**

| Field | Type | Description |
|-------|------|-------------|
| tabs | JSONField | **NEW** — Array of tab objects |

**Tabs JSON Structure**

```json
[
  {
    "id": "tab-1710901234567",
    "title": "Untitled Tab 1",
    "layout_config": [...],
    "components": {...}
  }
]
```

### API Endpoints

| Method | Endpoint | Change |
|--------|----------|--------|
| GET | /api/dashboards/:id/ | Returns `tabs` array in response |
| PUT | /api/dashboards/:id/ | Accepts `tabs` in request body |
| POST | /api/dashboards/ | Creates with default tab |

### Validation Rules

| Rule | Validation |
|------|------------|
| Min tabs | At least 1 tab required |
| Tab ID | Required, unique within dashboard |
| Tab title | Required, max 50 characters |

### File Details

#### MODIFY: `DDP_backend/ddpui/models/dashboard.py`

**Add this field:**
```python
tabs = models.JSONField(
    default=list,
    help_text="Array of tab objects"
)
```

---

#### MODIFY: `DDP_backend/ddpui/schemas/dashboard_schema.py`

**Add tab schema:**
```python
class DashboardTabSchema(Schema):
    id: str
    title: str
    layout_config: List[dict] = []
    components: dict = {}
```

**Update DashboardUpdate:**
```python
tabs: Optional[List[DashboardTabSchema]] = None
```

**Update DashboardResponse:**
```python
tabs: List[DashboardTabSchema] = []
```

---

#### MODIFY: `DDP_backend/ddpui/services/dashboard_service.py`

**Update create_dashboard:**
- Initialize with default tab: `[{id, title: "Untitled Tab 1", layout_config: [], components: {}}]`

**Update update_dashboard:**
- Validate tabs structure
- Save tabs to database

---

#### CREATE: `DDP_backend/ddpui/migrations/XXXX_add_dashboard_tabs.py`

**Schema migration to add tabs field:**
```python
migrations.AddField(
    model_name='dashboard',
    name='tabs',
    field=models.JSONField(default=list),
)
```

---

### M2 Progress Checklist

#### Backend Files Modified
- [x] `DDP_backend/ddpui/models/dashboard.py` — Add tabs JSONField
- [x] `DDP_backend/ddpui/schemas/dashboard_schema.py` — Add tab schemas
- [x] `DDP_backend/ddpui/services/dashboard_service.py` — Handle tabs CRUD
- [x] `DDP_backend/ddpui/api/dashboard_native_api.py` — Update duplicate endpoint

#### Schema Migration
- [x] `DDP_backend/ddpui/migrations/0153_add_dashboard_tabs.py` — Add tabs field to Dashboard model

#### Testing
- [x] Manual API testing (Swagger UI)

**✅ M2 Complete**

---

## Milestone 3 — Add Charts Inside Tabs

**Mar 25-27 (Tue-Thu) | Goal:** Charts are added inside the active tab and persisted to backend.

---

### What We're Building

| Feature | Description |
|---------|-------------|
| Chart Addition Flow | User selects tab → clicks "Add Chart" → chart appears in active tab |
| Per-Tab Layout | Each tab has its own grid layout and components |
| Save/Load | Tabs persist to backend on save, restore on load |
| Undo/Redo | Works with tab state |

### How It Works

When a user clicks "Add Chart", the system checks which tab is currently active and adds the chart to that tab's layout. Each tab maintains its own `layout_config` and `components` structure. On save, all tabs are sent to backend.

### Data Flow

| Step | Action |
|------|--------|
| 1 | User selects a tab (clicks on it) |
| 2 | User clicks "Add Chart" button |
| 3 | Chart selector modal opens |
| 4 | User selects a chart |
| 5 | Chart is added to active tab's `layout_config` and `components` |
| 6 | Tab content re-renders with new chart |
| 7 | On save, `tabs` array sent to backend |

### File Details

#### MODIFY: `webapp_v2/components/dashboard/dashboard-builder-v2.tsx`

**Update handleAddChart:**
- Add chart to `activeTab.layout_config` and `activeTab.components`

**Update handleAddText:**
- Add text to `activeTab.layout_config` and `activeTab.components`

**Update handleRemoveComponent:**
- Remove from active tab only

**Update handleSave:**
- Include `tabs` in API payload

**Update load logic:**
- Initialize tabs from API response

---

### Deliverables

- "Add Chart" adds to active tab
- "Add Text" adds to active tab
- Remove component from active tab
- Save persists tabs to backend
- Load restores tabs from backend
- Undo/redo works with tabs

---

### M3 Progress Checklist

#### Frontend Files Modified
- [x] `webapp_v2/components/dashboard/dashboard-builder-v2.tsx` — Tab handlers, state sync, save/load
- [x] `webapp_v2/components/dashboard/dashboard-native-view.tsx` — View mode uses active tab data
- [x] `webapp_v2/components/dashboard/tabs/tab-utils.ts` — Added `initializeTabsData`, `getActiveTabData`
- [x] `webapp_v2/components/dashboard/tabs/TabBar.tsx` — Single tab rename styling fix

#### Testing
- [x] Manual testing (add charts, switch tabs, save/reload, delete tabs)

**✅ M3 Complete**

---

## Milestone 4 — Migration, Preview Mode & Testing

**Mar 28, 31, Apr 1 (Fri, Mon, Tue) | Goal:** Migrate existing dashboards, preview mode, full testing.

---

### What We're Building

| Feature | Description |
|---------|-------------|
| Migration Script | Move existing dashboard data into tabs structure |
| Clear Root Level | Set root `layout_config` and `components` to empty |
| Preview Mode | Hide tab bar for single tab, show for multiple tabs |
| View Mode | Tab switching without edit controls |

### How It Works

**Migration:**
1. Run Django migration script
2. For each existing dashboard, copy `layout_config` and `components` into first tab
3. Clear root level fields (set to empty `[]` and `{}`)
4. All dashboards now use `tabs` for data

**Preview Mode:**
- Single tab → Tab bar hidden, looks like normal dashboard
- Multiple tabs → Tab bar visible, can click to switch, no edit controls

### Migration Script

```python
def migrate_dashboards_to_tabs(apps, schema_editor):
    Dashboard = apps.get_model('ddpui', 'Dashboard')

    for dashboard in Dashboard.objects.all():
        if not dashboard.tabs:
            dashboard.tabs = [{
                'id': f'tab-{int(time.time() * 1000)}',
                'title': 'Untitled Tab 1',
                'layout_config': dashboard.layout_config or [],
                'components': dashboard.components or {}
            }]
            dashboard.layout_config = []
            dashboard.components = {}
            dashboard.save()
```

### Preview Mode Behavior

| Tabs Count | Tab Bar | Edit Controls | User Experience |
|------------|---------|---------------|-----------------|
| 1 tab | Hidden | Hidden | Normal dashboard view |
| 2+ tabs | Visible | Hidden | Tab switching only |

### File Details

#### CREATE: `DDP_backend/ddpui/migrations/XXXX_migrate_dashboards_to_tabs.py`

**Data migration script to convert existing dashboards**

---

#### MODIFY: `webapp_v2/components/dashboard/dashboard-native-view.tsx`

**Add tab support to view mode:**
- Add tab state
- Render TabBar with `isPreviewMode={true}`
- Use active tab's layout and components

---

#### MODIFY: `webapp_v2/components/dashboard/tabs/TabBar.tsx`

**Preview mode logic (already built in M1):**
- `isPreviewMode && tabs.length === 1` → return null (hide bar)
- `isPreviewMode && tabs.length > 1` → show tabs, hide edit controls

---

### M4 Progress Checklist

#### Preview Mode ✅
- [x] Update `dashboard-native-view.tsx` for tabs
- [x] Use active tab's layout and components via `getActiveTabData`
- [x] Single tab → hide tab bar (`shouldShowTabs = tabs.length >= 2`)
- [x] Multiple tabs → show tab bar (no edit controls, `isEditMode={false}`)

#### Data Migration (Remaining)
- [ ] `DDP_backend/ddpui/migrations/XXXX_migrate_dashboards_to_tabs.py` — Data migration
- [ ] Test migration on staging
- [ ] Deploy migration to production

#### Testing (Remaining)
- [ ] E2E tests
- [ ] Manual testing all flows
- [ ] Bug fixes

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
| Mar 20 (Thu) | M1 – Complete | Tab Bar UI, add/remove/rename tabs, tab switching (frontend only) |
| Mar 24 (Mon) | M2 – Complete | Backend `tabs` field, API accepts/returns tabs |
| Mar 27 (Thu) | M3 – Complete | Charts added inside tabs, save/load tabs, undo/redo |
| Apr 1 (Tue) | M4 – Complete | Migration script, preview mode, full testing |

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
