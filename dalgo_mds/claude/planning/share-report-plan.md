# Plan: Share Report via Public Link

Follow the exact same pattern as dashboard sharing. No extra abstractions.

## Files to Modify

### Backend
1. `DDP_backend/ddpui/models/report.py` — Add sharing fields (same as Dashboard)
2. `DDP_backend/ddpui/schemas/report_schema.py` — Add sharing schemas (same as dashboard_schema.py)
3. `DDP_backend/ddpui/api/report_api.py` — Add share toggle + status endpoints (same as dashboard_native_api.py)
4. `DDP_backend/ddpui/api/public_api.py` — Add public report view + chart data endpoints
5. New migration via `makemigrations`

### Frontend
6. `webapp_v2/hooks/api/useReports.ts` — Add sharing API functions + public report hook
7. `webapp_v2/components/reports/ReportShareModal.tsx` — New file, copy of ShareModal.tsx adapted for reports
8. `webapp_v2/app/reports/[snapshotId]/page.tsx` — Wire Share button to modal
9. `webapp_v2/app/share/report/[token]/page.tsx` — New, copy of share/dashboard/[token]/page.tsx
10. `webapp_v2/app/share/report/[token]/PublicReportView.tsx` — New, copy of PublicDashboardView.tsx adapted for reports
11. `webapp_v2/components/dashboard/chart-element-view.tsx` — Support public+report combined mode
12. `webapp_v2/middleware.ts` — Allow `/share/report/` routes

---

## Step 1: Model — Add sharing fields to ReportSnapshot

**File**: `DDP_backend/ddpui/models/report.py`

Add exact same fields as `Dashboard` model (`models/dashboard.py:85-102`):

```python
# Public sharing (same as Dashboard)
is_public = models.BooleanField(default=False)
public_share_token = models.CharField(max_length=64, unique=True, null=True, blank=True, db_index=True)
public_shared_at = models.DateTimeField(null=True, blank=True)
public_disabled_at = models.DateTimeField(null=True, blank=True)
public_access_count = models.IntegerField(default=0)
last_public_accessed = models.DateTimeField(null=True, blank=True)
```

Then run `python manage.py makemigrations ddpui`.

---

## Step 2: Schemas — Add sharing schemas

**File**: `DDP_backend/ddpui/schemas/report_schema.py`

Copy exact same schemas from `schemas/dashboard_schema.py:140-162`:

```python
class ReportShareToggle(Schema):
    is_public: bool

class ReportShareResponse(Schema):
    is_public: bool
    public_url: Optional[str] = None
    public_share_token: Optional[str] = None
    message: str

class ReportShareStatus(Schema):
    is_public: bool
    public_url: Optional[str] = None
    public_access_count: int
    last_public_accessed: Optional[datetime] = None
    public_shared_at: Optional[datetime] = None
```

---

## Step 3: API — Add share endpoints to report_api.py

**File**: `DDP_backend/ddpui/api/report_api.py`

Copy the exact pattern from `dashboard_native_api.py:438-531`:

```python
@report_router.put("/{snapshot_id}/share/", response=ReportShareResponse)
@has_permission(["can_share_dashboards"])
def toggle_report_sharing(request, snapshot_id: int, payload: ReportShareToggle):
    # Get snapshot, verify ownership (created_by == orguser)
    # Same logic as toggle_dashboard_sharing:
    #   - Generate token via secrets.token_urlsafe(48) if not exists
    #   - Set public_shared_at / public_disabled_at
    #   - Build public_url = f"{frontend_url}/share/report/{token}"

@report_router.get("/{snapshot_id}/share/", response=ReportShareStatus)
@has_permission(["can_view_dashboards"])
def get_report_sharing_status(request, snapshot_id: int):
    # Same logic as get_dashboard_sharing_status
```

---

## Step 4: Public API — Add public report endpoints

**File**: `DDP_backend/ddpui/api/public_api.py`

Same pattern as public dashboard endpoints but:
- Auth via report `public_share_token` instead of dashboard token
- Get org from `snapshot.org` instead of `dashboard.org`
- For chart data endpoints, no `chart_id` in URL — use POST body `chartDataPayload` with frozen config info

### 4a) View report (mirrors `get_public_dashboard`)
```python
@public_router.get("/reports/{token}/view/", response={200: dict, 404: PublicErrorResponse})
def get_public_report(request, token: str):
    # Lookup ReportSnapshot by public_share_token + is_public=True
    # Increment public_access_count via F-expression
    # Return same data as get_snapshot_view_data() + org_name
```

### 4b) Chart data (mirrors existing report POST at `/api/charts/chart-data/`)
```python
@public_router.post("/reports/{token}/chart-data/", response={200: dict, 404: PublicErrorResponse})
def get_public_report_chart_data(request, token: str):
    # Validate token → get snapshot → get org → get OrgWarehouse
    # Parse ChartDataPayload from body
    # Call generate_chart_data_and_config(payload, org_warehouse)
```

### 4c) Table chart data preview
```python
@public_router.post("/reports/{token}/chart-data-preview/", response={200: dict, 404: PublicErrorResponse})
def get_public_report_table_data(request, token: str, page: int = 0, limit: int = 100):
    # Validate token → get snapshot → get org
    # Call charts_service.get_chart_data_table_preview(org_warehouse, payload, page, limit)
```

### 4d) Table total rows
```python
@public_router.post("/reports/{token}/chart-data-preview/total-rows/", response={200: dict, 404: PublicErrorResponse})
def get_public_report_table_total_rows(request, token: str):
    # Validate token → get snapshot → get org
    # Call charts_service.get_chart_data_total_rows(org_warehouse, payload)
```

### 4e) Map data overlay
```python
@public_router.post("/reports/{token}/map-data/", response={200: dict, 404: PublicErrorResponse})
def get_public_report_map_data(request, token: str):
    # Same logic as get_public_map_data_overlay but auth via report token
```

**Note**: GeoJSON/regions endpoints are already public (no dashboard token needed), so they work for reports too.

---

## Step 5: Frontend hooks

**File**: `webapp_v2/hooks/api/useReports.ts`

Add same functions as `useDashboards.ts:232-250`:

```typescript
export async function updateReportSharing(snapshotId: number, data: { is_public: boolean }) {
  return apiPut(`/api/reports/${snapshotId}/share/`, data);
}

export async function getReportSharingStatus(snapshotId: number) {
  return apiGet(`/api/reports/${snapshotId}/share/`);
}

export function usePublicReport(token: string) {
  // Same pattern as usePublicDashboard — direct fetch, no auth headers
  const { data, error, mutate } = useSWR(
    token ? `/api/v1/public/reports/${token}/view/` : null,
    async (url: string) => {
      const backendUrl = process.env.NEXT_PUBLIC_BACKEND_URL || 'http://localhost:8002';
      const response = await fetch(`${backendUrl}${url}`);
      if (!response.ok) throw new Error('Report not found');
      return response.json();
    }
  );
  return { viewData: data, isLoading: !error && !data, isError: error, mutate };
}
```

---

## Step 6: ReportShareModal component

**File**: `webapp_v2/components/reports/ReportShareModal.tsx` (new)

Copy `components/dashboard/ShareModal.tsx` exactly, but:
- Accept `snapshotId: number` instead of `dashboard: Dashboard`
- Call `updateReportSharing(snapshotId, ...)` / `getReportSharingStatus(snapshotId)`
- Title: "Share Report"
- No `dashboard.is_public` / `dashboard.public_access_count` defaults — init both to `false` / `0`

---

## Step 7: Wire Share button

**File**: `webapp_v2/app/reports/[snapshotId]/page.tsx`

- Import `ReportShareModal`
- Add `const [shareModalOpen, setShareModalOpen] = useState(false)`
- Wire `<Button title="Share">` → `onClick={() => setShareModalOpen(true)}`
- Render `<ReportShareModal snapshotId={snapshotId} isOpen={shareModalOpen} onClose={() => setShareModalOpen(false)} />`

---

## Step 8: Public report page

**File**: `webapp_v2/app/share/report/[token]/page.tsx` (new)

Copy `app/share/dashboard/[token]/page.tsx` exactly, but render `PublicReportView` instead of `PublicDashboardView`. No embed options needed for v1.

---

## Step 9: Public report viewer

**File**: `webapp_v2/app/share/report/[token]/PublicReportView.tsx` (new)

Simplified version of `PublicDashboardView.tsx`:
- Uses `usePublicReport(token)` hook
- Header: Dalgo logo + report title + date range + "Public View" badge + org name
- Executive summary as read-only `<p>` (not editable textarea)
- Renders `DashboardNativeView` with:
  - `dashboardId={0}`
  - `dashboardData={viewData.dashboard_data}`
  - `isReportMode={true}`
  - `isPublicMode={true}`
  - `publicToken={token}`
  - `frozenChartConfigs={viewData.frozen_chart_configs}`
  - `hideHeader={true}`
- Error/loading states same as `PublicDashboardView.tsx`
- "Powered by Dalgo" footer

---

## Step 10: ChartElementView — Support public+report combined mode

**File**: `webapp_v2/components/dashboard/chart-element-view.tsx`

When BOTH `isPublicMode=true` AND `frozenChartConfig` is set, chart data fetching must use public report endpoints instead of authenticated ones.

Add condition: `const isPublicReport = isPublicMode && !!frozenChartConfig;`

### 10a) Regular charts (bar/line/pie/number) — line ~448-457
Currently when `frozenChartConfig` is set, uses `apiPost('/api/charts/chart-data/', chartDataPayload)`.
Change: when `isPublicReport`, use direct `fetch` to `${backendUrl}/api/v1/public/reports/${publicToken}/chart-data/` instead.

### 10b) Table charts — line ~466-511
Currently public mode uses dashboard-based URL. When `isPublicReport`, use:
- Data: POST to `/api/v1/public/reports/${publicToken}/chart-data-preview/`
- Total rows: POST to `/api/v1/public/reports/${publicToken}/chart-data-preview/total-rows/`

### 10c) Map charts — line ~807-832
Currently public mode uses dashboard-based URL. When `isPublicReport`, use:
- POST to `/api/v1/public/reports/${publicToken}/map-data/`

---

## Step 11: Middleware

**File**: `webapp_v2/middleware.ts`

Add `/share/report/` to the existing pattern:

```typescript
if (request.nextUrl.pathname.startsWith('/share/dashboard/') ||
    request.nextUrl.pathname.startsWith('/share/report/')) {
  // same header removal logic
}

export const config = {
  matcher: ['/share/dashboard/:path*', '/share/report/:path*'],
};
```

---

## Verification

1. `cd DDP_backend && python manage.py makemigrations ddpui && python manage.py migrate`
2. `cd DDP_backend && black .`
3. `cd webapp_v2 && npm run lint`
4. Manual test: Open report → Share → toggle ON → URL copied → open in incognito → renders
