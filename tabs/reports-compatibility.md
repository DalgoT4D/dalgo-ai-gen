# DALGO REPORTS — TABS COMPATIBILITY
## Technical Specification
**Version 1.1 • May 2026**

---

## Progress Status

> **Instructions for Claude:** Read this section first to understand current progress. Continue from the current milestone.

| Milestone | Status | Notes |
|-----------|--------|-------|
| M1 — Backend Fixes | 🟢 Complete | Schema + service fixes |
| M2 — Backend Tests | 🟢 Complete | Unit tests for tab-based dashboards |
| M3 — Frontend Fixes | 🟢 Complete | Summary above tabs, print layout, public view |
| M4 — Data Migration | 🟢 Complete | Management command for old snapshots |

**Current Focus:** Complete ✅

**Last Updated:** May 6, 2026

**Status Legend:** 🔴 Not Started | 🟡 In Progress | 🟢 Complete

---

## Background

The Reports feature creates snapshots of dashboards by freezing their layout,
components, and chart configurations. The dashboard tabs feature (migration 0160)
restructured how dashboards store their content — all data now lives inside
`tabs[]` and root-level `layout_config` and `components` are always empty `[]`
and `{}`.

Neither the backend report service nor the schemas were updated to handle this
new structure, causing reports from tab-based dashboards to render blank.

**Linear Ticket:** DALGO-1184

**Branch:** `feature/dashboard_tabs` (backend + frontend combined)

---

## Data Structure Reference

**Old Dashboard Structure (Root Level):**
```
Dashboard:
├── layout_config: [{i: "chart-1", x: 0, y: 0, w: 6, h: 4}]  ← Data here
├── components: {chart-1: {...}, text-1: {...}}                ← Data here
└── tabs: []
```

**New Dashboard Structure (Tab-based):**
```
Dashboard:
├── layout_config: []                                          ← EMPTY
├── components: {}                                             ← EMPTY
└── tabs:                                                      ← Data here
    [
      {
        id: "tab-1",
        title: "Overview",
        layout_config: [{i: "chart-1", ...}],
        components: {chart-1: {...}}
      }
    ]
```

---

## Milestone 1 — Backend Fixes

**Goal:** Reports correctly freeze and serve tab-based dashboard data.

---

### Issues & Fixes

#### FIX 1: Add `tabs` to `FrozenDashboardConfig` schema (CRITICAL)

**File:** `DDP_backend/ddpui/schemas/report_schema.py`

**Problem:** `FrozenDashboardConfig` schema is missing the `tabs` field. When
`_freeze_dashboard()` returns tabs data, Pydantic strips it during
`.model_dump()` because it's not declared in the schema.

**Current Code:**
```python
class FrozenDashboardConfig(Schema):
    layout_config: Optional[Any] = None
    components: Optional[Dict[str, Any]] = None
    # ❌ MISSING: tabs field
```

**Fix:**
```python
class FrozenDashboardConfig(Schema):
    layout_config: Optional[Any] = None
    components: Optional[Dict[str, Any]] = None
    tabs: Optional[List[Dict[str, Any]]] = None  # ✅ ADD THIS
```

---

#### FIX 2: Add `tabs` to `_freeze_dashboard()` (CRITICAL)

**File:** `DDP_backend/ddpui/core/reports/report_service.py`

**Problem:** `_freeze_dashboard()` captures `layout_config` and `components`
but not `tabs`. For tab-based dashboards both are empty, so the snapshot
stores no content.

**Current Code:**
```python
return {
    "layout_config": dashboard.layout_config,  # Empty for tab-based!
    "components": dashboard.components,         # Empty for tab-based!
    # ❌ MISSING: "tabs": dashboard.tabs
}
```

**Fix:**
```python
return {
    "layout_config": dashboard.layout_config,
    "components": dashboard.components,
    "tabs": dashboard.tabs,  # ✅ ADD THIS
    # ... other fields
}
```

---

#### FIX 3: Create `_extract_chart_ids()` helper (CRITICAL)

**File:** `DDP_backend/ddpui/core/reports/report_service.py`

**Problem:** Both `_freeze_chart_configs()` and `discover_datetime_columns()`
only scan `dashboard.components` which is always `{}` for tab-based dashboards.
Charts inside tabs are never discovered.

**Fix — Create shared helper:**
```python
@staticmethod
def _extract_chart_ids(dashboard: Dashboard) -> List[int]:
    """Extract chart IDs from tabs (new) and root components (backward compat)."""
    chart_ids = []

    for tab in (dashboard.tabs or []):
        for component in (tab.get("components") or {}).values():
            if component.get("type") == "chart":
                chart_id = component.get("config", {}).get("chartId")
                if chart_id:
                    chart_ids.append(chart_id)

    for component in (dashboard.components or {}).values():
        if component.get("type") == "chart":
            chart_id = component.get("config", {}).get("chartId")
            if chart_id:
                chart_ids.append(chart_id)

    return list(set(chart_ids))  # Deduplicate
```

---

#### FIX 4: Update `_freeze_chart_configs()` to use helper (CRITICAL)

**File:** `DDP_backend/ddpui/core/reports/report_service.py`

**Problem:** Only scans `dashboard.components` (root level). Misses all charts
inside tabs.

**Fix:** Replace root-level components loop with `_extract_chart_ids()`.

---

#### FIX 5: Update `discover_datetime_columns()` to use helper (CRITICAL)

**File:** `DDP_backend/ddpui/core/reports/report_service.py`

**Problem:** Only scans `dashboard.components` (root level) to build chart_ids
list. Returns empty result for tab-based dashboards.

**Fix:** Replace root-level components loop with `_extract_chart_ids()`.

---

### M1 Progress Checklist

#### Backend Files Modified
- [x] `DDP_backend/ddpui/schemas/report_schema.py` — Add `tabs` to `FrozenDashboardConfig`
- [x] `DDP_backend/ddpui/core/reports/report_service.py` — Add `tabs` to `_freeze_dashboard()`
- [x] `DDP_backend/ddpui/core/reports/report_service.py` — Create `_extract_chart_ids()` helper
- [x] `DDP_backend/ddpui/core/reports/report_service.py` — Update `_freeze_chart_configs()`
- [x] `DDP_backend/ddpui/core/reports/report_service.py` — Update `discover_datetime_columns()`

---

## Milestone 2 — Backend Tests

**Goal:** Unit tests covering tab-based dashboard scenarios in report service.

---

### Test Scenarios

**Scenario 1: Create Report from Tab-based Dashboard**
- Given: Dashboard with 2 tabs, each containing charts
- When: User creates a report
- Then: All tabs captured in `frozen_dashboard.tabs`, all charts from all tabs in `frozen_chart_configs`, datetime columns from charts in tabs are discovered

**Scenario 2: View Report from Tab-based Dashboard**
- Given: Report created from tab-based dashboard
- When: User views the report
- Then: Tabs visible in tab bar, user can switch tabs, charts render with frozen data

**Scenario 3: Backward Compatibility**
- Given: Old dashboard with root-level layout/components (no tabs)
- When: User creates and views a report
- Then: Report works as before, no regression

---

### Files

- [x] `DDP_backend/ddpui/tests/core/reports/test_report_service.py` — Add tab-based dashboard test cases

---

## Milestone 3 — Frontend Fixes

**Goal:** Report viewer correctly renders tab-based snapshots with summary above tabs.

---

### Issues & Fixes

#### FIX 6: Executive Summary positioned above tabs (CRITICAL)

**Files:**
- `webapp_v2/app/reports/[snapshotId]/page.tsx`
- `webapp_v2/app/share/report/[token]/PublicReportView.tsx`

**Problem:** Executive Summary was rendered inside the canvas (`beforeContent`), placing it below the TabBar. Team decision (Pradeep + Noopur): one summary per report, shown above all tabs.

**Fix:** Moved summary block out of `DashboardNativeView`'s `beforeContent` prop and rendered it directly in the page between the fixed header and `DashboardNativeView`. TabBar (inside `DashboardNativeView`) now appears below the summary.

---

#### FIX 7: Print layout (PDF) supports tabs (CRITICAL)

**File:** `webapp_v2/components/reports/print-layout.tsx`

**Problem:** `PrintLayout` only rendered root-level `layout_config`/`components`. For tab-based dashboards both are empty, so the PDF was blank.

**Fix:** Added tabs-aware rendering — iterates all tabs and renders their charts flat (no tab headings in PDF). Legacy root-level path kept for backward compatibility.

---

### M3 Progress Checklist

- [x] `app/reports/[snapshotId]/page.tsx` — Summary moved above TabBar
- [x] `app/share/report/[token]/PublicReportView.tsx` — Summary above TabBar in public view
- [x] `components/reports/print-layout.tsx` — Tabs support in PDF print layout

---

## Milestone 4 — Data Migration

**Goal:** Convert all existing old-format report snapshots to tabs format so frontend only deals with one structure.

---

### Decision

Old snapshots store data at root level (`layout_config`, `components`). New snapshots store data inside `tabs[]`. Rather than write a Django schema migration (which runs automatically), a management command was written so it can be run explicitly per environment before deployment.

---

### Files

- [x] `DDP_backend/ddpui/management/commands/migrate_report_snapshots_to_tabs.py`

### Usage

```bash
# Preview changes (no DB writes)
uv run python manage.py migrate_report_snapshots_to_tabs --dry-run

# Apply migration
uv run python manage.py migrate_report_snapshots_to_tabs
```

**Deploy order:** Run this command on each environment **before** deploying the new frontend code.

---

## Acceptance Criteria

### Backend
- [x] `tabs` field added to `_freeze_dashboard()` output
- [x] `tabs` field added to `FrozenDashboardConfig` schema
- [x] `_extract_chart_ids()` helper created and used in both methods
- [x] `_freeze_chart_configs()` extracts charts from tabs
- [x] `discover_datetime_columns()` discovers columns from charts in tabs
- [x] Unit tests added for tab-based dashboard scenarios
- [x] Existing root-level dashboard tests still pass

### Frontend
- [x] Executive summary shown above tabs (not inside tabs)
- [x] PDF print layout renders charts from all tabs
- [x] Public share view shows summary above tabs

### Data Migration
- [x] Management command migrates old root-level snapshots to tabs format
- [x] Dry-run mode available for safe preview
- [x] Empty snapshots safely skipped

### End-to-End
- [ ] Can create report from tab-based dashboard
- [ ] Report shows all tabs correctly
- [ ] All charts from all tabs render with frozen data
- [ ] Datetime column discovery works for charts in tabs
- [ ] PDF download includes all tabs content
- [ ] Old root-level dashboards still work (backward compatible)

---

**Dalgo Engineering Team • May 2026**
