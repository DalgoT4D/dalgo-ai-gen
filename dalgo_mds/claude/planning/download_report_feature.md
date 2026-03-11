# Report Download (PDF Export) — Implementation Document

## Overview

Report PDF download is fully **client-side** — no backend PDF generation. The user clicks a Download button on the report viewer page, and a PDF is generated in the browser from the rendered dashboard DOM.

All code lives in `app/reports/[snapshotId]/page.tsx`, inside the `handleDownload` function (lines 94–198).

---

## Why This Approach?

### The Problem

A report page contains multiple ECharts visualizations in a grid layout, a header with metadata, and an executive summary — all of which need to be captured as a single cohesive PDF. There are three broad approaches to generating a PDF from a web page:

### Approaches Considered

#### 1. Server-side PDF generation (Puppeteer / Playwright on backend)
- **How**: Backend spins up a headless browser, navigates to the report URL, calls `page.pdf()`.
- **Why we didn't use it**: Requires a headless browser installed on the server (Chromium ~400MB). Adds server load, cold start latency, and infrastructure complexity. The report page also requires authentication and organization context, so the server would need to replicate the user's session. For an NGO-focused platform, keeping infrastructure simple matters.

#### 2. Build the PDF programmatically (jsPDF text/shape APIs)
- **How**: Manually draw text, rectangles, and images into the PDF using jsPDF's drawing API.
- **Why we didn't use it**: The report layout is complex — responsive grid, ECharts with tooltips/legends, styled text, icons. Recreating all of this with low-level PDF drawing commands would be extremely fragile, hard to maintain, and would never look like the actual page. Any UI change would require matching PDF code changes.

#### 3. Screenshot the rendered DOM → embed in PDF (what we chose)
- **How**: Clone the rendered page DOM, screenshot it with `html2canvas`, embed the image in a PDF with `jsPDF`, download with `file-saver`.
- **Why this works**: The PDF looks exactly like what the user sees on screen because it IS a screenshot of what they see. No layout duplication, no server infrastructure. Works entirely in the browser with zero backend changes.

### Why Client-Side?

| Consideration | Client-side (chosen) | Server-side |
|---------------|---------------------|-------------|
| Infrastructure | Zero — runs in user's browser | Needs headless Chromium on server |
| Auth complexity | Already has the session/cookies | Must replicate user session |
| Maintenance | PDF automatically matches UI changes | Must keep server rendering in sync |
| Latency | ~2-3 seconds in browser | Network round trip + server render time |
| Server load | None | CPU/memory per export request |
| Offline support | Could work with cached data | Requires server connectivity |

### Why Each Step Is Necessary

**1. Off-screen DOM clone** — We can't screenshot the live page directly because html2canvas would capture it at the current scroll position and viewport size. The clone lets us control the exact width (1200px), background (white), and padding without disrupting what the user sees. It also avoids flicker.

**2. Manual canvas pixel copy** — This is the non-obvious critical step. ECharts renders to `<canvas>` elements. When the browser's `cloneNode(true)` copies a DOM tree, it copies the HTML structure but NOT the drawn pixels on `<canvas>` elements — the cloned canvases are blank. Without this step, the PDF would have empty white boxes where every chart should be.

**3. html2canvas at 3x scale** — `html2canvas` re-renders the DOM onto a single canvas. The 3x scale produces a high-resolution image (effectively 3600px wide from a 1200px layout), which keeps text and chart labels sharp in the final PDF. Lower scales produce blurry text on retina screens.

**4. Single custom-height PDF page** — We initially tried splitting into multiple A4 pages, but charts would get cut in half at page boundaries. A single page with custom height (A4 width × calculated content height) avoids this entirely. The tradeoff is a non-standard page size, but for a downloaded report this is acceptable — the content is intact.

**5. file-saver for download** — The browser's native download mechanisms (`<a download>` or `window.open`) have inconsistent behavior across browsers for blob URLs. `file-saver` normalizes this.

### Known Tradeoffs

- **Large PDFs**: A dashboard with many charts produces a tall single-page PDF. For very large dashboards, this could be a large file. Acceptable for typical 4-8 chart dashboards.
- **500ms wait**: The hardcoded 500ms delay before html2canvas capture is a heuristic. Too short = charts might not finish rendering in the clone. Too long = slower export. 500ms works reliably in testing.
- **No text selection in PDF**: Since the PDF contains a screenshot image, text is not selectable/searchable. Acceptable because the primary use case is visual reports for stakeholders, not text extraction.
- **Browser memory**: 3x scale on a large dashboard produces a very large canvas. On low-memory devices this could be an issue. Not a problem for typical desktop usage.

---

## Libraries Used

| Library | Version | Purpose | Import Style |
|---------|---------|---------|--------------|
| `html2canvas-pro` | ^1.5.12 | Screenshots DOM elements into a `<canvas>` | Dynamic: `await import('html2canvas-pro')` |
| `jspdf` | ^3.0.2 | Generates PDF documents in the browser | Dynamic: `await import('jspdf')` |
| `file-saver` | ^2.0.5 | Triggers browser download dialog | Dynamic: `await import('file-saver')` |
| `@types/file-saver` | ^2.0.7 | TypeScript types for file-saver | Dev dependency |

All three are **dynamically imported** only when the user clicks Download, keeping ~500KB+ out of the initial bundle.

---

## How It Works

### Step-by-step Flow

1. **Dynamic import** `html2canvas-pro`, `jspdf`, and `file-saver`
2. **Create off-screen clone** — hidden `<div>` positioned at `-9999px`, width `1200px`, white background, `32px` padding
3. **Clone header** (`headerRef`) — title, date range, creator info
4. **Clone dashboard** (`dashboardCanvasRef`) — charts + executive summary
5. **Copy ECharts canvas pixel data** — `cloneNode()` does not preserve drawn content on `<canvas>` elements, so each canvas is manually copied via `ctx.drawImage(orig, 0, 0)`
6. **Wait 500ms** for the cloned DOM to render
7. **html2canvas** captures the wrapper at 3x scale with CORS enabled
8. **jsPDF** creates a single custom-sized page (A4 width x content height)
9. **JPEG at quality 1.0** added to the PDF with FAST compression
10. **file-saver** triggers the browser download with a sanitized filename

### Refs Wiring

| Ref | Attached To | Purpose |
|-----|-------------|---------|
| `headerRef` | Header `<div>` (title + metadata row) | Cloned into the off-screen wrapper for PDF |
| `dashboardCanvasRef` | Passed to `DashboardNativeView` via `onContainerRef` callback | Dashboard exposes its container element back to this page for cloning |

---

## Key Design Decisions

| Decision | Reason |
|----------|--------|
| Dynamic imports | Keeps ~500KB+ of PDF libraries out of the initial bundle |
| Off-screen clone | Avoids visual flicker — user never sees the export layout |
| Manual canvas pixel copy | `cloneNode` doesn't preserve ECharts rendered pixel data |
| Single custom-height page | Avoids page-break artifacts from multi-page slicing |
| JPEG at 1.0 quality | Better color preservation than PNG for dashboard screenshots |
| `foreignObjectRendering: false` | Forces canvas rendering path for more accurate colors |
| `compress: false` on jsPDF | Preserves color fidelity in the final PDF |
| `scale: 3` on html2canvas | Higher resolution for sharp text and charts |

---

## html2canvas Options

```typescript
html2canvas(wrapper, {
  scale: 3,
  useCORS: true,
  allowTaint: true,
  backgroundColor: '#ffffff',
  windowWidth: 1200,
  logging: false,
  imageTimeout: 0,
  removeContainer: false,
  foreignObjectRendering: false,
});
```

## jsPDF Options

```typescript
new jsPDF({
  orientation: 'portrait',
  unit: 'pt',
  format: [a4Width, pageHeight],  // A4 width (595.28pt) x content height
  compress: false,
});
```

---

## Commit History

| Commit | Message | Change |
|--------|---------|--------|
| `8c1d034` | new initial version | Basic report viewer (151 lines). **No download at all.** |
| `02fd35c` | pdf download is done | **Added entire download feature (+155 lines).** Off-screen cloning, canvas pixel copy, html2canvas, multi-page A4 slicing with PNG. Redesigned header with Download button. **Bug:** `file-saver` imported as `{ saveAs }` (named) instead of `{ default: saveAs }` — download was broken. |
| `4ea9a1f` | pdf download should be working now | **1-line fix:** `{ saveAs }` → `{ default: saveAs }`. This made download actually work. |
| `70f39fe` | pdf download for report | **Quality optimization (-34/+23 lines).** Replaced multi-page PNG slicing with single custom-height page. Scale 2→3, PNG→JPEG at 1.0 quality, disabled compression, added `foreignObjectRendering: false`. |

### What Changed Across Commits

**Before download (151 lines):**
- No refs, no PDF libraries, no download button
- Simple viewer: header bar, summary edit toggle, frozen dashboard

**After download added (306 lines):**
- Added `headerRef` + `dashboardCanvasRef` for DOM cloning
- Added `handleDownload` with full PDF pipeline
- Added Download button with loading spinner (`isExporting` state)
- Moved executive summary inside `DashboardNativeView` as `beforeContent` (so it's captured in the PDF)

**Bug fix (1 line):**
- `file-saver` exports `saveAs` as default export, not named — fixed the import

**Quality pass (net -11 lines):**

| Before | After |
|--------|-------|
| `scale: 2` | `scale: 3` |
| Multi-page: slice canvas into A4-height chunks in a loop | Single page: custom height `[a4Width, contentHeight]` |
| PNG format | JPEG at quality 1.0 |
| Default jsPDF compression | `compress: false` |
| No extra html2canvas options | `foreignObjectRendering: false`, `logging: false`, `imageTimeout: 0` |

---

## File Changed

| File | Lines | What |
|------|-------|------|
| `app/reports/[snapshotId]/page.tsx` | 94–198 | `handleDownload` function — the entire PDF generation logic |
| `package.json` | +3 deps | `html2canvas-pro`, `jspdf`, `file-saver` added |
