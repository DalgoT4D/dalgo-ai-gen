# Report Download (PDF Export) — Implementation & Knowledge Base

## Overview

Report PDF download is fully **client-side** — no backend PDF generation. The user clicks a Download button on the report viewer page, and a PDF is generated in the browser from the rendered dashboard DOM.

All code lives in `app/reports/[snapshotId]/page.tsx`, inside the `handleDownload` function (lines 94–198).

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

## Files Changed

| File | Lines | What |
|------|-------|------|
| `app/reports/[snapshotId]/page.tsx` | 94–198 | `handleDownload` function — the entire PDF generation logic |
| `package.json` | +3 deps | `html2canvas-pro`, `jspdf`, `file-saver` added |

---

---

# Knowledge Base: Why Chart Export Couldn't Work for Full Dashboard Download

## What Already Existed Before Report Download

### `lib/chart-export.ts` — ChartExporter class

A utility for exporting **individual charts**, not full dashboards:

| Method | What it does |
|--------|--------------|
| `exportEChartsInstance()` | Calls `ECharts.getDataURL()` on a **single** chart instance → saves as PNG or PDF |
| `exportChart()` | Finds one ECharts instance in a DOM element → calls `exportEChartsInstance()` |
| `exportTableAsImage()` | Uses `html2canvas` on a **single** table element → saves as PNG/JPEG |
| `exportTableAsCSV()` | Converts table data to CSV string → saves as `.csv` |

### `components/charts/ChartExportDropdown.tsx`

Dropdown menu on **individual charts** with PNG/PDF/CSV options. Uses `ChartExporter` internally.

### `components/dashboard/dashboard-list-v2.tsx` — Commented-out placeholder

```typescript
// COMMENTED OUT: Handle dashboard download - not needed
// const handleDownloadDashboard = useCallback((dashboardId: number, dashboardTitle: string) => {
//   toastSuccess.generic('Dashboard download will be available soon');
// }, []);
```

A Download option existed in the dashboard dropdown menu but was **fully commented out** with just a placeholder toast. **No actual implementation was ever attempted.**

---

## Why the Existing Chart Export Cannot Work for Full Dashboard PDF

| Problem | Chart Export (`ChartExporter`) | Report Download (`handleDownload`) |
|---------|-------------------------------|-------------------------------------|
| **Scope** | Single chart/table only | Entire page — header + summary + all charts in grid layout |
| **Multiple ECharts** | Gets one `ECharts.getDataURL()` | Clones the whole DOM, then manually copies **every** `<canvas>` element's pixel data |
| **Layout preservation** | No layout — just the raw chart image | Preserves the dashboard grid layout, header, metadata, summary |
| **DOM approach** | Uses ECharts API (`getDataURL`) or `html2canvas` on one element | Creates an off-screen 1200px wrapper, clones header + dashboard container, then `html2canvas` the whole composite |
| **Canvas pixel copy** | Not needed (uses ECharts API directly) | **Required** — `cloneNode()` doesn't preserve drawn `<canvas>` content, so each ECharts canvas is manually redrawn via `ctx.drawImage(orig, 0, 0)` |
| **PDF sizing** | Fits to single chart dimensions | Custom page height: A4 width x calculated content height to fit everything on one page |
| **Quality settings** | `pixelRatio: 2` via ECharts API | `scale: 3`, JPEG 1.0 quality, `compress: false`, `foreignObjectRendering: false` |
| **file-saver import** | Static: `import { saveAs } from 'file-saver'` (line 1 of chart-export.ts) | Dynamic: `const { default: saveAs } = await import('file-saver')` |

### The fundamental limitation

`ChartExporter` is built around **single ECharts instances**. It calls `chartInstance.getDataURL()` which returns one chart's pixels. There is no way to use this to capture a full dashboard with:
- Multiple charts in a grid
- Header with title, dates, metadata
- Executive summary text
- The overall page layout

---

## The 3 Key Techniques That Made Report Download Work

### 1. Off-screen DOM Cloning

The chart export works on **live DOM elements**. The report download creates a **hidden off-screen clone** of the entire page:

```typescript
const wrapper = document.createElement('div');
wrapper.style.position = 'absolute';
wrapper.style.left = '-9999px';
wrapper.style.width = '1200px';
wrapper.style.backgroundColor = '#ffffff';
wrapper.style.padding = '32px';

// Clone header
const headerClone = headerRef.current.cloneNode(true);
wrapper.appendChild(headerClone);

// Clone dashboard (all charts + summary)
const canvasClone = dashboardCanvasRef.current.cloneNode(true);
wrapper.appendChild(canvasClone);

document.body.appendChild(wrapper);
```

**Why this matters:** Cloning lets us capture the entire composite layout as one screenshot without visual flicker or disrupting the user's view.

### 2. Manual Canvas Pixel Transfer

This is the **biggest difference** and the hardest problem. When you `cloneNode(true)` a `<div>` containing ECharts `<canvas>` elements, the HTML structure is cloned but **the drawn pixel data is lost** — the cloned canvases are blank.

The report download fixes this by manually iterating every canvas and redrawing:

```typescript
const originalCanvases = dashboardCanvasRef.current.querySelectorAll('canvas');
const clonedCanvases = canvasClone.querySelectorAll('canvas');
originalCanvases.forEach((orig, i) => {
  const clone = clonedCanvases[i];
  if (clone) {
    clone.width = orig.width;
    clone.height = orig.height;
    const ctx = clone.getContext('2d');
    if (ctx) {
      ctx.drawImage(orig, 0, 0);
    }
  }
});
```

**Why the chart export doesn't need this:** It uses `ECharts.getDataURL()` which reads pixel data directly from the chart instance's internal API — it never clones the DOM.

### 3. Composite html2canvas + Custom-height PDF

Instead of exporting individual elements, `html2canvas` captures the **entire wrapper** (header + summary + all charts) as one big screenshot. Then jsPDF creates a single page sized exactly to fit:

```typescript
const imgHeight = (canvas.height * contentWidth) / canvas.width;
const pageHeight = imgHeight + margin * 2;

const pdf = new jsPDF({
  orientation: 'portrait',
  unit: 'pt',
  format: [a4Width, pageHeight], // Custom: A4 width x content height
  compress: false,
});
```

**Why this matters:** A fixed A4 page would cut charts mid-way. Custom height ensures everything fits on one page with no artifacts.

---

## What Would Be Needed for Live Dashboard Download

To add PDF download to live dashboards (not just frozen report snapshots), you would need the **same approach** as the report download:

1. **Refs** to the dashboard header + chart grid container
2. **Off-screen DOM clone** of the full dashboard layout
3. **Manual canvas pixel copy** for every ECharts `<canvas>` element in the grid
4. **html2canvas** on the composite clone
5. **Custom-sized jsPDF** page to fit all content

The `ChartExporter` class in `lib/chart-export.ts` **cannot be extended** for this — it is fundamentally designed around single chart instances, not composite page layouts. A new function similar to `handleDownload` from the report page would be needed.

---

## Gotchas & Lessons Learned

1. **`file-saver` default export** — `saveAs` is a default export. Use `{ default: saveAs }` with dynamic import, not `{ saveAs }`. This was the bug that initially broke the download.

2. **`cloneNode` doesn't copy canvas pixels** — This is a browser behavior. `<canvas>` elements lose their drawn content when cloned. You must manually copy with `drawImage()`.

3. **JPEG > PNG for color fidelity in dashboards** — Counter-intuitive, but JPEG at 1.0 quality with `compress: false` on jsPDF preserved colors better than PNG with default compression.

4. **`foreignObjectRendering: false`** — html2canvas has two rendering modes. Forcing canvas rendering (not foreignObject) produces more accurate colors for ECharts content.

5. **500ms render delay is necessary** — The cloned DOM needs time to settle before html2canvas captures it. Without this wait, charts may appear blank or partially rendered.

6. **Scale 3x for readability** — Scale 2x was blurry on retina displays. Scale 3x produces sharp text and chart labels in the final PDF.
