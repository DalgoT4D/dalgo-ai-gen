# Plan: Unify Chart Data Payload Construction (Backend-Owned)

## Context

Report charts show different data than dashboard charts for the same configuration. The root cause: **two separate code paths build the `ChartDataPayload`** — the backend builds it from the DB Chart model for dashboards, while the frontend manually reconstructs it from frozen config for reports. The frontend reconstruction was missing `time_grain` (and potentially other fields), causing monthly aggregation to be lost.

The spread fix (`...(effectiveChart.extra_config || {})`) patches the immediate bug, but the underlying problem remains: any future field added to `extra_config` that the frontend doesn't forward will silently break in report mode.

**Goal**: Single source of truth — the backend builds `ChartDataPayload` from chart config in both modes.

---

## Bug Summary

- **Dashboard** (Image 1): Line chart shows monthly aggregated data (~14 points, "Sep 2022", "Oct 2022", Y up to 2,500)
- **Report** (Image 2): Same line chart shows raw row-level data (hundreds of points, full ISO timestamps, Y up to 60)
- **Root cause**: `time_grain` field in `extra_config` was not being passed from frontend to backend in report mode
- **Quick fix applied**: Spread all `extra_config` fields in `chart-element-view.tsx` line 429
- **Permanent fix**: This plan — eliminate frontend payload construction for reports entirely

---

## Current Two Paths (The Problem)

**Dashboard (works)**: `GET /api/charts/{chart_id}/data/`
- Backend reads Chart from DB → builds `ChartDataPayload` at `charts_api.py:1150-1168` → passes to `generate_chart_data_and_config()`
- Backend passes `extra_config=extra_config` (the ENTIRE dict), so `time_grain` and all other fields reach the query builder

**Report (was broken)**: `POST /api/charts/chart-data/`
- Frontend manually maps 15+ fields from `frozenChartConfig` into a `ChartDataPayload` shape at `chart-element-view.tsx:356-465`
- Frontend was cherry-picking only `filters`, `pagination`, `sort` from `extra_config` — missing `time_grain`
- Backend receives incomplete payload → builds query without time grain → returns raw data

---

## Approaches Considered

### Approach 1: Eliminate the duplication (Best) ← CHOSEN
The report POST endpoint accepts the raw frozen config (same shape as the chart model) and the backend builds the payload server-side — identical to what the GET endpoint does. One code path, zero drift.

### Approach 2: Backend parity test (Safety net)
Add a backend test that creates a chart, calls the GET endpoint, simulates what the frontend would send to POST, and asserts both payloads are equivalent. Good as an additional safety net but doesn't eliminate the root cause.

### Approach 3: Frontend unit test (Acceptable)
Test that `chartDataPayload` preserves all `extra_config` keys. Weakest — still two code paths, just tested.

---

## Approach 1: Implementation Plan

### Files to Modify

#### 1. `ddpui/schemas/chart_schema.py` — Add frozen config schema

```python
class FrozenChartDataRequest(Schema):
    """Accepts a frozen chart config (same shape stored in report snapshots)."""
    chart_type: str
    schema_name: str
    table_name: str
    extra_config: Optional[dict] = None
```

Minimal — only the 4 fields the backend needs to build the full payload. No duplication of column fields.

#### 2. `ddpui/api/charts_api.py` — Extract helper + add endpoint

**a) Extract `build_chart_data_payload` helper** (from existing logic at lines 1150-1168):

```python
def build_chart_data_payload(
    chart_type: str,
    schema_name: str,
    table_name: str,
    extra_config: dict,
    dashboard_filters: list[dict] | None = None,
) -> ChartDataPayload:
    """Build ChartDataPayload from chart model fields.

    Single source of truth used by both the chart-by-ID GET endpoint
    and the frozen-config POST endpoint for reports.
    """
    customizations = extra_config.get("customizations", {})
    return ChartDataPayload(
        chart_type=chart_type,
        schema_name=schema_name,
        table_name=table_name,
        x_axis=extra_config.get("x_axis_column"),
        y_axis=extra_config.get("y_axis_column"),
        dimension_col=extra_config.get("dimension_column"),
        extra_dimension=extra_config.get("extra_dimension_column"),
        metrics=extra_config.get("metrics"),
        geographic_column=extra_config.get("geographic_column"),
        value_column=extra_config.get("value_column"),
        selected_geojson_id=extra_config.get("selected_geojson_id"),
        customizations=customizations,
        offset=0,
        limit=100,
        extra_config=extra_config,
        dashboard_filters=dashboard_filters,
    )
```

**b) Refactor `get_chart_data_by_id`** (line 1083) to use the helper:

```python
payload = build_chart_data_payload(
    chart.chart_type, chart.schema_name, chart.table_name,
    extra_config, resolved_dashboard_filters,
)
```

**c) Add new endpoint** `POST /api/charts/frozen-chart-data/`:

```python
@charts_router.post("/frozen-chart-data/", response=ChartDataResponse)
@has_permission(["can_view_dashboards"])
def get_frozen_chart_data(request, payload: FrozenChartDataRequest):
    """Get chart data from a frozen chart config (used by reports)."""
    org_warehouse = OrgWarehouse.objects.filter(org=request.orguser.org).first()
    if not org_warehouse:
        raise HttpError(404, "Warehouse not configured")

    chart_payload = build_chart_data_payload(
        payload.chart_type, payload.schema_name,
        payload.table_name, payload.extra_config or {},
    )
    result = generate_chart_data_and_config(chart_payload, org_warehouse)
    return ChartDataResponse(data=result["data"], echarts_config=result["echarts_config"])
```

Similarly add `POST /api/charts/frozen-chart-data-preview/` and `POST /api/charts/frozen-chart-data-preview/total-rows/` for table charts reusing the same helper.

#### 3. `ddpui/api/public_api.py` — Update public report endpoints

Update these 4 endpoints to use `build_chart_data_payload` instead of constructing `ChartDataPayload` from raw POST body:

- `get_public_report_chart_data` (line 1173) — bar/line/pie/number
- `get_public_report_table_data` (line 1205) — table preview
- `get_public_report_table_total_rows` (line 1244) — table total rows
- `get_public_report_map_data` (line 1274) — maps

Each currently does `ChartDataPayload(**payload_data)` from the raw body. Change to accept `FrozenChartDataRequest` shape and use `build_chart_data_payload()`.

#### 4. `webapp_v2/components/dashboard/chart-element-view.tsx` — Simplify frontend

**a) For report/frozen mode, send the frozen config directly** instead of the 100+ line manual payload construction:

```typescript
const frozenPayload = frozenChartConfig
  ? {
      chart_type: frozenChartConfig.chart_type,
      schema_name: frozenChartConfig.schema_name,
      table_name: frozenChartConfig.table_name,
      extra_config: {
        ...(frozenChartConfig.extra_config || {}),
        // Override filters with merged list (chart filters + resolved dashboard filters)
        filters: [
          ...(frozenChartConfig.extra_config?.filters || []),
          ...formatAsChartFilters(
            resolvedDashboardFilters.filter(
              (f) =>
                f.schema_name === frozenChartConfig.schema_name &&
                f.table_name === frozenChartConfig.table_name
            )
          ),
        ],
      },
    }
  : null;
```

**b) Update API URLs** for report mode:
- Private report: `POST /api/charts/frozen-chart-data/` (instead of `/api/charts/chart-data/`)
- Public report: `POST /api/v1/public/reports/{token}/chart-data/` (same URL, backend now expects frozen shape)

**c) Keep existing `chartDataPayload`** for non-report modes (chart creation/preview) — that code path remains unchanged.

**d) Simplify `mapDataOverlayPayload`** similarly — send frozen config shape for report mode maps.

---

## What Stays Unchanged

- `GET /api/charts/{chart_id}/data/` — dashboard mode, untouched (just refactored to use the shared helper)
- `POST /api/charts/chart-data/` — kept for chart creation/preview (non-report use cases)
- `_freeze_chart_configs()` in `report_service.py` — no changes needed
- `generate_chart_data_and_config()` — no changes, still receives `ChartDataPayload`
- Chart model, frozen config structure — no changes

---

## Implementation Order

1. `ddpui/schemas/chart_schema.py` — Add `FrozenChartDataRequest` schema
2. `ddpui/api/charts_api.py` — Extract `build_chart_data_payload` helper, refactor `get_chart_data_by_id` to use it, add `frozen-chart-data/` endpoints
3. `ddpui/api/public_api.py` — Update 4 public report endpoints to use `build_chart_data_payload`
4. `webapp_v2/components/dashboard/chart-element-view.tsx` — Simplify report mode to send frozen config; update URLs

---

## Verification

1. **Dashboard mode**: Open a dashboard with a line chart that has monthly `time_grain` → verify it still shows monthly aggregation (regression test)
2. **Report mode**: Open a report with the same line chart → verify it now shows monthly aggregation (the bug fix)
3. **Public report**: Open a public report link → verify charts render correctly
4. **Table charts**: Verify table charts work in both dashboard and report mode (with drill-down)
5. **Map charts**: Verify map charts work in both modes
6. **Chart creation**: Verify creating/previewing a new chart still works (uses the old `POST /api/charts/chart-data/` path)
7. **Backend test**: Test that `build_chart_data_payload` produces the same payload as the old inline construction for a chart with time_grain, filters, metrics, etc.
