# Page-Break-Safe PDF Export

## Problem

Two issues with the current PDF export:

1. **Charts get cropped across page boundaries** — `react-grid-layout` uses `position: absolute` with inline pixel positions. CSS `break-inside: avoid` doesn't work with absolutely positioned elements, so Chromium's print engine can't reflow them.

2. **Missing report metadata** — The PDF header only shows the report title and date range. It doesn't include the creator or dashboard name.

3. **Filters appear in the PDF** — Filter components are rendered in the PDF output but serve no purpose in a static document.

## Current Architecture

**How PDF export works today:**

1. User clicks Download on `app/reports/[snapshotId]/page.tsx` (line 149)
2. `usePdfDownload` hook calls `POST /api/reports/{id}/export/pdf/`
3. Backend `pdf_export_service.py` launches headless Chromium, navigates to `/share/report/{token}?print=true`
4. `PublicReportView.tsx` detects `printMode=true` and renders the dashboard with `isPrintMode={true}` on `DashboardNativeView`
5. `DashboardNativeView` still uses `react-grid-layout` (GridLayout) with `position: absolute` items
6. Playwright calls `page.pdf(format="A4")` — Chromium can't reflow absolute elements across pages, so charts get cropped

**Key insight:** The `isPrintMode` flag currently only removes height constraints and overflow (`h-auto overflow-visible`) but doesn't change the layout engine. The grid items remain absolutely positioned.

## Solution

Three changes:

1. **Replace grid layout with document flow in print mode** — A new `<PrintLayout>` component groups layout items by row and renders them as flex containers with `break-inside: avoid`, giving Chromium reflowable content.

2. **Show full report metadata in the PDF header** — Date range, created by, and dashboard name are all displayed at the top of the PDF.

3. **Exclude filter components from the PDF** — The `PrintLayout` component skips any component with `type === 'filter'`.

## Architecture

### Files Changed

| File | Change |
|------|--------|
| `webapp_v2/app/share/report/[token]/PublicReportView.tsx` | In `printMode` branch, replace `<DashboardNativeView>` with `<PrintLayout>` and render full metadata header |
| `webapp_v2/components/reports/print-layout.tsx` | **New file**: Renders dashboard components grouped by row in normal document flow, skips filters |
| `DDP_backend/.../pdf_export_service.py` | Change `VIEWPORT_WIDTH` to 794 (A4 width at 96 DPI) |

### How rows are grouped

`layout_config` items have `{ i, x, y, w, h }`. Items sharing the same `y` value are on the same visual row. We group by `y`, sort groups by `y` ascending, and within each group sort by `x`.

Example layout:
```
y=0:  [chart-A (x=0, w=6), chart-B (x=6, w=6)]   → Row 1 (two charts side by side)
y=15: [chart-C (x=0, w=12)]                        → Row 2 (full-width chart)
y=30: [chart-D (x=0, w=4), chart-E (x=4, w=4), chart-F (x=8, w=4)] → Row 3
```

Each row renders as:
```html
<div style="break-inside: avoid; display: flex; gap: 8px;">
  <div style="flex: 6/12">  <!-- proportional width from w/totalCols -->
    <ChartElementView ... />
  </div>
  <div style="flex: 6/12">
    <ChartElementView ... />
  </div>
</div>
```

### PrintLayout component

The `PrintLayout` component (`components/reports/print-layout.tsx`):

**Props:** Same data that `PublicReportView` already has — `dashboard_data`, `frozen_chart_configs`, `token`.

**Rendering logic:**
1. Read `dashboard_data.layout_config` and `dashboard_data.components`
2. **Skip any component where `type === 'filter'`** — filters are excluded from the PDF
3. Group remaining layout items by `y` value → each group = one "row"
4. Sort rows by `y`, items within a row by `x`
5. For each row, render a flex container where each item gets `flex: {w}` (proportional width)
6. Each item renders using the same component logic as `renderComponent()` in `dashboard-native-view.tsx` (lines 533-644): `ChartElementView` for charts, `UnifiedTextElement` for text, headings via tag
7. Apply `break-inside: avoid` on each row container
8. Give each chart item a fixed height derived from `h * ROW_HEIGHT_PX` (where `ROW_HEIGHT_PX = 20` matching the grid's `rowHeight={20}`) with a minimum of 300px

**Why a separate component instead of modifying DashboardNativeView?**
- `DashboardNativeView` is 1300+ lines with lots of interactive features (filters, fullscreen, landing page, etc.) that don't apply to print
- A focused component is easier to maintain and test
- No risk of breaking the interactive dashboard

### PDF Header with full metadata

The print mode branch in `PublicReportView.tsx` renders a header section with all report details:

```
┌─────────────────────────────────────────────────┐
│ Report Title                                     │
│ All - Mar 31st, 2026                            │
│ Created by: admin5@example.com                  │
│ new dashboard - test for report                 │
├─────────────────────────────────────────────────┤
│ Executive Summary (if present)                  │
│ ...                                             │
├─────────────────────────────────────────────────┤
│ Charts (no filters)                             │
└─────────────────────────────────────────────────┘
```

The metadata fields come from `report_metadata`:
- **Date range**: `period_start` (or "All" if null) and `period_end` — formatted as "All - Mar 31st, 2026"
- **Created by**: `created_by` — the email of the user who created the report
- **Dashboard name**: `dashboard_title` — the source dashboard name

### Backend change

In `pdf_export_service.py`:
- Change `VIEWPORT_WIDTH = 794` (A4 portrait width at 96 DPI: 210mm / 25.4 * 96 ≈ 794px)
- The print layout on the frontend auto-sizes to fill the viewport width, so this ensures charts render at the correct proportions for A4 paper

## Implementation Steps

### Step 1: Create `print-layout.tsx`

Create `webapp_v2/components/reports/print-layout.tsx`:

- Helper function `groupLayoutByRows(layoutConfig, components)`:
  - Filter out any layout item whose component has `type === 'filter'`
  - Group remaining items by `y` value
  - Sort groups by `y` ascending, items within each group by `x` ascending
- Render each group as a `<div className="flex gap-2" style={{ breakInside: 'avoid' }}>`
- Each item gets `style={{ flex: item.w, minHeight: Math.max(item.h * 20, 300) }}` for charts, or auto height for text/headings
- Component rendering logic duplicated from `renderComponent()` in `dashboard-native-view.tsx` (lines 533-644) but simplified:
  - `chart` → `<ChartElementView>` (read-only, no filter state)
  - `text` → `<UnifiedTextElement>` (read-only)
  - `heading` → `<HeadingTag>` with appropriate level
  - `filter` → **skipped** (not rendered)
- Wrap charts in a `<Card>` like the grid does (matching `dashboard-native-view.tsx` line 1206-1209)

### Step 2: Update `PublicReportView.tsx` print mode branch

Replace the current print mode rendering (lines 59-94) which uses `<DashboardNativeView isPrintMode={true}>` with `<PrintLayout>` and add full metadata header:

```tsx
if (printMode) {
  return (
    <div className="bg-white w-full" data-pdf-ready="true">
      {/* Full metadata header */}
      <div className="px-6 py-4 border-b">
        <h1 className="text-xl font-semibold text-gray-900">
          {report_metadata.title}
        </h1>
        <div className="flex items-center gap-2 text-sm text-gray-600 mt-1">
          <Calendar className="h-4 w-4" />
          <span>
            {report_metadata.period_start
              ? formatDateShort(report_metadata.period_start)
              : 'All'}{' '}
            - {formatDateShort(report_metadata.period_end)}
          </span>
        </div>
        {report_metadata.created_by && (
          <p className="text-sm text-gray-500 mt-1">
            Created by: {report_metadata.created_by}
          </p>
        )}
        {report_metadata.dashboard_title && (
          <p className="text-sm text-gray-500 mt-1">
            {report_metadata.dashboard_title}
          </p>
        )}
      </div>

      {/* Executive summary */}
      {report_metadata.summary && (
        <div className="px-6 py-4 border rounded-lg m-6 bg-background">
          <h2 className="text-lg font-semibold mb-2">Executive Summary</h2>
          <p className="text-sm text-muted-foreground whitespace-pre-wrap">
            {report_metadata.summary}
          </p>
        </div>
      )}

      {/* Charts only — no filters */}
      <PrintLayout
        dashboardData={dashboard_data}
        frozenChartConfigs={frozen_chart_configs}
        publicToken={token}
        isPublicMode={true}
      />
    </div>
  );
}
```

### Step 3: Update backend viewport

In `pdf_export_service.py`:
- Change line 10: `VIEWPORT_WIDTH = 794`
- This matches A4 width at 96 DPI so the print layout renders at the right proportions

## How page breaks are prevented

The core mechanism:

1. **Document flow layout** — Instead of `position: absolute` (which Chromium can't reflow), each row is a normal flex container in the document flow.

2. **`break-inside: avoid`** — Applied to each row div. Chromium's print engine respects this and pushes the entire row to the next page if it won't fit on the current page.

3. **Row-level granularity** — Page breaks can only occur *between* rows, never through a chart. Charts within the same row stay together.

**Edge case:** If a single row is taller than an entire A4 page (~1123px at 96 DPI), `break-inside: avoid` can't help — Chromium has no choice but to split it. The plan mitigates this with the height formula `h * 20` (matching the grid's `rowHeight=20`) with a 300px minimum, which keeps most charts well under a page height.

## Key Design Decisions

1. **Row grouping by `y` value**: Preserves the original dashboard layout intent — charts placed side-by-side stay side-by-side in the PDF.

2. **Separate `PrintLayout` component**: Playwright renders via `?print=true` → `PublicReportView` → `PrintLayout`. No interactive features needed.

3. **No changes to `DashboardNativeView`**: The 1300-line grid component stays untouched. The print layout is a completely separate render path.

4. **Chart height = `h * 20`**: Matches `rowHeight={20}` in `dashboard-native-view.tsx` (line 1190). Minimum 300px ensures charts are readable.

5. **Reuse existing component imports**: `ChartElementView`, `UnifiedTextElement` are imported and used directly — the rendering logic is duplicated from `renderComponent()` but simplified (no filters, no interactivity).

6. **Viewport width change**: 1200→794 only affects the headless Chromium render. The public web view isn't impacted because browsers use their own viewport width.

7. **Filters excluded**: Filter components (`type === 'filter'`) are skipped during rendering. They add no value in a static PDF — the date range is already shown in the header metadata.

8. **Full metadata in header**: Title, date range, created by, and dashboard name give the reader full context about the report without needing to reference the app.
