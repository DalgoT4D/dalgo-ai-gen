

## Progress Status

> **Instructions for Claude:** Read this section first to understand current progress. Continue from the current milestone.

| Milestone | Status | Notes |
|-----------|--------|-------|
| M1 — Backend Model & Migration | 🟢 Complete | Fields removed from model + to_json(); schema migration 0161 created |
| M2 — Backend Schemas | 🟢 Complete | Removed root fields from DashboardResponse and FrozenDashboardConfig |
| M3 — Backend Services & API | 🟢 Complete | Updated report_service, dashboard_service, chart_service, duplicate API |
| M4 — Backend Tests | 🟢 Complete | Updated fixtures and assertions across all test files |
| M5 — Delete Obsolete Commands | 🟢 Complete | migrate_dashboards_to_tabs.py and migrate_report_snapshots_to_tabs.py deleted; cleanup_frozen_dashboard_root_fields.py created |
| M6 — Frontend Interface & Components | 🟢 Complete | Removed layout_config/components from Dashboard interface; simplified DashboardMiniPreview to tabs-only |

**Current Focus:** All milestones complete

**Last Updated:** 2026-05-19

**Status Legend:** 🔴 Not Started | 🟡 In Progress | 🟢 Complete

---

## 1. Overview

Remove the deprecated root-level `layout_config` and `components` fields from the Dashboard model, schemas, services, and frontend code. These fields were kept after the dashboard tabs migration for rollback safety. Data migrations have run on staging and production — this cleanup removes the legacy fields permanently.


## 2. Blast Radius

| Surface | Hop Distance | Why Affected | Status | Notes |
|---------|--------------|--------------|--------|-------|
| Dashboard (model) | 0 | Primary entity — removing fields from model | **in-scope** | Fields removed, migration generated |
| Dashboard (API response) | 0 | Schema changes drop fields from response | **in-scope** | Frontend must read from `tabs` |
| Dashboard Builder | 1 | Saves to/reads from root fields currently | **in-scope** | Must use tabs structure |
| Dashboard View | 1 | Renders from root fields currently | **in-scope** | Must read from active tab |
| ReportSnapshot | 1 | `_freeze_dashboard()` captures root fields | **in-scope** | Remove root fields from freeze; tabs already captured |
| Report Print Layout | 2 | Reads frozen dashboard config | **in-scope** | Already reads from tabs (verify) |
| Dashboard Duplicate | 1 | Copies root fields + filter remapping | **in-scope** | Remove root-level copy logic |
| Chart Service | 1 | Iterates `dashboard.components` to find chart usage | **in-scope** | Rewrite to iterate tabs |
| DashboardMiniPreview | 1 | Has backward-compat fallback logic | **in-scope** | Simplify to tabs-only |
| Share Link (live) | 2 | Inherits from Dashboard view | **auto-inherited** | No direct changes needed |
| Share Link (report) | 2 | Inherits from ReportSnapshot render | **auto-inherited** | No direct changes needed |
| Migration Commands | — | Will fail after column removal | **in-scope** | Delete obsolete commands |

**Unaffected surfaces:**
- **Chart** — Chart model unchanged; only Dashboard's reference to charts changes
- **Metric/KPI** — Not yet implemented; no impact
- **Alert/Notification** — No dependency on dashboard structure
- **Pipeline/Transform/Source** — Data layer; no dashboard dependency

---

## 3. High-Level Design (HLD)

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        BEFORE (Current)                         │
├─────────────────────────────────────────────────────────────────┤
│  Dashboard Model                                                 │
│  ├── layout_config: JSONField (deprecated, kept for rollback)   │
│  ├── components: JSONField (deprecated, kept for rollback)      │
│  └── tabs: JSONField [{id, title, layout_config, components}]   │
│                                                                  │
│  Frontend reads: dashboard.layout_config, dashboard.components   │
│  Backend services: iterate dashboard.components                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        AFTER (Target)                            │
├─────────────────────────────────────────────────────────────────┤
│  Dashboard Model                                                 │
│  └── tabs: JSONField [{id, title, layout_config, components}]   │
│                                                                  │
│  Frontend reads: dashboard.tabs[activeTabIndex].layout_config    │
│  Backend services: iterate dashboard.tabs[].components           │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow

1. **Dashboard Create/Edit:**
   - Frontend sends: `{ tabs: [{ id, title, layout_config, components }] }`
   - Backend stores in `dashboard.tabs` JSONField
   - No root-level fields exist

2. **Dashboard View:**
   - API returns: `{ ..., tabs: [{ id, title, layout_config, components }] }`
   - Frontend renders active tab's layout and components

3. **Report Snapshot Creation:**
   - `_freeze_dashboard()` captures `tabs` array only
   - `_extract_chart_ids()` iterates tabs to find charts

4. **Dashboard Duplication:**
   - `copy_tabs_with_filter_remapping()` handles tab-level filter ID remapping
   - No root-level copy needed

### Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| Remove DB columns (not just code) | Clean schema; prevents accidental writes to deprecated fields |
| Delete migration commands | They reference `dashboard.layout_config` which will error after column drop |
| Deploy backend + frontend together | API response change requires frontend to read from tabs |
| Keep frozen snapshot JSON keys | Old snapshots may have `layout_config`/`components` in JSON blob; Pydantic ignores unknown keys |

---

## 4. Low-Level Design (LLD)

### 4.1 Data Model Changes

**File:** `ddpui/models/dashboard.py`

```python
# REMOVE these fields (lines 68 and 71):
# layout_config = models.JSONField(default=list, help_text="Grid layout positions and sizes")
# components = models.JSONField(default=dict, help_text="Dashboard components configuration")

# KEEP (lines 74-76):
tabs = models.JSONField(
    default=list, help_text="Array of tab objects: [{id, title, layout_config, components}]"
)
```

**Migration:**
```bash
python manage.py makemigrations ddpui --name remove_dashboard_root_layout_components
```

Generates:
```python
operations = [
    migrations.RemoveField(model_name='dashboard', name='layout_config'),
    migrations.RemoveField(model_name='dashboard', name='components'),
]
```

**Update `to_json()` method** (lines 131-152):
```python
# REMOVE lines 140-141:
# "layout_config": self.layout_config,
# "components": self.components,
```

### 4.2 Schema Changes

**File:** `ddpui/schemas/dashboard_schema.py`

```python
# DashboardResponse (lines 66-88) - REMOVE these fields (lines 76-77):
# layout_config: list[dict]
# components: dict

# KEEP (line 78):
tabs: List[DashboardTabSchema] = []
```

**File:** `ddpui/schemas/report_schema.py`

```python
# FrozenDashboardConfig (lines 22-34) - REMOVE these fields (lines 30-31):
# layout_config: Optional[Any] = None
# components: Optional[Dict[str, Any]] = None

# KEEP (line 32):
tabs: Optional[List[Dict[str, Any]]] = None
```

### 4.3 Service Changes

**File:** `ddpui/core/reports/report_service.py`

`_freeze_dashboard()` (lines 49-63) — Remove root fields from return dict:
```python
@staticmethod
def _freeze_dashboard(dashboard: Dashboard) -> Dict[str, Any]:
    """Freeze dashboard layout, structure & filters into one dict."""
    filters = dashboard.filters.all().order_by("order")
    return {
        "dashboard_id": dashboard.id,
        "title": dashboard.title,
        "description": dashboard.description,
        "grid_columns": dashboard.grid_columns,
        "target_screen_size": dashboard.target_screen_size,
        # REMOVE: "layout_config": dashboard.layout_config,
        # REMOVE: "components": dashboard.components,
        "tabs": dashboard.tabs,  # Only tabs, no root-level fields
        "filter_layout": dashboard.filter_layout,
        "filters": [f.to_json() for f in filters],
    }
```

`_extract_chart_ids()` (lines 66-83) — Remove backward-compat fallback:
```python
@staticmethod
def _extract_chart_ids(dashboard: Dashboard) -> List[int]:
    """Extract chart IDs from tabs only (remove backward compat)."""
    chart_ids = []

    for tab in dashboard.tabs or []:
        for component in (tab.get("components") or {}).values():
            if component.get("type") == "chart":
                chart_id = component.get("config", {}).get("chartId")
                if chart_id:
                    chart_ids.append(chart_id)

    # REMOVE lines 77-81 (fallback block that reads from dashboard.components):
    # for component in (dashboard.components or {}).values():
    #     if component.get("type") == "chart":
    #         chart_id = component.get("config", {}).get("chartId")
    #         if chart_id:
    #             chart_ids.append(chart_id)

    return list(set(chart_ids))
```

**File:** `ddpui/services/dashboard_service.py`

`apply_filters()` (lines 761-804) — Rewrite to iterate tabs:
```python
@staticmethod
def apply_filters(dashboard_id: int, filters: Dict[str, Any], orguser) -> Dict[str, Any]:
    # ... existing setup code ...

    # CHANGE line 783 from:
    # components = dashboard.components
    # TO iterate through tabs:
    chart_results = {}
    for tab in dashboard.tabs or []:
        components = tab.get("components") or {}
        for component_id, component_config in components.items():
            if component_config.get("type") == DashboardComponentType.CHART.value:
                chart_id = component_config.get("config", {}).get("chartId")
                if chart_id:
                    # ... existing chart processing logic ...
    return chart_results
```

`get_dashboard_charts()` (lines 989-1026) — Rewrite to iterate tabs:
```python
@staticmethod
def get_dashboard_charts(dashboard_id: int, orguser) -> List[Dict[str, Any]]:
    # ... existing setup code ...

    # CHANGE line 1004 from:
    # components = dashboard.components
    # TO iterate through tabs:
    chart_ids = []
    for tab in dashboard.tabs or []:
        components = tab.get("components") or {}
        for component_id, component_config in components.items():
            if component_config.get("type") == DashboardComponentType.CHART.value:
                chart_id = component_config.get("config", {}).get("chartId")
                if chart_id:
                    chart_ids.append(chart_id)

    # ... rest of function unchanged ...
```

**File:** `ddpui/services/chart_service.py`

`get_chart_dashboards()` (lines 308-341) — Rewrite to iterate tabs:
```python
@staticmethod
def get_chart_dashboards(chart_id: int, org: Org) -> List[Dict[str, Any]]:
    # Verify chart exists
    ChartService.get_chart(chart_id, org)

    # Find dashboards that have this chart in their tabs
    dashboards_with_chart = []
    dashboards = Dashboard.objects.filter(org=org)

    for dashboard in dashboards:
        # CHANGE from checking dashboard.components to checking tabs:
        for tab in dashboard.tabs or []:
            components = tab.get("components") or {}
            for component_id, component in components.items():
                if (
                    component.get("type") == DashboardComponentType.CHART.value
                    and component.get("config", {}).get("chartId") == chart_id
                ):
                    dashboards_with_chart.append(
                        {
                            "id": dashboard.id,
                            "title": dashboard.title,
                            "dashboard_type": dashboard.dashboard_type,
                            "tab_id": tab.get("id"),
                            "tab_title": tab.get("title"),
                        }
                    )
                    break  # Found chart in this tab

    return dashboards_with_chart
```

**File:** `ddpui/api/dashboard_native_api.py`

`duplicate_dashboard()` (lines 163-257) — Remove root-level handling:
```python
@dashboard_native_router.post("/{dashboard_id}/duplicate/", response=DashboardResponse)
@has_permission(["can_create_dashboards"])
def duplicate_dashboard(request, dashboard_id: int):
    # ... existing setup code ...

    with transaction.atomic():
        # CHANGE lines 184-185 - remove layout_config and components:
        new_dashboard = Dashboard.objects.create(
            title=f"Copy of {original_dashboard.title}",
            description=original_dashboard.description,
            dashboard_type=original_dashboard.dashboard_type,
            grid_columns=original_dashboard.grid_columns,
            target_screen_size=original_dashboard.target_screen_size,
            # REMOVE: layout_config=[],
            # REMOVE: components={},
            created_by=orguser,
            org=orguser.org,
            last_modified_by=orguser,
        )

        # ... filter copying logic unchanged ...

        # REMOVE entire block (lines 210-247) that handles root-level remapping:
        # new_layout_config = copy.deepcopy(original_dashboard.layout_config or [])
        # new_components = copy.deepcopy(original_dashboard.components or {})
        # ... filter remapping on root-level fields ...

        # KEEP only tabs handling (lines 248-250):
        new_dashboard.tabs = DashboardService.copy_tabs_with_filter_remapping(
            original_dashboard.tabs or [], filter_id_mapping
        )
        # REMOVE: new_dashboard.layout_config = new_layout_config
        # REMOVE: new_dashboard.components = updated_components
        new_dashboard.save()
```

### 4.4 Frontend Changes

**File:** `webapp_v2/hooks/api/useDashboards.ts`

```typescript
// ADD DashboardTab interface (before Dashboard interface):
export interface DashboardTab {
  id: string;
  title: string;
  layout_config: any[];
  components: Record<string, any>;
}

export interface Dashboard {
  id: number;
  title: string;
  dashboard_type: 'native' | 'superset';
  grid_columns: number;
  target_screen_size?: 'desktop' | 'tablet' | 'mobile' | 'a4';
  filter_layout?: 'vertical' | 'horizontal';
  // REMOVE line 11: layout_config: any;
  responsive_layouts?: any;
  // REMOVE line 13: components: any;
  // ADD:
  tabs: DashboardTab[];
  is_published: boolean;
  // ... rest unchanged ...
}
```

**File:** `webapp_v2/components/dashboard/dashboard-builder-v2.tsx`

Initialization (lines 282-284):
```typescript
// CHANGE FROM:
// let initialLayout = Array.isArray(initialData?.layout_config) ? initialData.layout_config : [];
// const initialComponents = initialData?.components || {};

// TO:
const activeTab = initialData?.tabs?.[0];
let initialLayout = Array.isArray(activeTab?.layout_config) ? activeTab.layout_config : [];
const initialComponents = activeTab?.components || {};
```

Save function (lines 842-843):
```typescript
// CHANGE FROM:
// layout_config: JSON.parse(JSON.stringify(state.layout)),
// components: JSON.parse(JSON.stringify(state.components)),

// TO:
tabs: [{
  id: currentTabId || 'default',
  title: currentTabTitle || 'Default',
  layout_config: JSON.parse(JSON.stringify(state.layout)),
  components: JSON.parse(JSON.stringify(state.components)),
}],
```

**File:** `webapp_v2/components/dashboard/dashboard-native-view.tsx`

```typescript
// CHANGE line 534 FROM:
// if (!dashboard?.components) return null;
// TO:
const activeTab = dashboard?.tabs?.[activeTabIndex || 0];
if (!activeTab?.components) return null;

// CHANGE line 536 FROM:
// const component = dashboard.components[componentId];
// TO:
const component = activeTab?.components?.[componentId];

// CHANGE lines 1133 and 1145 FROM:
// dashboard?.layout_config
// dashboard.layout_config || []
// TO:
// activeTab?.layout_config
// activeTab?.layout_config || []
```

**File:** `webapp_v2/components/dashboard/DashboardMiniPreview.tsx`

Simplify to tabs-only (replace lines 19-29):
```typescript
export function DashboardMiniPreview({ dashboardData, className }: DashboardMiniPreviewProps) {
  // REMOVE all backward-compat fallback logic (lines 19-29)
  // SIMPLIFY TO:
  const activeTab = dashboardData?.tabs?.[0];
  const layoutConfig = activeTab?.layout_config || [];
  const components = activeTab?.components
    ? Object.values(activeTab.components)
    : [];

  // Use layoutConfig for grid calculations
  // ... rest of component unchanged ...
}
```

**File:** `webapp_v2/components/reports/print-layout.tsx`

Verify reads from tab structure (should already be correct from tabs PR):
```typescript
// Should already be reading from:
// dashboardData.tabs[tabIndex].components
// dashboardData.tabs[tabIndex].layout_config
// If not, update accordingly
```

### 4.5 Files to Delete

| File | Reason |
|------|--------|
| `ddpui/management/commands/migrate_dashboards_to_tabs.py` | References `dashboard.layout_config` — will error after column drop |
| `ddpui/management/commands/migrate_report_snapshots_to_tabs.py` | References frozen `layout_config` — obsolete after migrations ran |

---

## 5. Security Review

| Area | Assessment | Action |
|------|------------|--------|
| Authentication & Authorization | No new endpoints; existing `@has_permission` decorators unchanged | None |
| Input Validation | `tabs` field validated by `DashboardTabSchema` Pydantic model | None |
| Data Access Control | Org-scoping unchanged; dashboards still filtered by `org_id` | None |
| Sensitive Data | No PII or credentials in layout/components | None |
| Injection Risks | JSON fields stored via Django ORM; no raw SQL | None |
| External Service Calls | None affected | None |
| Rate Limiting | No new endpoints | None |

**Risk:** None identified. This is a cleanup of existing validated structures.

---

## 6. Testing Strategy

### Unit Tests

| Area | Tests |
|------|-------|
| Dashboard model | `test_dashboard_create_success`, `test_dashboard_to_json` — remove assertions on root fields |
| Dashboard schema | `DashboardResponse` instantiation without root fields |
| Report service | `_freeze_dashboard` returns tabs only; `_extract_chart_ids` iterates tabs |
| Dashboard service | `get_dashboard_charts` iterates tabs |
| Chart service | `get_chart_dashboards` finds charts in tabs |

### Integration Tests

| Scenario | Validation |
|----------|------------|
| Create dashboard via API | Response has `tabs`, no root `layout_config`/`components` |
| Update dashboard via API | Tabs saved correctly |
| Duplicate dashboard | Tabs copied with filter remapping |
| Create report snapshot | Frozen config has tabs only |
| Render existing snapshot | Old snapshots with root fields in JSON still render (Pydantic ignores) |

### Edge Cases

| Case | Expected Behavior |
|------|-------------------|
| Dashboard with empty tabs array | Renders empty state |
| Old frozen snapshot with root fields in JSON | Pydantic ignores extra keys; renders from tabs |
| Dashboard with multiple tabs | All tabs accessible (future feature) |

### Test Files to Update

| File | Changes |
|------|---------|
| `ddpui/tests/models/test_dashboard_models.py` | Remove `layout_config=[]`, `components={}` from fixtures |
| `ddpui/tests/api_tests/test_dashboard_native_api.py` | Remove kwargs; update response assertions |
| `ddpui/tests/api_tests/test_report_api.py` | Remove kwargs from dashboard fixtures |
| `ddpui/tests/api_tests/test_report_permissions.py` | Keep frozen JSON keys (data, not model) |
| `ddpui/tests/api_tests/test_public_report_api.py` | Remove kwargs |
| `ddpui/tests/api_tests/test_charts_api.py` | Remove kwargs |
| `ddpui/tests/schema_tests/test_dashboard_schema.py` | Remove kwargs from schema tests |
| `ddpui/tests/services/test_chart_service.py` | Remove kwargs; update assertions |
| `ddpui/tests/services/test_dashboard_service.py` | Remove kwargs; update iteration tests |
| `ddpui/tests/core/reports/test_report_service.py` | Remove/rewrite root field assertions |

---

## 7. Milestones

### Milestone 1: Backend Model & Migration

**Deliverable:** Dashboard model without root-level fields; migration ready

**Services:** DDP_backend

**Key tasks:**
- [ ] Remove `layout_config` field from `Dashboard` model (line 68)
- [ ] Remove `components` field from `Dashboard` model (line 71)
- [ ] Remove fields from `to_json()` method (lines 140-141)
- [ ] Generate Django migration `remove_dashboard_root_layout_components`
- [ ] Review migration SQL for correctness

**Acceptance criteria:**
- `python manage.py makemigrations --check` passes
- Migration file has two `RemoveField` operations

---

### Milestone 2: Backend Schemas

**Deliverable:** API response schemas updated

**Services:** DDP_backend

**Key tasks:**
- [ ] Remove `layout_config` and `components` from `DashboardResponse` (lines 76-77)
- [ ] Remove fields from `FrozenDashboardConfig` (lines 30-31)
- [ ] Verify `DashboardTabSchema` unchanged

**Acceptance criteria:**
- Schema tests pass without root fields
- API response structure correct via manual test

---

### Milestone 3: Backend Services & API

**Deliverable:** All service methods iterate tabs instead of root fields

**Services:** DDP_backend

**Key tasks:**
- [ ] Update `_freeze_dashboard()` in report_service.py (lines 58-59)
- [ ] Update `_extract_chart_ids()` in report_service.py (lines 77-81)
- [ ] Rewrite `get_dashboard_charts()` in dashboard_service.py (line 1004)
- [ ] Rewrite `apply_filters()` in dashboard_service.py (line 783)
- [ ] Rewrite `get_chart_dashboards()` in chart_service.py (lines 326-339)
- [ ] Remove root-level handling from `duplicate_dashboard()` in dashboard_native_api.py (lines 184-185, 210-247)

**Acceptance criteria:**
- `grep -r "dashboard\.layout_config\|dashboard\.components" ddpui/` returns no hits (except tests)
- Service unit tests pass

---

### Milestone 4: Backend Tests

**Deliverable:** All tests pass without root fields

**Services:** DDP_backend

**Key tasks:**
- [ ] Update fixtures in all test files (see Section 6)
- [ ] Remove/rewrite assertions on root fields
- [ ] Run full test suite

**Acceptance criteria:**
- `uv run pytest ddpui/tests/` passes
- No warnings about unknown fields

---

### Milestone 5: Delete Obsolete Commands

**Deliverable:** Migration commands removed

**Services:** DDP_backend

**Key tasks:**
- [ ] Delete `ddpui/management/commands/migrate_dashboards_to_tabs.py`
- [ ] Delete `ddpui/management/commands/migrate_report_snapshots_to_tabs.py`

**Acceptance criteria:**
- Commands no longer exist
- `python manage.py help` doesn't list them

---

### Milestone 6: Frontend Interface & Components

**Deliverable:** Frontend reads/writes tabs structure only

**Services:** webapp_v2

**Key tasks:**
- [ ] Add `DashboardTab` interface and update `Dashboard` interface in useDashboards.ts (lines 11, 13)
- [ ] Update initialization in dashboard-builder-v2.tsx (lines 282-284)
- [ ] Update save function in dashboard-builder-v2.tsx (lines 842-843)
- [ ] Update rendering in dashboard-native-view.tsx (lines 534, 536, 1133, 1145)
- [ ] Simplify DashboardMiniPreview.tsx (lines 19-29)
- [ ] Verify print-layout.tsx reads from tabs

**Acceptance criteria:**
- `grep -r "dashboard\.layout_config\|dashboard\.components" --include="*.tsx" --include="*.ts" src/ components/ hooks/ app/` returns no hits
- Manual testing: create, edit, view, duplicate dashboard works

---


