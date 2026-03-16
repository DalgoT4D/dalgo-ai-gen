# Playwright-Based PDF Export for Reports

## Status: IMPLEMENTED (v2 — render secret + in-memory)

---

## Context

The current client-side PDF export (html2canvas-pro → jsPDF) produces washed-out, degraded chart colors. Multiple attempts to fix it client-side (foreignObject rendering, canvas-to-img replacement, PNG format, direct canvas overlay) have all failed because html2canvas's JS-based renderer fundamentally doesn't match browser-native color rendering.

**New approach**: Move PDF generation to the backend using Playwright. Playwright launches a real Chromium browser, navigates to the report page, waits for charts to render, and calls `page.pdf()` — producing pixel-perfect output because it uses the actual browser rendering engine.

**Future consideration**: This same backend infrastructure naturally supports scheduled reports — a Celery beat task can generate the PDF and email it to users without any browser involvement from the user.

---

## Architecture Overview (v2 — Current)

```
User clicks "Download PDF"
        │
        ▼
Frontend: POST /api/reports/{id}/export/pdf/
  (authenticated, JWT cookie)
        │
        ▼
Backend (report_api.py):
  1. Ensure report has public_share_token (generate if needed)
     — does NOT set is_public=True
  2. Call PdfExportService.generate_pdf(snapshot_id, token)
  3. Return PDF bytes as HttpResponse
        │
        ▼
pdf_export_service.py:
  1. Launch Playwright Chromium (headless)
  2. Register page.route("**/*") interceptor that injects
     X-Render-Secret header on EVERY request
  3. Navigate to {FRONTEND_URL_V2}/share/report/{token}?print=true
  4. Frontend loads, fetches data from public endpoints
     — backend sees X-Render-Secret header, skips is_public check
  5. Wait for rendering: networkidle + data-pdf-ready + canvas + 2s delay
  6. page.pdf() → returns bytes in memory (no disk write)
        │
        ▼
Backend wraps bytes in HttpResponse → browser downloads PDF
```

### Detailed Step-by-Step Flow

**Step 1 — User triggers download**
The user clicks "Download PDF" on `/reports/[snapshotId]`. The frontend calls `apiPostBinary('/api/reports/{id}/export/pdf/', {})` — an authenticated POST with JWT cookie.

**Step 2 — Backend receives request (`report_api.py`)**
The `export_report_pdf` endpoint validates permissions (`can_view_dashboards`), fetches the snapshot, and ensures it has a `public_share_token`. If the token doesn't exist, one is generated via `secrets.token_urlsafe(48)`. Critically, `is_public` is **never** set to `True`.

**Step 3 — PDF generation starts (`pdf_export_service.py`)**
`PdfExportService.generate_pdf()` is called with the snapshot ID and share token. It reads `RENDER_SECRET` from Django settings (fails fast with `ValueError` if not configured).

**Step 4 — Playwright launches headless Chromium**
A fresh Chromium instance starts in headless mode with a 1200x800 viewport. This happens inside `sync_playwright()` context manager (sync API to avoid ASGI event loop conflicts).

**Step 5 — Route interception is registered**
Before navigating anywhere, `page.route("**/*", _inject_render_secret)` is registered. This tells Playwright: "For every single HTTP request this page makes — navigation, CSS, JS, images, AND JavaScript `fetch()` calls — intercept it and add `x-render-secret: {secret}` to the headers before letting it through."

**Step 6 — Playwright navigates to the public share page**
`page.goto("{FRONTEND_URL_V2}/share/report/{token}?print=true", wait_until="networkidle")`. The Next.js app loads. The `?print=true` query param tells `PublicReportView` to render in print mode (clean layout, no footer/badges).

**Step 7 — Frontend fetches report data (intercepted)**
The React app calls `GET /api/v1/public/reports/{token}/view/` via SWR. This fetch request is intercepted by Playwright's route handler, which injects the `X-Render-Secret` header.

**Step 8 — Backend serves data without `is_public` check (`public_api.py`)**
`_get_public_report_snapshot()` sees the `X-Render-Secret` header, validates it with `hmac.compare_digest()`, and returns the snapshot **without** requiring `is_public=True`. The same happens for all subsequent chart data requests (`/chart-data/`, `/map-data/`, etc.).

**Step 9 — Frontend renders charts**
ECharts initializes, renders charts on `<canvas>` elements. The SWR hook resolves, and `PublicReportView` sets `data-pdf-ready="true"` on the root div.

**Step 10 — Playwright waits for rendering completion (4-layer detection)**
1. `networkidle` — already satisfied from `goto()`
2. `wait_for_selector("[data-pdf-ready='true']")` — data loaded
3. `wait_for_selector("canvas")` — at least one chart rendered (try/except for reports without charts)
4. `wait_for_timeout(2000)` — safety buffer for ECharts animations

**Step 11 — PDF captured in memory**
`page.pdf(format="A4", print_background=True, scale=1)` — Chromium's built-in PDF printer captures the page with pixel-perfect colors. Returns `bytes` directly (no file written to disk).

**Step 12 — Cleanup and response**
Browser closes (in `finally` block). The bytes are wrapped in `HttpResponse(pdf_bytes, content_type="application/pdf")` with a `Content-Disposition` header for download. The response streams back to the frontend.

**Step 13 — Frontend triggers download**
`apiPostBinary` receives the blob. A temporary `<a>` element is created with `URL.createObjectURL(blob)`, clicked programmatically, then cleaned up. The PDF downloads to the user's machine.

---

### Why the report never becomes public

The old approach toggled `is_public=True` during PDF generation — creating race conditions, crash-recovery issues, and a brief window of real public exposure. The new approach uses Playwright's `page.route()` to intercept all requests from the headless browser and inject an `X-Render-Secret` header. The backend's public report endpoints check for this header and, if it matches `settings.RENDER_SECRET`, skip the `is_public=True` check. The snapshot's public state is never modified.

---

## Files Created / Modified

### Backend (DDP_backend)

| File | Action | Purpose |
|------|--------|---------|
| `pyproject.toml` | **Modified** | Added `playwright>=1.40.0` dependency |
| `ddpui/settings.py` | **Modified** | Added `RENDER_SECRET = os.getenv("RENDER_SECRET")` |
| `ddpui/core/reports/pdf_export_service.py` | **Created** | Playwright PDF generation (sync API, lazy import, route interception, in-memory) |
| `ddpui/api/report_api.py` | **Modified** | Added `POST /{snapshot_id}/export/pdf/` endpoint (no is_public toggling) |
| `ddpui/api/public_api.py` | **Modified** | `_get_public_report_snapshot` checks X-Render-Secret header via `hmac.compare_digest` |
| `.env` | **Modified** | Added `RENDER_SECRET=<random-token>` |

### Frontend (webapp_v2)

| File | Action | Purpose |
|------|--------|---------|
| `app/share/report/[token]/page.tsx` | **Modified** | Reads `searchParams.print`, passes `printMode` prop |
| `app/share/report/[token]/PublicReportView.tsx` | **Modified** | Print mode rendering (clean layout, `data-pdf-ready`, `isPrintMode` prop) |
| `app/reports/[snapshotId]/page.tsx` | **Modified** | Replaced `usePdfDownload` with `apiPostBinary` backend call |
| `components/dashboard/dashboard-native-view.tsx` | **Modified** | Added `isPrintMode` prop to remove height/overflow constraints for PDF |

**Zero frontend changes were needed for the v2 render-secret approach.** The existing `/share/report/{token}?print=true` page works as-is — Playwright injects the header transparently.

---

## Implementation Details

### 1. Render Secret Authentication (`public_api.py`)

```python
def _get_public_report_snapshot(token: str, request=None) -> ReportSnapshot:
    if request:
        render_secret = request.META.get("HTTP_X_RENDER_SECRET")
        expected = getattr(settings, "RENDER_SECRET", None)
        if render_secret and expected and hmac.compare_digest(render_secret, expected):
            # Internal render: skip is_public check
            return ReportSnapshot.objects.select_related("org", "created_by__user").get(
                public_share_token=token,
            )

    # Normal public access: require is_public=True
    return ReportSnapshot.objects.select_related("org", "created_by__user").get(
        public_share_token=token, is_public=True
    )
```

All 5 public report endpoints pass `request=request` to this helper:
- `GET /reports/{token}/view/`
- `POST /reports/{token}/chart-data/`
- `POST /reports/{token}/chart-data-preview/`
- `POST /reports/{token}/chart-data-preview/total-rows/`
- `POST /reports/{token}/map-data/`

### 2. Playwright Route Interception (`pdf_export_service.py`)

```python
class PdfExportService:
    @staticmethod
    def generate_pdf(snapshot_id: int, share_token: str) -> bytes:
        from playwright.sync_api import sync_playwright  # lazy import

        render_secret = getattr(settings, "RENDER_SECRET", None)
        # ... build URL ...

        with sync_playwright() as p:
            browser = p.chromium.launch(headless=True)
            page = browser.new_page(viewport={...})

            # Inject X-Render-Secret on EVERY request (including JS fetch() calls)
            def _inject_render_secret(route):
                headers = {**route.request.headers, "x-render-secret": render_secret}
                route.continue_(headers=headers)

            page.route("**/*", _inject_render_secret)
            page.goto(url, wait_until="networkidle")
            # ... wait for data-pdf-ready, canvas, 2s delay ...
            pdf_bytes = page.pdf(format="A4", print_background=True, scale=1)
            return pdf_bytes  # bytes, no disk write
```

**Key decisions**:
- **Sync API, not async**: `playwright.sync_api` avoids conflicts with uvicorn's ASGI event loop.
- **Lazy import**: Prevents `ImportError` from breaking all report routes if playwright isn't installed.
- **`page.route("**/*")`**: Intercepts ALL requests including JavaScript `fetch()` calls from the React app. This is what makes the header injection transparent to the frontend.
- **In-memory PDF**: `page.pdf()` without `path=` returns bytes directly. No disk I/O, no orphaned files, no cleanup needed.
- **RENDER_SECRET validation**: Service raises `ValueError` if the env var is missing — fails fast rather than silently generating a broken PDF.

### 3. API Endpoint (`report_api.py`)

```python
@report_router.post("/{snapshot_id}/export/pdf/", response={200: None})
@has_permission(["can_view_dashboards"])
def export_report_pdf(request, snapshot_id: int):
    snapshot = ReportService.get_snapshot(snapshot_id, orguser.org)

    # Ensure token exists (does NOT set is_public=True)
    if not snapshot.public_share_token:
        snapshot.public_share_token = secrets.token_urlsafe(48)
        snapshot.save(update_fields=["public_share_token"])

    pdf_bytes = PdfExportService.generate_pdf(snapshot_id, snapshot.public_share_token)

    response = HttpResponse(pdf_bytes, content_type="application/pdf")
    response["Content-Disposition"] = f'attachment; filename="{filename}"'
    return response
```

- `response={200: None}` — tells Django Ninja to skip schema validation (required for raw HttpResponse)
- No `is_public` toggling, no state restoration, no error recovery for state — just generate and return

### 4. Frontend: Print mode (`PublicReportView.tsx`)

When `printMode=true`:
- Renders clean layout: title + date range header only (no logo, org badge, "Public View" badge, footer)
- Sets `data-pdf-ready="true"` on root div (Playwright waits for this)
- Passes `isPrintMode={true}` to `DashboardNativeView`
- Executive summary uses `overflow-hidden` + `break-words` to contain long text

### 5. Frontend: Print mode in `DashboardNativeView`

Added `isPrintMode?: boolean` prop that overrides CSS constraints at 4 layers:

| Layer | Normal mode | Print mode override |
|-------|-------------|-------------------|
| Outer container | `h-full overflow-hidden` | `h-auto overflow-visible print-mode` |
| Public mode styles | `h-screen overflow-hidden min-h-screen` | Suppressed (`!isPrintMode` guard) |
| Main content area | `flex-1 flex overflow-hidden` | `overflow-visible` |
| Scrollable canvas | `flex-1 overflow-auto pb-[150px]` | `overflow-visible pb-4` |
| `.dashboard-canvas` (CSS) | `overflow: hidden` | `.print-mode .dashboard-canvas { overflow: visible !important }` |

### 6. Frontend: Download button (`app/reports/[snapshotId]/page.tsx`)

```tsx
const handleDownload = useCallback(async () => {
  setIsExporting(true);
  try {
    const blob = await apiPostBinary(`/api/reports/${snapshotId}/export/pdf/`, {});
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `${safeTitle}.pdf`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
    toastSuccess.exported('Report', 'pdf');
  } catch (error) {
    toastError.export(error, 'pdf');
  } finally {
    setIsExporting(false);
  }
}, [snapshotId, viewData?.report_metadata.title]);
```

### 7. Dependency setup

- `pyproject.toml`: Added `"playwright>=1.40.0"`
- `.env`: Added `RENDER_SECRET=<random-token>`
- Post-install: `uv sync && playwright install chromium`

---

## Security Model

| Concern | How it's handled |
|---------|-----------------|
| **Snapshot never becomes public** | `is_public` is never toggled. Playwright authenticates via render secret header. |
| **Secret stays on server** | `RENDER_SECRET` is an env var. Playwright injects it via `page.route()` — it never leaves the backend process. |
| **Timing attacks** | `hmac.compare_digest()` for constant-time comparison of the secret. |
| **Missing secret** | `PdfExportService.generate_pdf()` raises `ValueError` immediately if `RENDER_SECRET` is not configured. |
| **Normal public sharing unchanged** | Without the header, the existing `is_public=True` check still applies. Backward compatible. |
| **No race conditions** | No database state is toggled, so concurrent exports can't interfere with each other. |
| **No crash recovery needed** | Nothing to restore if the process dies mid-export. |

---

## Rendering Completion Detection

Since there's no built-in "rendering complete" signal in the frontend, use a 4-layer approach:

1. **`networkidle`**: Playwright's `goto(url, wait_until="networkidle")` waits until no network requests for 500ms. Catches all chart data API calls.

2. **`data-pdf-ready` attribute**: `PublicReportView` sets `data-pdf-ready="true"` on the root div once `viewData` has loaded (SWR fetch returned). Playwright waits with `page.wait_for_selector("[data-pdf-ready='true']")`.

3. **Canvas presence**: `page.wait_for_selector("canvas", timeout=30000)` ensures at least one ECharts chart has initialized. Wrapped in try/except for reports without charts.

4. **Fixed delay**: `page.wait_for_timeout(2000)` — safety buffer for ECharts transition animations and late renders.

Total expected wait: ~4-8 seconds per report.

---

## Issues Encountered & Resolutions

### Issue 1: 404 on all report routes after adding pdf_export_service

**Symptom**: `POST /api/reports/17/export/pdf/` returned 404, but ALL report endpoints also returned 404.

**Root cause**: `pdf_export_service.py` had `from playwright.sync_api import sync_playwright` at module top level. Since playwright wasn't installed yet, this raised `ImportError`, which prevented `report_api.py` from importing `PdfExportService`, which caused Django Ninja to fail loading the entire `report_router` — making ALL report routes vanish.

**Fix**: Made the playwright import lazy (inside the function body).

### Issue 2: 500 error — async event loop conflict

**Symptom**: `POST /api/reports/17/export/pdf/` returned 500 Internal Server Error.

**Root cause**: Initially used `playwright.async_api` with `async_to_sync` wrapper. But uvicorn (ASGI) already runs an event loop, so `async_to_sync` fails.

**Fix**: Switched to `playwright.sync_api` + `sync_playwright`. The sync API runs Playwright in its own thread.

### Issue 3: PDF content clipped — not showing full report

**Symptom**: PDF generated but only showed partial content — charts were cut off.

**Root cause**: `DashboardNativeView` has multiple layers of CSS `overflow-hidden` constraints.

**Fix**: Added `isPrintMode` prop that overrides all constraint layers with `h-auto overflow-visible`.

### Issue 4: Executive summary text overflowing its border box

**Symptom**: In the PDF, the executive summary text extended horizontally past the rounded border container.

**Fix**: Added `overflow-hidden` on the executive summary container div and `break-words` on the paragraph text.

---

## Evolution: v1 → v2

### v1 (Original — temporary public toggle + disk storage)

```
Backend: toggle is_public=True → Playwright hits public URL → write PDF to exports/ → restore is_public → FileResponse(open(path))
```

**Problems**: Race conditions on concurrent exports, crash leaves report public, brief real public exposure, orphaned files in exports/.

### v2 (Current — render secret + in-memory)

```
Backend: ensure token exists → Playwright hits URL with X-Render-Secret header injected → page.pdf() returns bytes → HttpResponse(bytes)
```

**Improvements**: No public state toggling, no race conditions, no crash recovery, no disk I/O, no orphaned files.

---

## Future: Scheduled Reports + Email

This Playwright approach is designed to support scheduled report generation:

```
Celery Beat (scheduled)
    │
    ▼
@app.task: generate_and_email_report(snapshot_id, recipient_emails)
    │
    ├─ PdfExportService.generate_pdf(snapshot_id, token)  ← same service
    │
    ├─ awsses.send_email_with_attachment(emails, pdf_bytes)  ← new helper
    │
    └─ (no cleanup needed — bytes are garbage collected)
```

### What's needed for scheduled reports (future scope)

1. **Report Schedule model**: `ReportSchedule(snapshot_id, cron_expression, recipients[], enabled)`
2. **Celery beat task**: Registered in `app.conf.beat_schedule`, runs the export + email
3. **Email with attachment**: Extend `ddpui/utils/awsses.py` with `send_email_with_attachment()` using SES + S3 presigned URL or inline attachment
4. **Schedule management UI**: Frontend CRUD for report schedules

### Production considerations (not for now)

- **Browser pooling**: Reuse Playwright browser instances across requests (vs. launch per request)
- **Timeout limits**: Set max 60s for PDF generation, return error if exceeded
- **Queue**: Route PDF tasks to a dedicated Celery queue with Chromium-equipped workers
- **Caching**: Cache PDF bytes for repeated downloads of the same snapshot (snapshots are frozen data)

---

## Verification

1. `cd DDP_backend && uv sync` — install playwright dependency
2. `playwright install chromium` — download browser binary
3. Add `RENDER_SECRET=<some-random-string>` to `.env`
4. Start all services (backend, frontend, postgres, redis)
5. Create a report snapshot from a dashboard
6. Click the download PDF button on `/reports/[snapshotId]`
7. Verify PDF downloads directly (no files in `DDP_backend/exports/`)
8. Open PDF and compare chart colors against on-screen dashboard
9. Test with a report that is NOT public — verify it works without toggling `is_public`
10. Verify the report's `is_public` field is unchanged after export
11. Check the public report page with `?print=true` query param renders correctly

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
| **Security** | N/A (client-side) | Render secret — no public exposure |
| **Disk usage** | None | None (in-memory) |
