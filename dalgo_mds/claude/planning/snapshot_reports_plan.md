# Snapshot Reports (Dashboard-Derived) - Implementation Plan

## 1. Overall

### What are we building?

A way to create **immutable, point-in-time snapshots** of dashboards. A user picks a dashboard, selects a relevant datetime column (from the dashboard's existing datetime filters), defines a date range, and the system freezes the dashboard config at that moment. Later, anyone can open that snapshot and see the dashboard exactly as it was configured — with data filtered to the chosen date range on the chosen datetime column.

### Core rules

- **No data copying** — we freeze dashboard + chart configs only. Data is queried live from the warehouse with a forced date filter.
- **Datetime field selection** — the user picks which datetime column defines the reporting period. This comes from the dashboard's existing datetime filters (the `DashboardFilter` records with `filter_type = "datetime"`).
- **Date range**: end date is **compulsory**, start date is **optional**. If start date is omitted, only an upper bound is applied.
- **Chart independence** — if someone deletes or edits a chart after a snapshot is taken, the snapshot still works because it has its own copy of the chart config.
- **Editable summary** — each snapshot has a `summary` field where users can write observations/notes above the dashboard view.

### Why not just use the existing dashboard?

Dashboards are living documents — they get edited, charts get added/removed, filters change. Reports need to answer: "What did we review for January?" That requires an immutable record.

---

## 2. Approach

### Why one table, not two?

The previous plan had a `Report` table (template: dashboard + date field + frequency) and a `ReportSnapshot` table (actual frozen data). But:

- `date_field_schema`, `date_field_table`, `date_field_column` on Report assumed one global date column for all charts. In reality, different charts may query different tables with different date columns. The date filter is better handled as a regular dashboard filter.
- `frequency` was only needed for auto-scheduling (Celery beat), which we're not building now.
- The `Report` table was basically just a grouping mechanism with `dashboard_id` + `frequency`. Without auto-scheduling, it adds no value — you'd always navigate Report → Snapshot anyway.

**Simpler approach**: One `ReportSnapshot` table. Each snapshot directly references a dashboard and stores a date range + frozen config. No intermediate "report definition" needed.

### How snapshot creation works

```
User clicks "Create Report" (from reports page or dashboard)
    ↓
Dialog asks:
    1. Select dashboard (if not pre-selected)
    2. Enter title (e.g., "January 2025 Review")
    3. Select datetime field — dropdown populated from the dashboard's
       datetime filters (DashboardFilter records with filter_type="datetime").
       Shows: "{table_name}.{column_name}" for each datetime filter.
    4. Enter end date (compulsory)
    5. Enter start date (optional — if omitted, no lower bound)
    ↓
Backend does 2 freezes:
    1. Freeze dashboard (layout, components, grid settings, filters)
    2. Freeze chart configs (fetch full Chart records for each chartId in components)
    ↓
All frozen data + date range + selected datetime column stored in one ReportSnapshot row
    ↓
User can view snapshot later — sees dashboard rendered from frozen config
    with a forced date filter on the selected datetime column
```

### How snapshot viewing works

```
Frontend calls: GET /api/reports/{id}/view/
    ↓
Backend returns:
    - dashboard_data (from frozen_dashboard, same shape as Dashboard.to_json())
    - frozen_chart_configs (full chart configs keyed by chart_id)
    - report_metadata (title, date range, date_column, summary, etc.)
    ↓
Frontend passes dashboard_data to DashboardNativeView component (already supports this!)
    ↓
For each chart, instead of:
    GET /api/charts/{chartId}/data/     ← needs Chart record in DB, breaks if deleted
Frontend uses:
    POST /api/charts/chart-data/        ← accepts config inline, no DB lookup needed
    (this endpoint ALREADY EXISTS — used by chart builder)
    ↓
The frozen chart config is sent in the POST body, PLUS a forced
datetime dashboard_filter is injected using:
    - date_column from report_metadata (schema_name, table_name, column_name)
    - period_start / period_end from report_metadata
    ↓
Backend's apply_dashboard_filters() adds WHERE clauses:
    - column_name >= period_start  (if start date provided)
    - column_name <= period_end + "T23:59:59"  (always — end date is compulsory)
    ↓
Dashboard + charts render in read-only mode with date-filtered data.
No dependency on Dashboard or Chart tables.
```

### How the date range filter is applied (end-to-end)

The existing dashboard filter system handles this naturally:

1. **DashboardFilter model** has `filter_type = "datetime"` with `schema_name`, `table_name`, `column_name`
2. The frontend normally resolves filter IDs → column info via `resolveDashboardFilters()`
3. `apply_dashboard_filters()` in the backend adds WHERE clauses based on column + value

For report mode, we skip the normal filter resolution and inject a **forced datetime filter** directly:

```typescript
// At view time, frontend builds a forced filter from report metadata:
const reportDateFilter = {
  filter_id: 'report_period',         // synthetic ID
  column: date_column.column_name,
  type: 'datetime',
  value: {
    start_date: period_start || undefined,  // optional
    end_date: period_end,                   // always present
  },
  settings: {},
  schema_name: date_column.schema_name,
  table_name: date_column.table_name,
};

// This is added to dashboard_filters in the POST /api/charts/chart-data/ payload
```

The backend's `apply_dashboard_filters()` already handles datetime filters with optional start/end. The filter is only applied to charts whose `table_name` matches — charts querying other tables are unaffected (existing behavior).

**Key insight**: Two chart data endpoints already exist in the backend:

| Endpoint | How it works | Needs Chart in DB? |
|----------|-------------|-------------------|
| `GET /api/charts/{id}/data/` | Looks up Chart record, builds payload from `chart.extra_config`, calls `generate_chart_data_and_config()` | YES |
| `POST /api/charts/chart-data/` | Accepts `ChartDataPayload` directly (schema_name, table_name, extra_config), calls the same `generate_chart_data_and_config()` | NO |

Both call the **same underlying function**. For report mode, the frontend uses the POST endpoint with frozen config. No new backend code needed for data fetching.

### What gets frozen (2 layers)

The dashboard `components` JSON stores chart references like this:

```json
{
    "chart-1769275109095": {
        "id": "chart-1769275109095",
        "type": "chart",
        "config": {
            "chartId": 8,
            "chartType": "pie",
            "title": "Pie chart - surveys",
            "description": null
        }
    }
}
```

`chartId: 8` is just a reference. The actual config lives in the `Chart` table. So we need to freeze two things:

| Layer | Source | What's stored | Why separate? |
|-------|--------|---------------|---------------|
| `frozen_dashboard` | `Dashboard` model + `DashboardFilter` table | layout_config, components, grid_columns, target_screen_size, filter_layout, title, description, filters (array of filter configs) | Dashboard layout + filters in one object. Filters come from a separate table but are logically part of the dashboard. |
| `frozen_chart_configs` | `Chart` table | Dict of `{chartId: {title, chart_type, schema_name, table_name, extra_config}}` | Components only store `chartId` reference. We need the full config so snapshots survive chart deletion/editing |

### What's frozen vs what's live

| Data | Frozen? | Stored in | Why? |
|------|---------|-----------|------|
| Dashboard layout (grid positions) | YES | `frozen_dashboard` | Layout may change later |
| Dashboard components list | YES | `frozen_dashboard` | Charts may be added/removed |
| Dashboard filters | YES | `frozen_dashboard.filters` | Filters may change |
| Chart configs (extra_config, chart_type, schema, table) | YES | `frozen_chart_configs` | Charts may be deleted/edited |
| Actual warehouse data | NO — queried live | — | Data corrections, late arrivals should be reflected |

---

## 3. Database Schema

### One new table: `report_snapshot`

```python
# DDP_backend/ddpui/models/report.py

"""Report models for Dalgo platform"""

from enum import Enum
from django.db import models
from ddpui.models.org import Org
from ddpui.models.org_user import OrgUser


class SnapshotStatus(str, Enum):
    """Snapshot status enum"""
    GENERATED = "generated"
    VIEWED = "viewed"
    ARCHIVED = "archived"

    @classmethod
    def choices(cls):
        return [(key.value, key.name) for key in cls]


class ReportSnapshot(models.Model):
    """
    Immutable snapshot of a dashboard at a specific date range.
    Stores frozen config — data is queried live with date filter.

    No FK to Dashboard or Chart. Once frozen, the snapshot is fully
    self-contained. Deleting dashboards or charts does NOT affect snapshots.
    """
    id = models.BigAutoField(primary_key=True)
    title = models.CharField(max_length=255, help_text="User-provided title for this snapshot")

    # Selected datetime column for the report period filter
    # Stores: {"schema_name": "...", "table_name": "...", "column_name": "..."}
    # Comes from the dashboard's DashboardFilter with filter_type="datetime"
    date_column = models.JSONField(
        default=dict,
        help_text="Selected datetime column: {schema_name, table_name, column_name}",
    )

    # Date range this snapshot covers
    # period_start is optional (NULL = no lower bound)
    # period_end is compulsory (always set)
    period_start = models.DateField(
        null=True, blank=True,
        help_text="Start of reporting period (inclusive). NULL = no lower bound.",
    )
    period_end = models.DateField(help_text="End of reporting period (inclusive). Always set.")

    # --- FROZEN DATA (2 layers) ---

    # Layer 1: Dashboard layout, structure & filters
    # Contains: title, description, grid_columns, target_screen_size,
    #           layout_config, components, filter_layout, filters
    frozen_dashboard = models.JSONField(
        default=dict,
        help_text="Frozen dashboard config + filters at snapshot time",
    )

    # Layer 2: Full chart configs keyed by chart_id
    # {str(chart_id): {id, title, description, chart_type, schema_name, table_name, extra_config}}
    # CRITICAL: ensures snapshots survive chart deletion or editing
    frozen_chart_configs = models.JSONField(
        default=dict,
        help_text="Frozen chart configs keyed by chart_id",
    )

    # User-editable summary (the ONLY mutable field)
    summary = models.TextField(
        blank=True,
        null=True,
        help_text="Executive summary or notes displayed above the dashboard",
    )

    # Status tracking
    status = models.CharField(
        max_length=20,
        choices=SnapshotStatus.choices(),
        default=SnapshotStatus.GENERATED.value,
    )

    # Metadata
    created_by = models.ForeignKey(
        OrgUser, on_delete=models.SET_NULL, null=True,
        help_text="User who created this snapshot",
    )
    org = models.ForeignKey(Org, on_delete=models.CASCADE)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        start = self.period_start or "..."
        return f"{self.title} ({start} - {self.period_end})"

    class Meta:
        db_table = "report_snapshot"
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["org", "created_at"]),
        ]
```

### Why each field exists

| Field | Why it's needed |
|-------|----------------|
| `title` | User gives each snapshot a name (e.g., "January 2025 Review") |
| `date_column` | Which datetime column the date range applies to. Stored as `{schema_name, table_name, column_name}`. Selected by user from the dashboard's datetime filters at creation time. |
| `period_start` | Start of reporting period. **Optional** — NULL means no lower bound on the date filter. |
| `period_end` | End of reporting period. **Always required.** |
| `frozen_dashboard` | Dashboard layout, components, and filters frozen at snapshot time (see Layer 1 above) |
| `frozen_chart_configs` | Full chart configs frozen at snapshot time (see Layer 2 above). This is what makes snapshots survive chart deletion. |
| `summary` | Editable field for observations/notes. Only mutable field on a snapshot. |
| `status` | Track if snapshot has been viewed |
| `created_by` | Who created it |
| `org` | Multi-tenancy |

### Why NO foreign key to Dashboard or Chart

Once a snapshot is created, it must be **completely self-contained**. Deleting a dashboard or any chart should have zero effect on existing snapshots.

- `frozen_dashboard` has the full dashboard layout + filters (no need to read from Dashboard or DashboardFilter table)
- `frozen_chart_configs` has the full chart configs (no need to read from Chart table)
- Chart data is fetched via `POST /api/charts/chart-data/` which takes config inline (no Chart DB lookup)

The only thing we need from the creation flow is the `dashboard_id` parameter — to know which dashboard to freeze. But that's a creation-time parameter, not a stored FK.

### What was removed vs previous plan

| Removed | Why |
|---------|-----|
| `Report` table | Was only useful for grouping + auto-scheduling. Without scheduler, it's unnecessary indirection. |
| `dashboard` FK | Snapshot must survive dashboard deletion. Frozen config has everything needed. |
| `date_field_schema/table/column` (on Report) | The old Report table stored these as separate fields. We now store this as `date_column` JSON on ReportSnapshot — selected by the user from the dashboard's existing datetime filters. |
| `frequency` | Only needed for auto-scheduling (Celery beat), which is out of scope. |
| `is_active` | Only needed for auto-scheduling. |
| `version_number` | Was auto-incrementing within a Report. Without Report grouping, not needed — `created_at` ordering is sufficient. |
| `period_label` | Can be derived from `period_start`/`period_end` on the frontend. No need to store. |

---

## 4. Files to Change

### Backend — New Files

| File | Purpose |
|------|---------|
| `DDP_backend/ddpui/models/report.py` | `ReportSnapshot` model |
| `DDP_backend/ddpui/core/reports/__init__.py` | Module exports |
| `DDP_backend/ddpui/core/reports/report_service.py` | Business logic (freeze, view, CRUD) |
| `DDP_backend/ddpui/core/reports/exceptions.py` | Custom exceptions |
| `DDP_backend/ddpui/schemas/report_schema.py` | Pydantic request/response schemas |
| `DDP_backend/ddpui/api/report_api.py` | API endpoints |
| `DDP_backend/ddpui/tests/api_tests/test_report_api.py` | Tests |

### Backend — Modified Files

| File | Change |
|------|--------|
| `DDP_backend/ddpui/routes.py` | Add `report_router` registration (~3 lines) |

### Frontend — New Files

| File | Purpose |
|------|---------|
| `webapp_v2/hooks/api/useReports.ts` | SWR hooks + mutation functions |
| `webapp_v2/app/reports/page.tsx` | Snapshot list page |
| `webapp_v2/app/reports/[snapshotId]/page.tsx` | Snapshot viewer page |
| `webapp_v2/components/reports/create-snapshot-dialog.tsx` | Creation dialog |

### Frontend — Modified Files

| File | Change | Lines |
|------|--------|-------|
| `webapp_v2/components/dashboard/dashboard-native-view.tsx` | Add `isReportMode`, `frozenChartConfigs`, `reportMetadata` props; pass frozen config + report metadata down to `ChartElementView`; hide edit/share/delete buttons in report mode | ~25 lines |
| `webapp_v2/components/dashboard/chart-element-view.tsx` | **Key change**: Accept optional `frozenChartConfig` + `reportMetadata` props; skip `useChart()` API call when frozen config is provided; inject forced datetime filter from reportMetadata into chart data POST requests | ~30 lines |
| `webapp_v2/components/dashboard/unified-filters-panel.tsx` | Check `is_forced` flag to disable filter widget | ~3 lines |
| Navigation component | Add "Reports" link | ~1 line |

**Total frontend modifications to existing files: ~60 lines across 3 components**

### Why `chart-element-view.tsx` is the key change

Today, `ChartElementView` always fetches the chart from the API:

```typescript
// chart-element-view.tsx, line 188
const { data: chart } = useChart(isPublicMode ? null : chartId);
```

Then it uses `chart.schema_name`, `chart.table_name`, `chart.extra_config` to build data queries. If the chart is deleted → 404 → broken.

For report mode, we skip the API call and use the frozen config instead:

```typescript
// New behavior:
const { data: chart } = useChart(frozenChartConfig ? null : chartId);
const effectiveChart = frozenChartConfig || chart;
```

This is the **same pattern** already used for `isPublicMode` — public mode skips the private chart API and uses public endpoints. Report mode skips the chart API entirely and uses pre-fetched frozen data.

The data flows like this:

```
ReportSnapshot.frozen_dashboard + frozen_chart_configs + date_column + period  (stored in DB)
        ↓
GET /api/reports/{id}/view/   (returned by backend)
        ↓
SnapshotViewerPage                      (receives frozen_chart_configs + report_metadata)
        ↓
DashboardNativeView                     (passes per-chart config + report_metadata down)
        ↓
ChartElementView(frozenChartConfig, reportMetadata)
    → uses frozen config instead of useChart() API call
    → builds forced datetime filter from reportMetadata.date_column + period
        ↓
POST /api/charts/chart-data/            (frozen config + forced date filter sent inline)
    → apply_dashboard_filters() adds WHERE clause for date range
        ↓
Data returned filtered to the snapshot's date range
```

**Important**: In report mode, chart data is fetched via `POST /api/charts/chart-data/` (not the `GET /api/charts/{id}/data/` endpoint). The POST endpoint accepts `ChartDataPayload` inline — `schema_name`, `table_name`, `extra_config`, etc. — and calls the same `generate_chart_data_and_config()` function. This means snapshots work even if the original chart has been deleted.

---

## 5. Code Implementation

### 5.1 Backend — Exceptions

**File**: `DDP_backend/ddpui/core/reports/exceptions.py`

```python
"""Report exceptions"""


class ReportError(Exception):
    """Base exception for report errors"""
    def __init__(self, message: str, error_code: str = "REPORT_ERROR"):
        self.message = message
        self.error_code = error_code
        super().__init__(self.message)


class SnapshotNotFoundError(ReportError):
    def __init__(self, snapshot_id: int):
        super().__init__(
            f"Snapshot with id {snapshot_id} not found", "SNAPSHOT_NOT_FOUND"
        )

class SnapshotValidationError(ReportError):
    def __init__(self, message: str):
        super().__init__(message, "SNAPSHOT_VALIDATION_ERROR")
```

### 5.2 Backend — Service

**File**: `DDP_backend/ddpui/core/reports/report_service.py`

```python
"""Report service — snapshot creation, viewing, CRUD"""

from typing import Optional, List, Dict, Any
from datetime import date

from django.db.models import Q

from ddpui.models.org import Org
from ddpui.models.org_user import OrgUser
from ddpui.models.dashboard import Dashboard, DashboardFilter
from ddpui.models.report import ReportSnapshot, SnapshotStatus
from ddpui.models.visualization import Chart
from ddpui.services.dashboard_service import DashboardService
from ddpui.utils.custom_logger import CustomLogger

from .exceptions import SnapshotNotFoundError, SnapshotValidationError

logger = CustomLogger("ddpui.core.reports")


class ReportService:
    """Service class for snapshot operations"""

    # =========================================================================
    # Config Freezing
    # =========================================================================

    @staticmethod
    def _freeze_dashboard(dashboard: Dashboard) -> Dict[str, Any]:
        """Freeze dashboard layout, structure & filters into one dict."""
        filters = dashboard.filters.all().order_by("order")
        return {
            "title": dashboard.title,
            "description": dashboard.description,
            "grid_columns": dashboard.grid_columns,
            "target_screen_size": dashboard.target_screen_size,
            "layout_config": dashboard.layout_config,
            "components": dashboard.components,
            "filter_layout": dashboard.filter_layout,
            "filters": [f.to_json() for f in filters],
        }

    @staticmethod
    def _freeze_chart_configs(dashboard: Dashboard) -> Dict[str, Any]:
        """
        Layer 2: Freeze ALL chart configs referenced in dashboard components.

        Walks through components, extracts chartId from each chart component,
        batch-fetches the Chart records, and stores full configs.
        """
        components = dashboard.components or {}
        chart_ids = []

        for comp_id, component in components.items():
            if component.get("type") == "chart":
                chart_id = component.get("config", {}).get("chartId")
                if chart_id:
                    chart_ids.append(chart_id)

        charts = Chart.objects.filter(id__in=chart_ids, org=dashboard.org)
        frozen = {}
        for chart in charts:
            frozen[str(chart.id)] = {
                "id": chart.id,
                "title": chart.title,
                "description": chart.description,
                "chart_type": chart.chart_type,
                "schema_name": chart.schema_name,
                "table_name": chart.table_name,
                "extra_config": chart.extra_config,
            }
        return frozen

    # =========================================================================
    # Snapshot CRUD
    # =========================================================================

    @staticmethod
    def create_snapshot(
        title: str,
        dashboard_id: int,
        date_column: Dict[str, str],
        period_end: date,
        orguser: OrgUser,
        period_start: Optional[date] = None,
    ) -> ReportSnapshot:
        """
        Create a snapshot from a dashboard.

        Steps:
        1. Validate dashboard exists
        2. Validate date range (if both dates provided)
        3. Validate date_column references a real datetime filter on this dashboard
        4. Freeze 2 layers of config
        5. Create snapshot record (no FK to dashboard — fully self-contained)

        period_start is optional (no lower bound if omitted).
        period_end is always required.
        date_column = {"schema_name": ..., "table_name": ..., "column_name": ...}
        """
        if period_start is not None and period_start > period_end:
            raise SnapshotValidationError("period_start must be before period_end")

        # Validate date_column has required keys
        required_keys = {"schema_name", "table_name", "column_name"}
        if not required_keys.issubset(date_column.keys()):
            raise SnapshotValidationError(
                f"date_column must contain: {required_keys}"
            )

        # Fetch dashboard with filters prefetched (used only for freezing)
        try:
            dashboard = Dashboard.objects.prefetch_related("filters").get(
                id=dashboard_id, org=orguser.org
            )
        except Dashboard.DoesNotExist:
            raise SnapshotValidationError(f"Dashboard {dashboard_id} not found")

        # Validate that the selected date_column matches a datetime filter on this dashboard
        datetime_filters = dashboard.filters.filter(filter_type="datetime")
        match = datetime_filters.filter(
            schema_name=date_column["schema_name"],
            table_name=date_column["table_name"],
            column_name=date_column["column_name"],
        ).exists()
        if not match:
            raise SnapshotValidationError(
                f"No datetime filter found on dashboard for "
                f"{date_column['schema_name']}.{date_column['table_name']}.{date_column['column_name']}"
            )

        frozen_dashboard = ReportService._freeze_dashboard(dashboard)
        frozen_chart_configs = ReportService._freeze_chart_configs(dashboard)

        snapshot = ReportSnapshot.objects.create(
            title=title,
            date_column=date_column,
            period_start=period_start,
            period_end=period_end,
            frozen_dashboard=frozen_dashboard,
            frozen_chart_configs=frozen_chart_configs,
            created_by=orguser,
            org=orguser.org,
        )

        logger.info(f"Created snapshot {snapshot.id} from dashboard {dashboard_id}")
        return snapshot

    @staticmethod
    def list_snapshots(org: Org, search: Optional[str] = None) -> List[ReportSnapshot]:
        """List all snapshots for an org."""
        query = Q(org=org)
        if search:
            query &= Q(title__icontains=search)
        return list(
            ReportSnapshot.objects.filter(query)
            .select_related("created_by__user")
            .order_by("-created_at")
        )

    @staticmethod
    def get_snapshot(snapshot_id: int, org: Org) -> ReportSnapshot:
        """Get a snapshot by ID."""
        try:
            return ReportSnapshot.objects.select_related("created_by__user").get(
                id=snapshot_id, org=org
            )
        except ReportSnapshot.DoesNotExist:
            raise SnapshotNotFoundError(snapshot_id)

    @staticmethod
    def get_snapshot_view_data(snapshot_id: int, org: Org) -> Dict[str, Any]:
        """
        Get full data to render a snapshot in the frontend.

        Returns dashboard_data shaped like Dashboard.to_json() so
        DashboardNativeView can render it directly.
        """
        snapshot = ReportService.get_snapshot(snapshot_id, org)

        # Mark as viewed
        if snapshot.status == SnapshotStatus.GENERATED.value:
            snapshot.status = SnapshotStatus.VIEWED.value
            snapshot.save(update_fields=["status"])

        # Build dashboard-like response from frozen dashboard
        # No dashboard_id — snapshot is fully self-contained
        dashboard_data = {
            **snapshot.frozen_dashboard,
            "id": snapshot.id,  # Use snapshot ID as the dashboard "id" for rendering
            "dashboard_type": "native",
            "is_published": True,
            "is_locked": False,
            "locked_by": None,
            "is_public": False,
            "created_by": snapshot.created_by.user.email if snapshot.created_by else None,
            "org_id": snapshot.org.id,
            "created_at": snapshot.created_at.isoformat(),
            "updated_at": snapshot.created_at.isoformat(),
        }

        report_metadata = {
            "snapshot_id": snapshot.id,
            "title": snapshot.title,
            "date_column": snapshot.date_column,  # {schema_name, table_name, column_name}
            "period_start": snapshot.period_start.isoformat() if snapshot.period_start else None,
            "period_end": snapshot.period_end.isoformat(),
            "summary": snapshot.summary,
            "status": snapshot.status,
            "created_at": snapshot.created_at.isoformat(),
            "created_by": snapshot.created_by.user.email if snapshot.created_by else None,
            "dashboard_title": snapshot.frozen_dashboard.get("title", ""),
        }

        return {
            "dashboard_data": dashboard_data,
            "report_metadata": report_metadata,
            "frozen_chart_configs": snapshot.frozen_chart_configs or {},
        }

    @staticmethod
    def update_snapshot(snapshot_id: int, org: Org, **fields) -> ReportSnapshot:
        """Update mutable fields on a snapshot."""
        snapshot = ReportService.get_snapshot(snapshot_id, org)
        update_fields = []
        for field, value in fields.items():
            if value is not None:
                setattr(snapshot, field, value)
                update_fields.append(field)
        if update_fields:
            snapshot.save(update_fields=update_fields)
        return snapshot

    @staticmethod
    def delete_snapshot(snapshot_id: int, org: Org) -> bool:
        """Delete a snapshot."""
        snapshot = ReportService.get_snapshot(snapshot_id, org)
        snapshot.delete()
        logger.info(f"Deleted snapshot {snapshot_id}")
        return True
```

### 5.3 Backend — Schemas

**File**: `DDP_backend/ddpui/schemas/report_schema.py`

```python
"""Report schemas for request/response validation"""

from typing import Optional, Dict, Any
from datetime import date, datetime
from ninja import Schema, Field


# Nested schemas

class DateColumnSchema(Schema):
    """The datetime column that defines the report period"""
    schema_name: str
    table_name: str
    column_name: str


# Request schemas

class SnapshotCreate(Schema):
    title: str = Field(..., min_length=1, max_length=255)
    dashboard_id: int
    date_column: DateColumnSchema         # Selected datetime filter column
    period_end: date                       # Compulsory
    period_start: Optional[date] = None    # Optional (no lower bound if omitted)


class SnapshotUpdate(Schema):
    summary: Optional[str] = Field(None, max_length=10000)


# Response schemas

class SnapshotListResponse(Schema):
    id: int
    title: str
    dashboard_title: Optional[str]  # From frozen_dashboard, not a live FK
    date_column: Dict[str, str]     # {schema_name, table_name, column_name}
    period_start: Optional[date]    # NULL if no lower bound
    period_end: date                # Always set
    status: str
    summary: Optional[str]
    created_by: Optional[str]
    created_at: datetime


class SnapshotViewResponse(Schema):
    dashboard_data: Dict[str, Any]
    report_metadata: Dict[str, Any]
    frozen_chart_configs: Dict[str, Any]
```

### 5.4 Backend — API

**File**: `DDP_backend/ddpui/api/report_api.py`

```python
"""Report API endpoints"""

from ninja import Router
from ninja.errors import HttpError

from ddpui.auth import has_permission
from ddpui.models.org_user import OrgUser
from ddpui.utils.custom_logger import CustomLogger

from ddpui.core.reports.report_service import ReportService
from ddpui.core.reports.exceptions import (
    SnapshotNotFoundError,
    SnapshotValidationError,
)
from ddpui.schemas.report_schema import (
    SnapshotCreate,
    SnapshotUpdate,
    SnapshotListResponse,
    SnapshotViewResponse,
)

logger = CustomLogger("ddpui.report_api")

report_router = Router()


@report_router.get("/", response=list[SnapshotListResponse])
@has_permission(["can_view_dashboards"])
def list_snapshots(request, search: str = None):
    """List all snapshots for the organization"""
    orguser: OrgUser = request.orguser
    snapshots = ReportService.list_snapshots(orguser.org, search=search)
    return [
        SnapshotListResponse(
            id=s.id,
            title=s.title,
            dashboard_title=s.frozen_dashboard.get("title") if s.frozen_dashboard else None,
            date_column=s.date_column,
            period_start=s.period_start,
            period_end=s.period_end,
            status=s.status,
            summary=s.summary,
            created_by=s.created_by.user.email if s.created_by else None,
            created_at=s.created_at,
        )
        for s in snapshots
    ]


@report_router.post("/", response=SnapshotListResponse)
@has_permission(["can_create_dashboards"])
def create_snapshot(request, payload: SnapshotCreate):
    """Create a new snapshot from a dashboard"""
    orguser: OrgUser = request.orguser
    try:
        s = ReportService.create_snapshot(
            title=payload.title,
            dashboard_id=payload.dashboard_id,
            date_column=payload.date_column.dict(),
            period_end=payload.period_end,
            period_start=payload.period_start,
            orguser=orguser,
        )
        return SnapshotListResponse(
            id=s.id,
            title=s.title,
            dashboard_title=s.frozen_dashboard.get("title") if s.frozen_dashboard else None,
            date_column=s.date_column,
            period_start=s.period_start,
            period_end=s.period_end,
            status=s.status,
            summary=s.summary,
            created_by=orguser.user.email,
            created_at=s.created_at,
        )
    except SnapshotValidationError as err:
        raise HttpError(400, str(err)) from err
    except Exception as e:
        logger.error(f"Error creating snapshot: {e}")
        raise HttpError(500, "Failed to create snapshot") from e


@report_router.get("/{snapshot_id}/view/", response=SnapshotViewResponse)
@has_permission(["can_view_dashboards"])
def get_snapshot_view(request, snapshot_id: int):
    """Get snapshot view data for rendering"""
    orguser: OrgUser = request.orguser
    try:
        view_data = ReportService.get_snapshot_view_data(snapshot_id, orguser.org)
        return SnapshotViewResponse(**view_data)
    except SnapshotNotFoundError as err:
        raise HttpError(404, str(err)) from err


@report_router.put("/{snapshot_id}/")
@has_permission(["can_edit_dashboards"])
def update_snapshot(request, snapshot_id: int, payload: SnapshotUpdate):
    """Update a snapshot"""
    orguser: OrgUser = request.orguser
    try:
        snapshot = ReportService.update_snapshot(
            snapshot_id, orguser.org, **payload.dict(exclude_none=True)
        )
        return {"success": True, "summary": snapshot.summary}
    except SnapshotNotFoundError as err:
        raise HttpError(404, str(err)) from err


@report_router.delete("/{snapshot_id}/")
@has_permission(["can_delete_dashboards"])
def delete_snapshot(request, snapshot_id: int):
    """Delete a snapshot"""
    orguser: OrgUser = request.orguser
    try:
        ReportService.delete_snapshot(snapshot_id, orguser.org)
        return {"success": True}
    except SnapshotNotFoundError as err:
        raise HttpError(404, str(err)) from err
```

**Register in `DDP_backend/ddpui/routes.py`**:

```python
from ddpui.api.report_api import report_router
report_router.tags = ["Reports"]
src_api.add_router("/api/reports/", report_router)
```

### 5.5 Frontend — Hooks

**File**: `webapp_v2/hooks/api/useReports.ts`

```typescript
import useSWR from 'swr';
import { apiGet, apiPost, apiPut, apiDelete } from '@/lib/api';

export interface DateColumn {
  schema_name: string;
  table_name: string;
  column_name: string;
}

export interface ReportSnapshot {
  id: number;
  title: string;
  dashboard_title?: string;
  date_column: DateColumn;
  period_start?: string;     // Optional (no lower bound if omitted)
  period_end: string;        // Always set (compulsory)
  status: 'generated' | 'viewed' | 'archived';
  summary?: string;
  created_by?: string;
  created_at: string;
}

export interface FrozenChartConfig {
  id: number;
  title: string;
  description?: string;
  chart_type: string;
  schema_name: string;
  table_name: string;
  extra_config: Record<string, any>;
}

export interface SnapshotViewData {
  dashboard_data: any;
  report_metadata: {
    snapshot_id: number;
    title: string;
    date_column: DateColumn;
    period_start?: string;       // Optional
    period_end: string;          // Always set
    summary?: string;
    status: string;
    created_at: string;
    created_by?: string;
    dashboard_title: string;
  };
  frozen_chart_configs: Record<string, FrozenChartConfig>;
}

// Hooks

export function useSnapshots(search?: string) {
  const params = search ? `?search=${encodeURIComponent(search)}` : '';
  const { data, error, mutate } = useSWR<ReportSnapshot[]>(
    `/api/reports/${params}`,
    apiGet,
    { revalidateOnFocus: true }
  );
  return { snapshots: data || [], isLoading: !error && !data, isError: error, mutate };
}

export function useSnapshotView(snapshotId: number | null) {
  const { data, error, mutate } = useSWR<SnapshotViewData>(
    snapshotId ? `/api/reports/${snapshotId}/view/` : null,
    apiGet
  );
  return { viewData: data, isLoading: !error && !data, isError: error, mutate };
}

// Mutations

export async function createSnapshot(data: {
  title: string;
  dashboard_id: number;
  date_column: DateColumn;
  period_end: string;           // Compulsory
  period_start?: string | null; // Optional
}) {
  return apiPost('/api/reports/', data);
}

export async function updateSnapshot(
  snapshotId: number,
  data: { summary?: string }
) {
  return apiPut(`/api/reports/${snapshotId}/`, data);
}

export async function deleteSnapshot(snapshotId: number) {
  return apiDelete(`/api/reports/${snapshotId}/`);
}
```

### 5.6 Frontend — Snapshot Viewer Page

**File**: `webapp_v2/app/reports/[snapshotId]/page.tsx`

```typescript
'use client';

import { useState } from 'react';
import { useParams, useRouter } from 'next/navigation';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { ArrowLeft, Calendar, Clock, Edit, Save, X } from 'lucide-react';
import { useSnapshotView, updateSnapshot } from '@/hooks/api/useReports';
import { DashboardNativeView } from '@/components/dashboard/dashboard-native-view';
import { Skeleton } from '@/components/ui/skeleton';
import { useToast } from '@/components/ui/use-toast';

export default function SnapshotViewerPage() {
  const params = useParams();
  const router = useRouter();
  const { toast } = useToast();
  const snapshotId = Number(params.snapshotId);

  const { viewData, isLoading, isError, mutate } = useSnapshotView(snapshotId);

  const [isEditingSummary, setIsEditingSummary] = useState(false);
  const [summaryDraft, setSummaryDraft] = useState('');
  const [isSavingSummary, setIsSavingSummary] = useState(false);

  if (isLoading) {
    return (
      <div className="p-6 space-y-4">
        <Skeleton className="h-8 w-64" />
        <Skeleton className="h-[600px] w-full" />
      </div>
    );
  }

  if (isError || !viewData) {
    return (
      <div className="p-6">
        <p className="text-red-500">Failed to load snapshot.</p>
        <Button variant="outline" onClick={() => router.back()}>Go Back</Button>
      </div>
    );
  }

  const { dashboard_data, report_metadata, frozen_chart_configs } = viewData;

  const handleSaveSummary = async () => {
    setIsSavingSummary(true);
    try {
      await updateSnapshot(snapshotId, { summary: summaryDraft });
      mutate();
      setIsEditingSummary(false);
      toast({ title: 'Summary saved' });
    } catch {
      toast({ title: 'Failed to save summary', variant: 'destructive' });
    } finally {
      setIsSavingSummary(false);
    }
  };

  return (
    <div className="flex flex-col h-full">
      {/* Header */}
      <div className="flex items-center justify-between px-4 py-3 border-b bg-muted/30">
        <div className="flex items-center gap-3">
          <Button variant="ghost" size="sm" onClick={() => router.back()}>
            <ArrowLeft className="h-4 w-4 mr-1" /> Back
          </Button>
          <div>
            <h1 className="text-lg font-semibold">{report_metadata.title}</h1>
            <div className="flex items-center gap-2 text-sm text-muted-foreground">
              <Calendar className="h-3 w-3" />
              <span>
                {report_metadata.period_start
                  ? `${report_metadata.period_start} — ${report_metadata.period_end}`
                  : `Up to ${report_metadata.period_end}`}
              </span>
            </div>
          </div>
        </div>
        <div className="text-xs text-muted-foreground flex items-center gap-1">
          <Clock className="h-3 w-3" />
          {new Date(report_metadata.created_at).toLocaleDateString()}
        </div>
      </div>

      {/* Summary */}
      <div className="px-4 py-3 border-b bg-background">
        {isEditingSummary ? (
          <div className="space-y-2">
            <Textarea
              value={summaryDraft}
              onChange={(e) => setSummaryDraft(e.target.value)}
              placeholder="Write observations, key findings, or action items..."
              rows={4}
              className="resize-y"
            />
            <div className="flex gap-2">
              <Button size="sm" onClick={handleSaveSummary} disabled={isSavingSummary}>
                <Save className="h-3 w-3 mr-1" /> {isSavingSummary ? 'Saving...' : 'Save'}
              </Button>
              <Button size="sm" variant="ghost" onClick={() => setIsEditingSummary(false)}>
                <X className="h-3 w-3 mr-1" /> Cancel
              </Button>
            </div>
          </div>
        ) : (
          <div className="flex items-start justify-between">
            <div className="flex-1">
              {report_metadata.summary ? (
                <p className="text-sm whitespace-pre-wrap">{report_metadata.summary}</p>
              ) : (
                <p className="text-sm text-muted-foreground italic">No summary yet.</p>
              )}
            </div>
            <Button
              variant="ghost"
              size="sm"
              onClick={() => { setSummaryDraft(report_metadata.summary || ''); setIsEditingSummary(true); }}
              className="ml-2 shrink-0"
            >
              <Edit className="h-3 w-3 mr-1" />
              {report_metadata.summary ? 'Edit' : 'Add Summary'}
            </Button>
          </div>
        )}
      </div>

      {/* Dashboard — reused component */}
      <div className="flex-1">
        <DashboardNativeView
          dashboardId={dashboard_data.id}
          dashboardData={dashboard_data}
          isReportMode={true}
          frozenChartConfigs={frozen_chart_configs}
          reportMetadata={{
            date_column: report_metadata.date_column,
            period_start: report_metadata.period_start,
            period_end: report_metadata.period_end,
          }}
          hideHeader={true}
        />
      </div>
    </div>
  );
}
```

### 5.7 Frontend — Modifications to Existing Components

#### `dashboard-native-view.tsx` — pass frozen config down

**What it does today**: Renders the grid, iterates over `components`, and for each chart component renders `<ChartElementView chartId={component.config.chartId} />`.

**What changes**: Add `isReportMode` + `frozenChartConfigs` props. When rendering chart components in report mode, look up the frozen config and pass it down:

```typescript
// Add to props interface:
interface DashboardNativeViewProps {
  // ... existing props ...
  isReportMode?: boolean;
  frozenChartConfigs?: Record<string, FrozenChartConfig>;
  reportMetadata?: {
    date_column: { schema_name: string; table_name: string; column_name: string };
    period_start?: string;
    period_end: string;
  };
}

// In the component render, where it renders ChartElementView (line ~507):
case 'chart':
  return (
    <ChartElementView
      chartId={component.config?.chartId}
      dashboardFilters={selectedFilters}
      dashboardFilterConfigs={dashboardFilters}
      viewMode={true}
      isPublicMode={isPublicMode}
      // NEW: pass frozen chart config + report metadata in report mode
      frozenChartConfig={
        isReportMode && frozenChartConfigs
          ? frozenChartConfigs[String(component.config?.chartId)]
          : undefined
      }
      reportMetadata={isReportMode ? reportMetadata : undefined}
    />
  );
```

Also: hide edit/share/delete buttons when `isReportMode` (same pattern as `isPublicMode`).

#### `chart-element-view.tsx` — use frozen config instead of API call

**What it does today** (line 188):
```typescript
const { data: chart } = useChart(isPublicMode ? null : chartId);
```
It always fetches the chart config from `GET /api/charts/{chartId}/`. Then uses `chart.schema_name`, `chart.table_name`, `chart.extra_config` for data queries.

**What changes**: Accept `frozenChartConfig` prop. When provided, skip the API call:

```typescript
interface ChartElementViewProps {
  // ... existing props ...
  frozenChartConfig?: {
    id: number;
    title: string;
    chart_type: string;
    schema_name: string;
    table_name: string;
    extra_config: Record<string, any>;
    description?: string;
  };
  reportMetadata?: {
    date_column: { schema_name: string; table_name: string; column_name: string };
    period_start?: string;
    period_end: string;
  };
}

// Skip useChart() when we have frozen config:
const { data: chartFromApi } = useChart(
  frozenChartConfig || isPublicMode ? null : chartId
);
const chart = frozenChartConfig || chartFromApi;

// For data fetching in report mode, use POST /api/charts/chart-data/
// instead of GET /api/charts/{chartId}/data/ (which would 404 if chart deleted).
// Build ChartDataPayload from the frozen config + inject report date filter:
const fetchChartData = async () => {
  if (frozenChartConfig) {
    // Build the forced date range filter from report metadata
    const reportDateFilter = reportMetadata ? {
      filter_id: 'report_period',
      column: reportMetadata.date_column.column_name,
      type: 'datetime',
      value: {
        start_date: reportMetadata.period_start || undefined,
        end_date: reportMetadata.period_end,
      },
      settings: {},
      schema_name: reportMetadata.date_column.schema_name,
      table_name: reportMetadata.date_column.table_name,
    } : undefined;

    // POST with frozen config + forced date filter
    return apiPost('/api/charts/chart-data/', {
      chart_type: frozenChartConfig.chart_type,
      schema_name: frozenChartConfig.schema_name,
      table_name: frozenChartConfig.table_name,
      ...frozenChartConfig.extra_config,  // x_axis, y_axis, metrics, etc.
      dashboard_filters: reportDateFilter ? [reportDateFilter] : undefined,
    });
  }
  // Normal mode — use GET endpoint
  return apiGet(`/api/charts/${chartId}/data/?...`);
};
```

This follows the **exact same pattern** the component already uses for public mode (line 188: `useChart(isPublicMode ? null : chartId)`).

**Important**: In report mode, chart data is fetched via `POST /api/charts/chart-data/` which accepts config inline and calls the same `generate_chart_data_and_config()` function. This means snapshots work even if the original chart has been deleted — no dependency on the Chart table at all.

#### `unified-filters-panel.tsx` — disable forced filters

If a filter has `settings.is_forced === true`, disable the widget (grey it out, remove clear/change controls). ~3 lines.

---

## 6. Open Questions (to resolve during implementation)

### Q1: Chart data after chart deletion — RESOLVED

~~When a chart is deleted, `GET /api/charts/{chartId}/data/` returns 404.~~

**Resolved**: In report mode, the frontend uses `POST /api/charts/chart-data/` which accepts chart config inline (`schema_name`, `table_name`, `extra_config`) and calls the same `generate_chart_data_and_config()`. No Chart DB lookup needed. Snapshots work even after chart deletion.

### Q2: How the date range filter is applied to charts — RESOLVED

**Resolved**: The user selects a datetime field (from the dashboard's datetime filters) at snapshot creation time. This is stored as `date_column` on the snapshot (`{schema_name, table_name, column_name}`).

At view time, the frontend injects a **forced datetime dashboard filter** into every `POST /api/charts/chart-data/` request:

```json
{
  "dashboard_filters": [{
    "filter_id": "report_period",
    "column": "order_date",
    "type": "datetime",
    "value": { "start_date": "2024-01-01", "end_date": "2024-12-31" },
    "settings": {},
    "schema_name": "public",
    "table_name": "sales"
  }]
}
```

The backend's existing `apply_dashboard_filters()` function handles this natively:
- It validates that the filter column exists in the chart's target table
- If the column doesn't exist in a particular chart's table → filter is skipped (chart data is unfiltered)
- Datetime filters generate: `column >= start_date` AND `column <= end_date + "T23:59:59"`
- If `start_date` is absent (period_start was NULL), only the upper bound is applied

**No backend changes needed** — the existing filter infrastructure handles everything. The only addition is that the frontend builds and injects this forced filter from report metadata.

### Migration

```bash
cd DDP_backend
python manage.py makemigrations
python manage.py migrate
```

No existing tables are modified. This is purely additive.

### Permissions

Reuses existing dashboard permissions (`can_view_dashboards`, `can_create_dashboards`, etc.). No seed data changes needed.

### Dependencies

- `python-dateutil` — check if already in `DDP_backend/pyproject.toml`. Only needed if we add period calculation helpers later.
- No new frontend packages.
