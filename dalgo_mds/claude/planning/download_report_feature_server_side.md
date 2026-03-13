# Playwright-Based PDF Export for Reports

## Context

The current client-side PDF export (html2canvas-pro → jsPDF) produces washed-out, degraded chart colors. Multiple attempts to fix it client-side (foreignObject rendering, canvas-to-img replacement, PNG format, direct canvas overlay) have all failed because html2canvas's JS-based renderer fundamentally doesn't match browser-native color rendering.

**New approach**: Move PDF generation to the backend using Playwright. Playwright launches a real Chromium browser, navigates to the report page, waits for charts to render, and calls `page.pdf()` — producing pixel-perfect output because it uses the actual browser rendering engine.

**Future consideration**: This same backend infrastructure naturally supports scheduled reports — a Celery beat task can generate the PDF and email it to users without any browser involvement from the user.

---

## Architecture Overview

```
User clicks "Download PDF"
        │
        ▼
Frontend: POST /api/reports/{id}/export/pdf/
        │
        ▼
Backend (report_api.py):
  1. Ensure report has public_share_token (generate if needed)
  2. Temporarily set is_public=True (if not already)
  3. Call pdf_export_service.generate_pdf()
  4. Restore original is_public state
  5. Return PDF file as download
        │
        ▼
pdf_export_service.py:
  1. Launch Playwright Chromium (headless)
  2. Navigate to {FRONTEND_URL_V2}/share/report/{token}?print=true
  3. Wait for rendering: networkidle + canvas elements + fixed delay
  4. page.pdf() → save to DDP_backend/exports/{snapshot_id}.pdf
  5. Return file path
        │
        ▼
Frontend receives file blob → browser downloads PDF
```

---

## Files to Create / Modify

### Backend (DDP_backend)

| File | Action | Purpose |
|------|--------|---------|
| `pyproject.toml` | **Modify** | Add `playwright` dependency |
| `ddpui/core/reports/pdf_export_service.py` | **Create** | Playwright PDF generation logic |
| `ddpui/api/report_api.py` | **Modify** | Add `POST /{snapshot_id}/export/pdf/` endpoint |
| `ddpui/schemas/report_schema.py` | **Modify** | Add `PdfExportResponse` schema |

### Frontend (webapp_v2)

| File | Action | Purpose |
|------|--------|---------|
| `app/share/report/[token]/page.tsx` | **Modify** | Pass `print` query param to PublicReportView |
| `app/share/report/[token]/PublicReportView.tsx` | **Modify** | Support `?print=true` mode (hide chrome) |
| `app/reports/[snapshotId]/page.tsx` | **Modify** | Replace `usePdfDownload` with backend API call |
| `hooks/usePdfDownload.ts` | **Keep as-is** | Will be deprecated but not deleted yet |

---

## Implementation Details

### 1. Backend: `ddpui/core/reports/pdf_export_service.py` (new file)

```python
class PdfExportService:
    @staticmethod
    async def generate_pdf(snapshot_id: int, share_token: str) -> str:
        """
        Launch Playwright, navigate to public report page in print mode,
        wait for charts to render, generate PDF, save locally.
        Returns the file path of the saved PDF.
        """
```

**Playwright wait strategy** (critical for chart rendering):
1. Navigate to `{FRONTEND_URL_V2}/share/report/{token}?print=true`
2. `page.wait_for_load_state("networkidle")` — wait until no network requests for 500ms (chart data APIs finish)
3. `page.wait_for_selector("canvas")` — at least one ECharts canvas exists
4. `page.wait_for_timeout(2000)` — additional 2s buffer for ECharts animation/rendering to complete
5. `page.pdf(format="A4", print_background=True, scale=1)` — generate PDF

**PDF save location** (for testing): `DDP_backend/exports/report_{snapshot_id}_{timestamp}.pdf`

**Key details**:
- Use `playwright.async_api` with `async_to_sync` wrapper for Django
- Viewport: `1200x800` (matches `PDF_CLONE_WIDTH_PX`)
- Set `print_background=True` so chart backgrounds render
- The Playwright browser instance is created per-request for now (can pool later for production)

### 2. Backend: API endpoint in `report_api.py`

```python
@report_router.post("/{snapshot_id}/export/pdf/")
@has_permission(["can_view_dashboards"])
def export_report_pdf(request, snapshot_id: int):
    """Generate PDF of report via Playwright and return as download"""
```

**Temporary sharing flow**:
1. Fetch snapshot for org
2. Save original `is_public` and `public_share_token` values
3. If no token exists, generate one: `secrets.token_urlsafe(48)`
4. Set `is_public=True`, save
5. Call `PdfExportService.generate_pdf(snapshot_id, token)`
6. Restore original `is_public` state, save
7. Return PDF as `FileResponse` with `Content-Disposition: attachment`

The token is NOT removed after export — only `is_public` is restored. This avoids regenerating tokens repeatedly.

### 3. Frontend: Print mode for public report page

**`app/share/report/[token]/page.tsx`** — read `searchParams.print` and pass to component:
```tsx
<PublicReportView token={token} printMode={searchParams?.print === 'true'} />
```

**`PublicReportView.tsx`** — when `printMode=true`:
- Hide the `<header>` (logo, org name, "Public View" badge, date range)
- Hide the `<PoweredByDalgoFooter />`
- Add a clean report header inside the content area: title + date range (similar to current authenticated report header captured by `usePdfDownload`)
- Set `min-h-screen` → `min-h-0` (no unnecessary whitespace)
- Add `data-pdf-ready="true"` attribute on the root div after data loads (Playwright can wait for this)

### 4. Frontend: Replace download button handler

**`app/reports/[snapshotId]/page.tsx`**:
- Replace `usePdfDownload` with a simple async function:
```tsx
const handleDownload = async () => {
  setIsExporting(true);
  try {
    const blob = await apiPost(`/api/reports/${snapshotId}/export/pdf/`, {}, { responseType: 'blob' });
    saveAs(blob, `${sanitizedTitle}.pdf`);
    toastSuccess.exported('Report', 'pdf');
  } catch (error) {
    toastError.export(error, 'pdf');
  } finally {
    setIsExporting(false);
  }
};
```

Need to check how `lib/api.ts` handles blob responses — may need to add a `apiDownload` helper or use raw `fetch` with `credentials: 'include'`.

### 5. Dependency setup

**`pyproject.toml`**: Add `playwright` to dependencies.

**Post-install**: Run `playwright install chromium` to download the browser binary. This needs to be documented and run once after `uv sync`.

---

## Rendering Completion Detection (Refined Strategy)

Since there's no built-in "rendering complete" signal in the frontend, use a layered approach:

1. **`data-pdf-ready` attribute** (new): The `PublicReportView` component sets `data-pdf-ready="true"` on the root div once `viewData` has loaded (i.e., the SWR fetch returned). Playwright waits for this first.

2. **`networkidle`**: Playwright waits for no network activity for 500ms. This catches all chart data API calls (`/chart-data/`, `/map-data/`, etc.).

3. **Canvas presence**: `page.wait_for_selector('canvas', { timeout: 30000 })` ensures at least one ECharts chart has initialized.

4. **Fixed delay**: `page.wait_for_timeout(2000)` — safety buffer for ECharts transition animations and any late renders.

Total expected wait: ~4-6 seconds per report.

---

## Key Files Already Read (state as of planning session)

### Backend files understood:
- `DDP_backend/pyproject.toml` — dependency list, need to add playwright
- `DDP_backend/ddpui/api/report_api.py` — existing report endpoints, sharing logic pattern
- `DDP_backend/ddpui/schemas/report_schema.py` — existing schemas
- `DDP_backend/ddpui/routes.py` — report_router mounted at `/api/reports/`
- `DDP_backend/ddpui/core/reports/` — has `__init__.py`, `exceptions.py`, `report_service.py`

### Frontend files understood:
- `webapp_v2/app/share/report/[token]/page.tsx` — simple Suspense wrapper, passes token
- `webapp_v2/app/share/report/[token]/PublicReportView.tsx` — full public view with header/footer, separate mobile/desktop layouts
- `webapp_v2/app/reports/[snapshotId]/page.tsx` — uses `usePdfDownload` hook currently
- `webapp_v2/lib/api.ts` — has `apiPostBinary` helper that returns blob (perfect for PDF download)

### Key patterns observed:
- `apiPostBinary` already exists in `lib/api.ts` — returns `response.blob()`, perfect for PDF download
- Report sharing already uses `secrets.token_urlsafe(48)` pattern
- `PublicReportView` has separate mobile/desktop layouts in header
- The `page.tsx` for public report is an async server component that receives params
- Django Ninja `FileResponse` is the way to return file downloads
- Report router is at `/api/reports/` in routes.py

---

## Future: Scheduled Reports + Email

This Playwright approach is designed to support scheduled report generation:

### How it fits

```
Celery Beat (scheduled)
    │
    ▼
@app.task: generate_and_email_report(snapshot_id, recipient_emails)
    │
    ├─ PdfExportService.generate_pdf(snapshot_id, token)  ← same service
    │
    ├─ awsses.send_email_with_attachment(emails, pdf_path)  ← new helper
    │
    └─ Clean up PDF file
```

### What's needed for scheduled reports (future scope, not implementing now)

1. **Report Schedule model**: `ReportSchedule(snapshot_id, cron_expression, recipients[], enabled)`
2. **Celery beat task**: Registered in `app.conf.beat_schedule`, runs the export + email
3. **Email with attachment**: Extend `ddpui/utils/awsses.py` with `send_email_with_attachment()` using SES + S3 presigned URL or inline attachment
4. **Schedule management UI**: Frontend CRUD for report schedules

### Why Playwright is the right choice for this

- **No browser needed**: Celery worker runs headless Chromium server-side — no user interaction required
- **Exact rendering**: Same browser engine = same visual output as the dashboard
- **Scalable**: Can run on a dedicated worker node with Chromium installed
- **Reusable**: Same `PdfExportService.generate_pdf()` works for both on-demand and scheduled exports
- **Alternative considered**: Client-side export requires a browser session — impossible for scheduled/automated reports

### Production considerations (not for now)

- **Browser pooling**: Reuse Playwright browser instances across requests (vs. launch per request)
- **Timeout limits**: Set max 60s for PDF generation, return error if exceeded
- **Storage**: Move from local filesystem to S3 for generated PDFs
- **Queue**: Route PDF tasks to a dedicated Celery queue with Chromium-equipped workers
- **Caching**: Cache PDFs for repeated downloads of the same snapshot (snapshots are frozen data)

---

## Verification

1. `cd DDP_backend && uv sync` — install playwright dependency
2. `playwright install chromium` — download browser binary
3. Start all services (backend, frontend, postgres, redis)
4. Create a report snapshot from a dashboard
5. Click the download PDF button on `/reports/[snapshotId]`
6. Check `DDP_backend/exports/` for the generated PDF
7. Open PDF and compare chart colors against on-screen dashboard
8. Verify: bar chart, pie chart, line chart, map chart, table chart, number chart, executive summary, header text
9. Test with a report that is NOT currently public (verify temporary sharing works correctly and is_public is restored)
10. Check the public report page with `?print=true` query param renders correctly (no header/footer chrome)

---

## Trade-offs

| | Client-side (html2canvas) | Server-side (Playwright) |
|---|---|---|
| **Color accuracy** | Degraded (JS renderer) | Pixel-perfect (real browser) |
| **Dependencies** | None (already installed) | Playwright + Chromium (~200MB) |
| **Server resources** | None | Headless browser per export |
| **Scheduled exports** | Impossible | Natural fit |
| **Export time** | ~2-3s | ~5-8s (browser launch + render) |
| **Offline/no-backend** | Works | Requires running backend + frontend |
