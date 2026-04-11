# Plan: Refactor Report Chart Data to Use GET (like Dashboards)

## Problem

Currently, report charts use a different data-fetching pattern than dashboard charts:

- **Dashboard**: `GET /api/charts/{chartId}/data/` — frontend sends only the chart ID. Backend reads config from DB, builds payload, runs query.
- **Report**: `POST /api/v1/public/reports/{token}/chart-data/` — frontend unpacks the frozen config into a full `ChartDataPayload` and sends it back to the backend. The backend already has this config in `frozen_chart_configs`.

This means the frontend echoes data back that the backend already has. It also requires the frontend to do the `extra_config` → top-level field mapping (lines 356-465 in `chart-element-view.tsx`), which duplicates backend logic.

## Proposed Change

Make reports work like dashboards: frontend sends **chart ID**, backend reads config from `frozen_chart_configs` and builds the payload itself — using the **same payload construction pattern** already in `get_chart_data_by_id`.

## Branch

Both repos: `refactor/report-chart-data-get` (from `main`)

---

## How Dashboard Loads Charts (Current — Reference)

### Dashboard (authenticated, private)

| Chart Type | Method | Endpoint | Frontend Sends |
|---|---|---|---|
| Bar/Line/Pie/Number | **GET** | `/api/charts/{chartId}/data/` | Just chart ID in URL |
| Table | **POST** | `/api/charts/chart-data-preview/?page=X&limit=Y` | Full `ChartDataPayload` |
| Table rows | **POST** | `/api/charts/chart-data-preview/total-rows/` | Full `ChartDataPayload` |
| Map | **POST** | `/api/charts/map-data-overlay/` | `MapDataOverlayPayload` |

**Frontend code** (`chart-element-view.tsx`):
- Bar/Line/Pie/Number: Line 232-235 → `apiUrl = /api/charts/${chartId}/data/`, fetched via SWR GET (line 324)
- Table: Line 563 → `useChartDataPreview(chartDataPayload)` hook
- Table rows: Line 571 → `useChartDataPreviewTotalRows(chartDataPayload)` hook
- Map: Line 886 → `useMapDataOverlay(mapDataOverlayPayload)` hook

### Dashboard (public, shared via token)

| Chart Type | Method | Endpoint | Frontend Sends |
|---|---|---|---|
| Bar/Line/Pie/Number | **GET** | `/api/v1/public/dashboards/{token}/charts/{chartId}/data` | Just chart ID + token |
| Table | **POST** | `/api/v1/public/dashboards/{token}/charts/{chartId}/data-preview/` | Full `ChartDataPayload` |
| Table rows | **POST** | `/api/v1/public/dashboards/{token}/charts/{chartId}/data-preview/total-rows/` | Full `ChartDataPayload` |
| Map | **POST** | `/api/v1/public/dashboards/{token}/charts/{chartId}/map-data/` | `MapDataOverlayPayload` |

**Frontend code** (`chart-element-view.tsx`):
- Bar/Line/Pie/Number: Line 234 → `apiUrl = /api/v1/public/dashboards/${publicToken}/charts/${chartId}/data`, fetched via SWR GET (line 324)
- Table: Line 513 → `publicTableDataUrl` with POST fetch (line 543)
- Table rows: Line 581 → `publicTableTotalRowsUrl` with `apiPublicPost` (line 593)
- Map: Line 861 → `publicMapDataUrl` with `apiPublicPost` (line 874)

**Key insight**: Only bar/line/pie/number charts use GET on dashboards. Table and map charts still use POST even on dashboards.

---

## How Report Loads Charts (Current — What We're Changing)

### Report (public, shared via token)

| # | Chart Type | Method | Endpoint | Frontend Sends | Frontend Line |
|---|---|---|---|---|---|
| 1 | Bar/Line/Pie/Number | **POST** | `/api/v1/public/reports/{token}/chart-data/` | Full `ChartDataPayload` | 487 |
| 2 | Table | **POST** | `/api/v1/public/reports/{token}/chart-data-preview/` | Full `ChartDataPayload` | 512 |
| 3 | Table rows | **POST** | `/api/v1/public/reports/{token}/chart-data-preview/total-rows/` | Full `ChartDataPayload` | 580 |
| 4 | Map | **POST** | `/api/v1/public/reports/{token}/map-data/` | `MapDataOverlayPayload` | 860 |

**Backend**: All 4 endpoints are in `ddpui/api/public_api.py`

### Report (authenticated, viewed in the app)

| # | Chart Type | Method | Endpoint | Frontend Sends | Frontend Line |
|---|---|---|---|---|---|
| 5 | Bar/Line/Pie/Number | **POST** | `/api/charts/chart-data/` | Full `ChartDataPayload` | 497 |
| 6 | Table | **POST** | `/api/charts/chart-data-preview/` | Full `ChartDataPayload` | 563 (via `useChartDataPreview` hook) |
| 7 | Table rows | **POST** | `/api/charts/chart-data-preview/total-rows/` | Full `ChartDataPayload` | 571 (via `useChartDataPreviewTotalRows` hook) |
| 8 | Map | **POST** | `/api/charts/map-data-overlay/` | `MapDataOverlayPayload` | 886 (via `useMapDataOverlay` hook) |

**Backend**: Endpoints #5-7 are in `ddpui/api/charts_api.py`, #8 is also in `charts_api.py`

**Important**: Endpoints #5-8 are the **same generic chart endpoints** that dashboards also use for table/map. The difference is:
- Dashboard bar/line/pie/number → `GET /api/charts/{chartId}/data/` (backend builds payload from Chart model)
- Report bar/line/pie/number → `POST /api/charts/chart-data/` (frontend builds payload from `frozenChartConfig` and sends it)
- Both dashboard and report use the same POST endpoints for table and map

### Total: 8 POST API calls for reports (4 public + 4 authenticated)

All 8 send the full payload from the frontend, even though the backend already has everything in `frozen_chart_configs`.

---

## Proposed New Endpoints

### Public report endpoints (replace 4 POST → 4 GET)

| # | Current (POST) | New (GET) |
|---|---|---|
| 1 | `POST /reports/{token}/chart-data/` | `GET /reports/{token}/charts/{chart_id}/data/` |
| 2 | `POST /reports/{token}/chart-data-preview/` | `GET /reports/{token}/charts/{chart_id}/table-data/?page=0&limit=100` |
| 3 | `POST /reports/{token}/chart-data-preview/total-rows/` | `GET /reports/{token}/charts/{chart_id}/table-data/total-rows/` |
| 4 | `POST /reports/{token}/map-data/` | `GET /reports/{token}/charts/{chart_id}/map-data/` |

No body. The `chart_id` refers to a key in `frozen_chart_configs`.

### Authenticated report endpoints (new — 4 GET endpoints)

| # | Current (POST to generic chart endpoint) | New (GET) |
|---|---|---|
| 5 | `POST /api/charts/chart-data/` | `GET /api/reports/{snapshot_id}/charts/{chart_id}/data/` |
| 6 | `POST /api/charts/chart-data-preview/` | `GET /api/reports/{snapshot_id}/charts/{chart_id}/table-data/?page=0&limit=100` |
| 7 | `POST /api/charts/chart-data-preview/total-rows/` | `GET /api/reports/{snapshot_id}/charts/{chart_id}/table-data/total-rows/` |
| 8 | `POST /api/charts/map-data-overlay/` | `GET /api/reports/{snapshot_id}/charts/{chart_id}/map-data/` |

These are new endpoints in `report_api.py` (authenticated). Same logic as the public ones but use `snapshot_id` instead of `token`.

---

## Implementation Status

| Step | Status |
|------|--------|
| Backend: 4 public GET endpoints in `public_api.py` | TODO |
| Backend: 4 authenticated GET endpoints in `report_api.py` | TODO |
| Frontend: Switch report mode to GET in `chart-element-view.tsx` | TODO |
| Tests | TODO |

Previous attempt only changed 3 of 4 public endpoints (missed map-data) and none of the authenticated ones. That attempt has been reverted — both repos are clean on `refactor/report-chart-data-get` with no diff from `main`.

---

## Files to Change

### Backend (ddp_backend) — 2 files

#### 1. `ddpui/api/public_api.py` — Replace 4 POST with 4 GET endpoints

**Remove:**
- `POST /reports/{token}/chart-data/`
- `POST /reports/{token}/chart-data-preview/`
- `POST /reports/{token}/chart-data-preview/total-rows/`
- `POST /reports/{token}/map-data/`

**Add:**
- `GET /reports/{token}/charts/{chart_id}/data/`
- `GET /reports/{token}/charts/{chart_id}/table-data/?page=0&limit=100`
- `GET /reports/{token}/charts/{chart_id}/table-data/total-rows/`
- `GET /reports/{token}/charts/{chart_id}/map-data/`

**Two private helper functions (in `public_api.py` only, NOT in service layer):**

1. **`_get_report_chart_config(snapshot, chart_id)`** — Looks up `frozen_chart_configs[str(chart_id)]`, deep-copies it, and injects period date filters into `extra_config.filters`. Replicates the logic from `ReportService._inject_period_into_chart_configs` but for a single chart.

2. **`_build_report_chart_payload(config)`** — Builds a `ChartDataPayload` from the frozen config dict. Uses the same field-by-field mapping as `get_chart_data_by_id` in `charts_api.py` (lines 1150-1169):
   - `extra_config.x_axis_column` → `payload.x_axis`
   - `extra_config.dimension_column` → `payload.dimension_col`
   - `extra_config.extra_dimension_column` → `payload.extra_dimension`
   - `extra_config.dimension_columns` → `payload.dimensions`
   - `extra_config.metrics` → `payload.metrics`
   - etc.

**Each endpoint follows the same pattern:**
1. Look up snapshot via token (`_get_public_report_snapshot`)
2. Get org warehouse
3. `config = _get_report_chart_config(snapshot, chart_id)` — 404 if not found
4. `payload = _build_report_chart_payload(config)`
5. Run query via existing functions:
   - Chart data: `generate_chart_data_and_config(payload, org_warehouse)`
   - Table data: `charts_service.get_chart_data_table_preview(org_warehouse, payload, page, limit)`
   - Total rows: `charts_service.get_chart_data_total_rows(org_warehouse, payload)`
   - Map data: build `MapDataOverlayPayload` from config + run map query

#### 2. `ddpui/api/report_api.py` — Add 4 authenticated GET endpoints

Same logic as the public endpoints but:
- Uses `snapshot_id` (int) instead of `token` (string)
- Requires authentication (`@has_permission`)
- Looks up snapshot by ID instead of token

New endpoints:
- `GET /{snapshot_id}/charts/{chart_id}/data/`
- `GET /{snapshot_id}/charts/{chart_id}/table-data/?page=0&limit=100`
- `GET /{snapshot_id}/charts/{chart_id}/table-data/total-rows/`
- `GET /{snapshot_id}/charts/{chart_id}/map-data/`

Can reuse the same `_get_report_chart_config` and `_build_report_chart_payload` helpers (move to a shared location or import from `public_api.py`).

---

### Frontend (webapp_v2) — 1 file

#### `components/dashboard/chart-element-view.tsx` — Simplify report chart data fetching

**4 areas to change in report mode (`frozenChartConfig` is set):**

1. **Bar/Line/Pie/Number chart data** (lines ~469-500):
   - Public: `POST /reports/{token}/chart-data/` → `GET /reports/{token}/charts/{chartId}/data/`
   - Authenticated: `POST /api/charts/chart-data/` → `GET /api/reports/{snapshotId}/charts/{chartId}/data/`

2. **Table chart data** (lines ~508-556):
   - Public: `POST /reports/{token}/chart-data-preview/` → `GET /reports/{token}/charts/{chartId}/table-data/`
   - Authenticated: `useChartDataPreview(payload)` → `GET /api/reports/{snapshotId}/charts/{chartId}/table-data/`

3. **Table total rows** (lines ~577-597):
   - Public: `POST /reports/{token}/chart-data-preview/total-rows/` → `GET /reports/{token}/charts/{chartId}/table-data/total-rows/`
   - Authenticated: `useChartDataPreviewTotalRows(payload)` → `GET /api/reports/{snapshotId}/charts/{chartId}/table-data/total-rows/`

4. **Map data** (lines ~857-878):
   - Public: `POST /reports/{token}/map-data/` → `GET /reports/{token}/charts/{chartId}/map-data/`
   - Authenticated: `useMapDataOverlay(payload)` → `GET /api/reports/{snapshotId}/charts/{chartId}/map-data/`

**Frontend needs `snapshotId` for authenticated report endpoints.** This is already available as a prop (`snapshotId?: number`).

**What stays:**
- The `chartDataPayload` useMemo block (lines 356-465) still needed for:
  - Dashboard mode (non-report charts)
  - CSV export
- `effectiveChart` / `frozenChartConfig` still used for chart type detection, rendering config, title, etc.
- `mapDataOverlayPayload` still needed for dashboard mode maps

---

### Tests — 1 file

#### New test for report chart-data GET endpoints

Test both public and authenticated GET endpoints:
- Valid token/snapshot_id + valid chart_id → returns chart data
- Valid token/snapshot_id + invalid chart_id → 404
- Invalid token/snapshot_id → 404
- Period filters are injected correctly

---

## What Stays the Same

- **Dashboard path**: Completely untouched (both authenticated and public)
- **`charts_service.py`**: No changes
- **`charts_api.py`**: No changes — the generic POST chart endpoints stay, they just won't be used by reports anymore
- **Report view endpoint** (`GET /reports/{token}/view/` and `GET /api/reports/{id}/view/`): Still returns `frozen_chart_configs` to the frontend for layout rendering
- **Frontend dashboard mode**: All dashboard data fetching stays exactly the same
- **PDF generation**: Playwright renders the public report page which uses the new GET endpoints

---

## Key Concepts

### `chart_id` in `frozen_chart_configs`

When a report snapshot is created, all chart configs are deep-copied into `frozen_chart_configs` — a JSONField on `ReportSnapshot`. The original chart ID (e.g., `"42"`) is used as the dict key. This is just a string key linking layout components to chart configs — **not a DB FK**. The approach works even when original charts/dashboards are deleted because `frozen_chart_configs` is a self-contained JSON blob.

### `extra_config` key name mismatch

`extra_config` uses keys like `x_axis_column`, `dimension_column` while `ChartDataPayload` uses `x_axis`, `dimension_col`. The mapping is done in `_build_report_chart_payload`, same as `get_chart_data_by_id` does it.

### `time_grain`

Never unpacked into a top-level payload field. Always read directly from `extra_config.get("time_grain")` by the query builder. No special handling needed.

### `MapDataOverlayPayload` vs `ChartDataPayload`

Map charts use a different payload structure (`MapDataOverlayPayload`) that includes `geographic_column`, `value_column`, `aggregate_function`, and `filters`. These fields also come from the frozen chart config's `extra_config`, so the backend can build this payload too.

---

## Size Estimate

| Area | Files | Complexity |
|------|-------|------------|
| Backend: 4 public GET endpoints in `public_api.py` | 1 | Medium — 4 endpoints + 2 helpers |
| Backend: 4 authenticated GET endpoints in `report_api.py` | 1 | Medium — 4 endpoints (reuse helpers) |
| Frontend: Simplify report chart fetching | 1 | Medium — switch 8 POST calls to GET in report mode |
| Tests | 1 | Small — test new GET endpoints |
| **Total** | **4 files** | **Medium** |
