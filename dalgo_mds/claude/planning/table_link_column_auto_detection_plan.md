# Table Link Column Auto-Detection Feature Plan

**Date Created:** 2026-01-27
**Author:** Planning Document
**Status:** Draft for Review

---

## Executive Summary

This document outlines a comprehensive plan to implement automatic URL/link detection and rendering in table charts within Dalgo. The feature will automatically recognize columns containing URLs (values starting with `http://` or `https://`) and render them as clickable blue hyperlinks instead of plain text.

### Feature Scope

| Aspect | Description |
|--------|-------------|
| **Target Component** | Table Chart only (not applicable to bar, pie, line, number, map charts) |
| **Detection Method** | Auto-detect URL patterns in cell values at render time |
| **Configuration** | Optional manual override via column formatting configuration |
| **Visual Style** | Blue text with underline on hover (matches existing drill-down style) |
| **Behavior** | Opens URL in new tab when clicked |

### Key Benefits

1. **Zero Configuration**: Automatically works without user setup
2. **Improved UX**: Clickable links instead of copying/pasting URLs
3. **Consistent Design**: Matches existing clickable cell styling (drill-down)
4. **Backward Compatible**: Existing table charts work without changes

---

## 1. Current Implementation Analysis

### 1.1 Table Chart Architecture

**Key File:** `/Users/siddhant/Documents/Dalgo/webapp_v2/components/charts/TableChart.tsx`

The `TableChart` component is responsible for rendering tabular data with the following capabilities:
- Column formatting (currency, percentage, date, number, text)
- Pagination (client-side and server-side)
- Sorting via column headers
- Drill-down support (clickable cells for hierarchical navigation)

**Current Cell Rendering Logic (Lines 131-165):**
```typescript
const formatCellValue = (value: any, column: string) => {
  const formatting = column_formatting[column];

  if (!formatting || value == null) {
    return value?.toString() || '';
  }

  const { type, precision = 2, prefix = '', suffix = '' } = formatting;

  switch (type) {
    case 'currency':
      const currencyValue = typeof value === 'number' ? value : parseFloat(value) || 0;
      return `${prefix}$${currencyValue.toFixed(precision)}${suffix}`;

    case 'percentage':
      const percentValue = typeof value === 'number' ? value : parseFloat(value) || 0;
      return `${prefix}${(percentValue * 100).toFixed(precision)}%${suffix}`;

    case 'number':
      const numValue = typeof value === 'number' ? value : parseFloat(value) || 0;
      return `${prefix}${numValue.toFixed(precision)}${suffix}`;

    case 'date':
      try {
        const dateValue = new Date(value);
        return `${prefix}${dateValue.toLocaleDateString()}${suffix}`;
      } catch {
        return value?.toString() || '';
      }

    case 'text':
    default:
      return `${prefix}${value?.toString() || ''}${suffix}`;
  }
};
```

**Current Cell Rendering (Lines 281-302):**
```typescript
{columns.map((column) => {
  const isClickable =
    drillDownEnabled && currentDimensionColumn === column && onRowClick;
  const cellValue = formatCellValue(row[column], column);

  return (
    <TableCell
      key={column}
      className={`py-1.5 px-2 ${
        isClickable ? 'text-blue-600 hover:text-blue-800 hover:underline' : ''
      }`}
      onClick={
        isClickable
          ? () => {
              onRowClick(row, column);
            }
          : undefined
      }
    >
      {cellValue}
    </TableCell>
  );
})}
```

### 1.2 Column Formatting Configuration

**Key File:** `/Users/siddhant/Documents/Dalgo/webapp_v2/components/charts/TableConfiguration.tsx`

Current formatting types available:
- `text` - Plain text (default)
- `currency` - Dollar format with precision
- `percentage` - Percentage with precision
- `date` - Date format
- `number` - Numeric with precision

**Interface Definition (Lines 24-38):**
```typescript
interface TableConfigurationProps {
  availableColumns: string[];
  selectedColumns: string[];
  columnFormatting: Record<
    string,
    {
      type?: 'currency' | 'percentage' | 'date' | 'number' | 'text';
      precision?: number;
      prefix?: string;
      suffix?: string;
    }
  >;
  onColumnsChange: (columns: string[]) => void;
  onFormattingChange: (formatting: Record<string, any>) => void;
}
```

### 1.3 Type Definitions

**Key File:** `/Users/siddhant/Documents/Dalgo/webapp_v2/types/charts.ts`

```typescript
// Current column_formatting type in ChartCreate (Lines 117-125)
column_formatting?: Record<
  string,
  {
    type?: 'currency' | 'percentage' | 'date' | 'number' | 'text';
    precision?: number;
    prefix?: string;
    suffix?: string;
  }
>;
```

### 1.4 Existing Link Styling Pattern

The codebase already has a pattern for clickable blue links used in drill-down functionality:
```typescript
className={`py-1.5 px-2 ${
  isClickable ? 'text-blue-600 hover:text-blue-800 hover:underline' : ''
}`}
```

This same styling will be reused for URL links to maintain visual consistency.

---

## 2. Technical Design

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Data Flow                                    │
│                                                                     │
│  1. API Response with table data                                    │
│     ↓                                                               │
│  2. TableChart receives data and config                             │
│     ↓                                                               │
│  3. For each cell:                                                  │
│     a. Check column_formatting for explicit 'link' type             │
│     b. If no explicit type, auto-detect URL pattern in value        │
│     c. If URL detected → render as <a> tag                          │
│     d. Otherwise → render as plain text (existing logic)            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 URL Detection Strategy

**Option A: Auto-Detection Only (Recommended)**
- Detect URLs at render time based on value pattern
- No configuration required
- Works immediately for all existing table charts

**Option B: Configuration-Based Only**
- User explicitly marks columns as 'link' type
- Requires manual setup per column
- More control but less convenient

**Option C: Hybrid Approach (Recommended for MVP)**
- Auto-detect URLs by default
- Allow explicit `type: 'link'` or `type: 'url'` in column_formatting to force link rendering
- Allow `auto_detect_links: false` to disable auto-detection globally

**Recommended: Option C (Hybrid Approach)**

### 2.3 URL Detection Logic

```typescript
/**
 * Checks if a value is a valid URL that should be rendered as a link
 * @param value - The cell value to check
 * @returns boolean indicating if value is a URL
 */
function isValidUrl(value: any): boolean {
  if (value == null || typeof value !== 'string') {
    return false;
  }

  const trimmedValue = value.trim();

  // Check for common URL patterns
  // Matches: http://, https://, or www.
  const urlPattern = /^(https?:\/\/|www\.)/i;

  return urlPattern.test(trimmedValue);
}

/**
 * Normalizes a URL for use in href attribute
 * Adds https:// prefix to www. URLs if missing
 */
function normalizeUrl(url: string): string {
  const trimmed = url.trim();
  if (trimmed.toLowerCase().startsWith('www.')) {
    return `https://${trimmed}`;
  }
  return trimmed;
}
```

### 2.4 Component Changes

#### 2.4.1 Updated Column Formatting Type

```typescript
// types/charts.ts - Update column_formatting type
column_formatting?: Record<
  string,
  {
    type?: 'currency' | 'percentage' | 'date' | 'number' | 'text' | 'link';  // Add 'link'
    precision?: number;
    prefix?: string;
    suffix?: string;
    // New link-specific options
    link_target?: '_blank' | '_self';  // Default: '_blank'
    link_display?: 'full' | 'truncated' | 'text';  // How to display the link
    link_text?: string;  // Static text to display instead of URL
  }
>;
```

#### 2.4.2 Updated TableChart Component

```typescript
// TableChart.tsx - New helper functions

/**
 * URL detection regex pattern
 * Matches http://, https://, and www. prefixed URLs
 */
const URL_PATTERN = /^(https?:\/\/|www\.)/i;

/**
 * Check if a value should be rendered as a link
 */
function shouldRenderAsLink(
  value: any,
  column: string,
  columnFormatting: Record<string, any>
): boolean {
  const formatting = columnFormatting[column];

  // If explicitly set to 'link' type, render as link
  if (formatting?.type === 'link') {
    return true;
  }

  // If explicitly set to another type, don't auto-detect
  if (formatting?.type && formatting.type !== 'text') {
    return false;
  }

  // Auto-detect URLs in text/default columns
  return isValidUrl(value);
}

function isValidUrl(value: any): boolean {
  if (value == null || typeof value !== 'string') {
    return false;
  }
  return URL_PATTERN.test(value.trim());
}

function normalizeUrl(url: string): string {
  const trimmed = url.trim();
  if (trimmed.toLowerCase().startsWith('www.')) {
    return `https://${trimmed}`;
  }
  return trimmed;
}

/**
 * Get display text for a link
 * Can truncate long URLs or use custom text
 */
function getLinkDisplayText(
  url: string,
  formatting?: { link_display?: string; link_text?: string }
): string {
  if (formatting?.link_text) {
    return formatting.link_text;
  }

  if (formatting?.link_display === 'truncated') {
    // Show domain + truncated path
    try {
      const urlObj = new URL(normalizeUrl(url));
      const path = urlObj.pathname.length > 20
        ? urlObj.pathname.slice(0, 20) + '...'
        : urlObj.pathname;
      return `${urlObj.hostname}${path}`;
    } catch {
      return url.length > 50 ? url.slice(0, 47) + '...' : url;
    }
  }

  // 'full' or default - show entire URL
  return url;
}
```

#### 2.4.3 Updated Cell Rendering

```typescript
// TableChart.tsx - Updated cell rendering logic

{columns.map((column) => {
  const isDrillDownClickable =
    drillDownEnabled && currentDimensionColumn === column && onRowClick;
  const cellValue = row[column];
  const formatting = column_formatting[column];
  const isLink = shouldRenderAsLink(cellValue, column, column_formatting);

  // Determine if cell should be rendered as a link
  if (isLink && !isDrillDownClickable) {
    const href = normalizeUrl(cellValue);
    const displayText = getLinkDisplayText(cellValue, formatting);
    const target = formatting?.link_target || '_blank';

    return (
      <TableCell key={column} className="py-1.5 px-2">
        <a
          href={href}
          target={target}
          rel={target === '_blank' ? 'noopener noreferrer' : undefined}
          className="text-blue-600 hover:text-blue-800 hover:underline cursor-pointer"
          onClick={(e) => e.stopPropagation()} // Prevent row click from firing
        >
          {displayText}
        </a>
      </TableCell>
    );
  }

  // Existing rendering logic for non-link cells
  const formattedValue = formatCellValue(cellValue, column);

  return (
    <TableCell
      key={column}
      className={`py-1.5 px-2 ${
        isDrillDownClickable ? 'text-blue-600 hover:text-blue-800 hover:underline cursor-pointer' : ''
      }`}
      onClick={
        isDrillDownClickable
          ? () => onRowClick(row, column)
          : undefined
      }
    >
      {formattedValue}
    </TableCell>
  );
})}
```

### 2.5 Configuration UI Updates (Optional Enhancement)

#### 2.5.1 TableConfiguration.tsx Updates

Add 'Link' as a formatting option in the column configuration UI:

```typescript
// TableConfiguration.tsx - Add link option to format type dropdown

<Select
  value={formatting.type || 'text'}
  onValueChange={(value) => handleFormattingChange(column, 'type', value)}
>
  <SelectTrigger className="h-8">
    <SelectValue />
  </SelectTrigger>
  <SelectContent>
    <SelectItem value="text">Text</SelectItem>
    <SelectItem value="number">Number</SelectItem>
    <SelectItem value="currency">Currency</SelectItem>
    <SelectItem value="percentage">Percentage</SelectItem>
    <SelectItem value="date">Date</SelectItem>
    <SelectItem value="link">Link</SelectItem>  {/* NEW */}
  </SelectContent>
</Select>

{/* Link-specific options (shown when type is 'link') */}
{formatting.type === 'link' && (
  <div className="mt-2 space-y-2">
    <div>
      <Label className="text-xs">Link Target</Label>
      <Select
        value={formatting.link_target || '_blank'}
        onValueChange={(value) => handleFormattingChange(column, 'link_target', value)}
      >
        <SelectTrigger className="h-8">
          <SelectValue />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="_blank">New Tab</SelectItem>
          <SelectItem value="_self">Same Tab</SelectItem>
        </SelectContent>
      </Select>
    </div>
    <div>
      <Label className="text-xs">Display Mode</Label>
      <Select
        value={formatting.link_display || 'full'}
        onValueChange={(value) => handleFormattingChange(column, 'link_display', value)}
      >
        <SelectTrigger className="h-8">
          <SelectValue />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="full">Full URL</SelectItem>
          <SelectItem value="truncated">Truncated</SelectItem>
        </SelectContent>
      </Select>
    </div>
  </div>
)}
```

---

## 3. Implementation Plan

### 3.1 Phase 1: Core Auto-Detection (MVP)

**Files to Modify:**
1. `webapp_v2/components/charts/TableChart.tsx`
   - Add URL detection helpers (`isValidUrl`, `normalizeUrl`)
   - Add `shouldRenderAsLink` function
   - Update cell rendering to check for URLs and render `<a>` tags
   - Add proper click handling to prevent conflicts with drill-down

**Implementation Steps:**

1. **Add URL detection utilities at the top of TableChart.tsx:**
```typescript
// URL detection pattern
const URL_PATTERN = /^(https?:\/\/|www\.)/i;

function isValidUrl(value: any): boolean {
  if (value == null || typeof value !== 'string') {
    return false;
  }
  return URL_PATTERN.test(value.trim());
}

function normalizeUrl(url: string): string {
  const trimmed = url.trim();
  if (trimmed.toLowerCase().startsWith('www.')) {
    return `https://${trimmed}`;
  }
  return trimmed;
}
```

2. **Update the cell rendering logic inside the `columns.map()` loop:**
```typescript
{columns.map((column) => {
  const isDrillDownClickable =
    drillDownEnabled && currentDimensionColumn === column && onRowClick;
  const rawValue = row[column];
  const isLink = !isDrillDownClickable && isValidUrl(rawValue);

  if (isLink) {
    const href = normalizeUrl(rawValue);
    return (
      <TableCell key={column} className="py-1.5 px-2">
        <a
          href={href}
          target="_blank"
          rel="noopener noreferrer"
          className="text-blue-600 hover:text-blue-800 hover:underline"
          onClick={(e) => e.stopPropagation()}
        >
          {rawValue}
        </a>
      </TableCell>
    );
  }

  // Existing logic for non-link cells
  const cellValue = formatCellValue(rawValue, column);
  return (
    <TableCell
      key={column}
      className={`py-1.5 px-2 ${
        isDrillDownClickable ? 'text-blue-600 hover:text-blue-800 hover:underline' : ''
      }`}
      onClick={
        isDrillDownClickable
          ? () => onRowClick(row, column)
          : undefined
      }
    >
      {cellValue}
    </TableCell>
  );
})}
```

### 3.2 Phase 2: Type Definition Updates

**Files to Modify:**
1. `webapp_v2/types/charts.ts`
   - Add 'link' to column_formatting type union

**Implementation:**
```typescript
// Update column_formatting type in multiple places
column_formatting?: Record<
  string,
  {
    type?: 'currency' | 'percentage' | 'date' | 'number' | 'text' | 'link';
    precision?: number;
    prefix?: string;
    suffix?: string;
    link_target?: '_blank' | '_self';
    link_display?: 'full' | 'truncated';
  }
>;
```

### 3.3 Phase 3: Configuration UI (Optional Enhancement)

**Files to Modify:**
1. `webapp_v2/components/charts/TableConfiguration.tsx`
   - Add 'Link' option to format type dropdown
   - Add link-specific options when link type is selected

### 3.4 Phase 4: Testing

**Unit Tests to Create:**
1. `webapp_v2/components/charts/__tests__/TableChart.link.test.tsx`

```typescript
import { render, screen } from '@testing-library/react';
import { TableChart } from '../TableChart';

describe('TableChart Link Detection', () => {
  it('should render http:// URLs as clickable links', () => {
    const data = [
      { name: 'Test', url: 'http://example.com' }
    ];

    render(<TableChart data={data} />);

    const link = screen.getByRole('link', { name: 'http://example.com' });
    expect(link).toHaveAttribute('href', 'http://example.com');
    expect(link).toHaveAttribute('target', '_blank');
  });

  it('should render https:// URLs as clickable links', () => {
    const data = [
      { name: 'Test', url: 'https://secure.example.com' }
    ];

    render(<TableChart data={data} />);

    const link = screen.getByRole('link', { name: 'https://secure.example.com' });
    expect(link).toHaveAttribute('href', 'https://secure.example.com');
  });

  it('should render www. URLs as clickable links with https prefix', () => {
    const data = [
      { name: 'Test', url: 'www.example.com' }
    ];

    render(<TableChart data={data} />);

    const link = screen.getByRole('link', { name: 'www.example.com' });
    expect(link).toHaveAttribute('href', 'https://www.example.com');
  });

  it('should NOT render non-URL text as links', () => {
    const data = [
      { name: 'Test', description: 'This is not a URL' }
    ];

    render(<TableChart data={data} />);

    expect(screen.queryByRole('link')).not.toBeInTheDocument();
    expect(screen.getByText('This is not a URL')).toBeInTheDocument();
  });

  it('should NOT render null/undefined values as links', () => {
    const data = [
      { name: 'Test', url: null },
      { name: 'Test2', url: undefined }
    ];

    render(<TableChart data={data} />);

    expect(screen.queryByRole('link')).not.toBeInTheDocument();
  });

  it('should prioritize drill-down over link rendering', () => {
    const onRowClick = jest.fn();
    const data = [
      { dimension: 'http://example.com', value: 100 }
    ];

    render(
      <TableChart
        data={data}
        drillDownEnabled={true}
        currentDimensionColumn="dimension"
        onRowClick={onRowClick}
      />
    );

    // Should not be a link since it's a drill-down column
    expect(screen.queryByRole('link')).not.toBeInTheDocument();
  });

  it('should open link in new tab by default', () => {
    const data = [{ url: 'https://example.com' }];

    render(<TableChart data={data} />);

    const link = screen.getByRole('link');
    expect(link).toHaveAttribute('target', '_blank');
    expect(link).toHaveAttribute('rel', 'noopener noreferrer');
  });

  it('should not trigger row click when clicking link', () => {
    const onRowClick = jest.fn();
    const data = [
      { name: 'Test', url: 'https://example.com' }
    ];

    render(
      <TableChart
        data={data}
        drillDownEnabled={true}
        currentDimensionColumn="name"
        onRowClick={onRowClick}
      />
    );

    const link = screen.getByRole('link');
    fireEvent.click(link);

    expect(onRowClick).not.toHaveBeenCalled();
  });
});
```

### 3.5 Phase 5: E2E Testing

**E2E Test File:** `webapp_v2/e2e/table-link-detection.spec.ts`

```typescript
import { test, expect } from '@playwright/test';

test.describe('Table Link Detection', () => {
  test('should render URLs as clickable links in table chart', async ({ page }) => {
    // Navigate to a dashboard with a table containing URLs
    await page.goto('/dashboards/test-dashboard');

    // Find the table chart
    const tableChart = page.locator('[data-testid="table-chart"]');

    // Find a link cell
    const linkCell = tableChart.locator('a[href*="http"]').first();

    // Verify link styling
    await expect(linkCell).toHaveClass(/text-blue-600/);

    // Verify link opens in new tab
    const [newPage] = await Promise.all([
      page.waitForEvent('popup'),
      linkCell.click()
    ]);

    await expect(newPage).toHaveURL(/example\.com/);
  });
});
```

---

## 4. Service Impact Analysis

### 4.1 Frontend Only Change

This feature is **frontend-only**. No backend changes are required because:

1. **Data remains unchanged** - The backend returns raw URL strings as before
2. **No new API endpoints** - Existing chart data APIs work as-is
3. **Configuration storage** - The `column_formatting` field in `extra_config` already supports additional properties

### 4.2 Backend Considerations (Future Enhancement)

For a future enhancement, the backend could provide URL type hints:

```python
# Future: Backend could detect URL columns and add type hints
# This would be in charts_service.py when building table responses

def detect_column_types(data: list[dict]) -> dict[str, str]:
    """Detect column data types from sample data"""
    column_types = {}
    for column in data[0].keys():
        values = [row.get(column) for row in data[:100] if row.get(column)]
        if all(is_url(v) for v in values if v):
            column_types[column] = 'url'
        # ... other type detection
    return column_types
```

This is **not required for MVP** but could improve performance by avoiding client-side detection.

---

## 5. Edge Cases and Error Handling

### 5.1 Edge Cases to Handle

| Case | Behavior | Implementation |
|------|----------|----------------|
| Null/undefined values | Don't render as link | `isValidUrl` returns false for null/undefined |
| Empty strings | Don't render as link | `isValidUrl` returns false for empty strings |
| Partial URLs (missing protocol) | Don't render as link unless starts with www. | Pattern requires `http://`, `https://`, or `www.` |
| Long URLs | Display full URL (truncation is optional enhancement) | Use full URL by default |
| Malformed URLs | Display as plain text | Try-catch in normalizeUrl |
| URLs in drill-down columns | Prioritize drill-down behavior | Check `isDrillDownClickable` first |
| Mixed content columns | Render URLs as links, others as text | Check each cell individually |
| Email addresses | Don't render as links (unless mailto:) | Pattern doesn't match emails without mailto: |
| Javascript: URLs | Don't render (security) | Only match http:// and https:// |
| File:// URLs | Don't render (security) | Only match http:// and https:// |

### 5.2 Security Considerations

1. **XSS Prevention**: Only render `http://` and `https://` URLs
2. **No JavaScript URLs**: Pattern explicitly excludes `javascript:` URLs
3. **New Tab Security**: Always add `rel="noopener noreferrer"` for `target="_blank"` links
4. **No Dynamic Href**: URL is taken directly from data, not constructed from user input

### 5.3 Error Handling

```typescript
function normalizeUrl(url: string): string {
  try {
    const trimmed = url.trim();
    if (trimmed.toLowerCase().startsWith('www.')) {
      return `https://${trimmed}`;
    }
    // Validate URL is parseable
    new URL(trimmed);
    return trimmed;
  } catch {
    // If URL parsing fails, return original value
    // Cell will render as plain text
    return url;
  }
}
```

---

## 6. Testing Strategy

### 6.1 Unit Test Coverage

| Test Category | Test Cases |
|--------------|------------|
| **URL Detection** | http:// URLs, https:// URLs, www. URLs, non-URLs, null/undefined, empty strings |
| **Link Rendering** | Correct href, target attribute, rel attribute, styling classes |
| **Drill-Down Priority** | Links not rendered in drill-down columns, proper click handling |
| **Edge Cases** | Long URLs, malformed URLs, mixed content columns |

### 6.2 Integration Test Coverage

| Test Category | Test Cases |
|--------------|------------|
| **ChartBuilder Flow** | Create table chart with URL column, save and reload |
| **Dashboard Rendering** | Table with URLs renders correctly on dashboard |
| **Public Dashboards** | Links work in public/shared dashboards |

### 6.3 E2E Test Coverage

| Test Category | Test Cases |
|--------------|------------|
| **User Journey** | Create chart → View table → Click link → Verify new tab opens |
| **Visual Regression** | Link styling matches design specs |

### 6.4 Manual Testing Checklist

- [ ] Create a table chart with a column containing http:// URLs
- [ ] Create a table chart with a column containing https:// URLs
- [ ] Create a table chart with a column containing www. URLs
- [ ] Verify links are blue and underlined on hover
- [ ] Verify clicking link opens new tab
- [ ] Verify links don't interfere with drill-down functionality
- [ ] Verify links work in shared/public dashboards
- [ ] Verify null/empty values don't render as links
- [ ] Verify non-URL text doesn't render as links
- [ ] Test in Chrome, Firefox, Safari browsers

---

## 7. Success Criteria

### 7.1 Functional Requirements

- [ ] URLs starting with `http://` are rendered as clickable blue links
- [ ] URLs starting with `https://` are rendered as clickable blue links
- [ ] URLs starting with `www.` are rendered as clickable blue links (with https:// prefix)
- [ ] Non-URL text is NOT rendered as links
- [ ] Links open in new tab by default
- [ ] Links have proper security attributes (`rel="noopener noreferrer"`)
- [ ] Drill-down columns still work correctly (priority over link detection)
- [ ] Feature works without any user configuration

### 7.2 Quality Requirements

- [ ] 80%+ unit test coverage on new link detection code
- [ ] All existing TableChart tests continue to pass
- [ ] No console errors or warnings
- [ ] No accessibility regressions (links are keyboard navigable)

### 7.3 Performance Requirements

- [ ] No noticeable render delay for tables with 100+ rows
- [ ] URL detection should be O(n) where n is number of cells

---

## 8. File Change Summary

### 8.1 Files to Modify

| File | Changes |
|------|---------|
| `webapp_v2/components/charts/TableChart.tsx` | Add URL detection logic, update cell rendering |
| `webapp_v2/types/charts.ts` | Add 'link' to column_formatting type (optional Phase 2) |
| `webapp_v2/components/charts/TableConfiguration.tsx` | Add Link option to dropdown (optional Phase 3) |

### 8.2 New Files to Create

| File | Purpose |
|------|---------|
| `webapp_v2/components/charts/__tests__/TableChart.link.test.tsx` | Unit tests for link detection |
| `webapp_v2/e2e/table-link-detection.spec.ts` | E2E tests for link feature |

### 8.3 No Backend Changes Required

This is a frontend-only feature. The backend returns data unchanged.

---

## 9. Rollout Plan

### 9.1 Development (Day 1-2)
- Implement core URL detection in TableChart.tsx
- Add unit tests
- Manual testing

### 9.2 Code Review (Day 3)
- PR review
- Address feedback
- Merge to main

### 9.3 QA Testing (Day 4)
- E2E test execution
- Cross-browser testing
- Regression testing

### 9.4 Deployment (Day 5)
- Deploy to staging
- Final verification
- Deploy to production

### 9.5 Monitoring
- Monitor for any console errors
- Check for user feedback
- Address any issues

---

## 10. Future Enhancements

After MVP, consider these enhancements:

1. **Link Display Options**
   - Truncated display for long URLs
   - Custom display text per column
   - Icon indicator for external links

2. **Configuration UI**
   - Add "Link" as explicit format type in TableConfiguration
   - Link-specific options (target, display mode)

3. **Backend Type Detection**
   - Server-side URL column detection
   - Type hints in API response

4. **Email Link Support**
   - Detect email addresses
   - Render as `mailto:` links

5. **Phone Link Support**
   - Detect phone numbers
   - Render as `tel:` links

---

## 11. Appendix

### 11.1 URL Pattern Regex Explanation

```javascript
const URL_PATTERN = /^(https?:\/\/|www\.)/i;
```

- `^` - Start of string
- `(https?:\/\/|www\.)` - Match either:
  - `https?://` - "http://" or "https://" (`s?` makes 's' optional)
  - `www.` - Literal "www."
- `i` - Case insensitive flag

This pattern:
- Matches: `http://example.com`, `https://example.com`, `www.example.com`, `HTTP://EXAMPLE.COM`
- Does NOT match: `ftp://files.com`, `mailto:email@example.com`, `javascript:alert(1)`, `example.com`

### 11.2 References

- Existing drill-down styling: `webapp_v2/components/charts/TableChart.tsx:289-291`
- Column formatting types: `webapp_v2/types/charts.ts:117-125`
- Table configuration UI: `webapp_v2/components/charts/TableConfiguration.tsx`
- Drill-Down Enhancement Plan: `dalgo_mds/planning/drill_down_table_enhancement_plan.md`

---

## End of Document

**Next Steps:**
1. Review this plan with the development team
2. Approve MVP scope (auto-detection only)
3. Begin Phase 1 implementation
4. Create PR with unit tests
5. Execute E2E tests before deployment
