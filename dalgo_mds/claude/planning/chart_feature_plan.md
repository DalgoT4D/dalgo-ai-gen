# Dalgo Charts Feature Implementation Plan

## Feature Overview

This plan details the implementation of charting capabilities in Dalgo, allowing users to create, view, edit, update, and save charts with a smooth user experience. The feature integrates Apache ECharts for visualization and leverages Dalgo's existing infrastructure for data querying and transformation.

## Architecture Overview

### High-Level Data Flow
```
User Interface (webapp_v2) → Backend API (DDP_backend) → Query Builder → Data Warehouse → Data Transformation → ECharts Config → Chart Rendering
```

### Service Modifications Required

#### 1. **DDP_backend** (Django Backend)
- Create new chart management endpoints
- Enhance query builder for aggregation support
- Add ECharts configuration generation
- Implement data transformation logic

#### 2. **webapp_v2** (Frontend)
- Create chart listing page
- Implement chart creation/editing interface
- Integrate Apache ECharts
- Add live preview functionality
- Implement data preview table

## Chart Types and Form Fields

### Common Fields for All Chart Types

#### 1. Data Source Selection
- **Schema Name** (dropdown) - Required
- **Table Name** (dropdown) - Required, populated after schema selection

#### 2. Chart Metadata
- **Chart Title** (text input) - Required
- **Chart Description** (textarea) - Optional

#### 3. Data Configuration
Depends on computation type:

##### For Raw Data:
- **X-Axis Column** (dropdown) - Required
- **Y-Axis Column** (dropdown) - Required, only numeric columns

##### For Aggregated Data:
- **Dimension Column** (dropdown) - Required, grouping column
- **Aggregate Column** (dropdown) - Required, numeric column to aggregate
- **Aggregate Function** (dropdown) - Required
  - Options: SUM, AVG, COUNT, MIN, MAX, COUNT_DISTINCT

### Bar Chart Specific Configuration

#### Basic Configuration
1. **Orientation** (radio buttons)
   - Vertical Bar (default)
   - Horizontal Bar

2. **Bar Style** (radio buttons)
   - Regular Bars (default)
   - Stacked Bars (only available when extra dimension is added)

3. **Extra Dimension** (optional dropdown)
   - Allows adding one additional dimension for grouped/stacked bars

#### Customization Options
1. **Show Data Labels** (toggle) - Default: false
2. **X-Axis Title** (text input) - Default: Column name
3. **Y-Axis Title** (text input) - Default: Column name or aggregate expression
4. **Show Legend** (toggle) - Default: true (when extra dimension exists)

### Pie Chart Specific Configuration

#### Basic Configuration
1. **Chart Style** (radio buttons)
   - Full Pie (default)
   - Donut Chart

2. **Extra Dimension** (optional dropdown)
   - Allows adding one additional dimension for nested data

#### Customization Options
1. **Show Data Labels** (toggle) - Default: true
2. **Label Format** (dropdown) - Only shown when labels enabled
   - Percentage only (default)
   - Value only
   - Name + Percentage
   - Name + Value
3. **Show Legend** (toggle) - Default: true
4. **Legend Position** (dropdown) - Right, Bottom, Left, Top

### Line Chart Specific Configuration

#### Basic Configuration
1. **Line Style** (radio buttons)
   - Straight Lines (default)
   - Smooth Curves

2. **Extra Dimension** (optional dropdown)
   - Allows adding one additional dimension for multiple lines

#### Customization Options
1. **Show Data Labels** (toggle) - Default: false
2. **Show Data Points** (toggle) - Default: true
3. **X-Axis Title** (text input) - Default: Column name
4. **Y-Axis Title** (text input) - Default: Column name
5. **Show Legend** (toggle) - Default: true (when extra dimension exists)

## Detailed Implementation Plan

### Phase 1: Backend Infrastructure

#### 1.1 Database Schema Updates

**File:** `DDP_backend/ddpui/models/visualization.py`

The existing `Chart` model already supports our requirements with:
- `id`, `title`, `description` fields
- `chart_type` field (currently limited to 'echarts')
- `config` JSONField for storing ECharts configuration
- `user` and `org` for access control

**Required changes:**
1. Update `chart_type` choices to include: 'bar', 'pie', 'line'
2. Ensure `config` field can store complete ECharts configurations

**Migration file:** `DDP_backend/ddpui/migrations/00XX_update_chart_types.py`
```python
from django.db import migrations

def update_chart_types(apps, schema_editor):
    # Update existing 'echarts' type to specific chart types if needed
    pass

class Migration(migrations.Migration):
    dependencies = [
        ('ddpui', 'previous_migration'),
    ]
    operations = [
        migrations.RunPython(update_chart_types),
    ]
```

#### 1.2 Query Builder Enhancement

**File:** `DDP_backend/ddpui/datainsights/query_builder.py`

The `AggQueryBuilder` class needs to properly support aggregation functions:

```python
from sqlalchemy import func, select

class AggQueryBuilder:
    def add_aggregate_column(self, column_name: str, agg_func: str, alias: str = None):
        """Add an aggregate column with specified function"""
        column = self.table.c[column_name]
        
        agg_functions = {
            'sum': func.sum,
            'avg': func.avg,
            'count': func.count,
            'min': func.min,
            'max': func.max,
            'count_distinct': func.count(func.distinct(column))
        }
        
        if agg_func.lower() in agg_functions:
            agg_column = agg_functions[agg_func.lower()](column)
            if alias:
                agg_column = agg_column.label(alias)
            self.select_query = self.select_query.add_columns(agg_column)
        
        return self
```

#### 1.3 ECharts Configuration Generator

**New File:** `DDP_backend/ddpui/core/echarts_config_generator.py`

```python
from typing import Dict, List, Any, Optional

class EChartsConfigGenerator:
    """Generate ECharts configurations based on chart type and data"""
    
    @staticmethod
    def generate_bar_config(
        data: Dict[str, Any], 
        customizations: Dict[str, Any] = None
    ) -> Dict:
        """Generate bar chart configuration"""
        customizations = customizations or {}
        orientation = customizations.get('orientation', 'vertical')
        is_stacked = customizations.get('stacked', False)
        show_data_labels = customizations.get('showDataLabels', False)
        x_axis_title = customizations.get('xAxisTitle', '')
        y_axis_title = customizations.get('yAxisTitle', '')
        
        config = {
            'title': {'text': customizations.get('title', '')},
            'tooltip': {
                'trigger': 'axis',
                'axisPointer': {'type': 'shadow'}
            },
            'legend': {'data': data.get('legend', [])},
            'grid': {
                'left': '3%',
                'right': '4%',
                'bottom': '3%',
                'containLabel': True
            },
            'xAxis': {
                'type': 'category' if orientation == 'vertical' else 'value',
                'data': data.get('xAxisData', []) if orientation == 'vertical' else None,
                'name': x_axis_title
            },
            'yAxis': {
                'type': 'value' if orientation == 'vertical' else 'category',
                'data': data.get('yAxisData', []) if orientation == 'horizontal' else None,
                'name': y_axis_title
            },
            'series': []
        }
        
        # Build series
        for series_data in data.get('series', []):
            series_config = {
                'name': series_data.get('name', ''),
                'type': 'bar',
                'data': series_data.get('data', []),
                'label': {
                    'show': show_data_labels,
                    'position': 'top' if orientation == 'vertical' else 'right'
                }
            }
            if is_stacked:
                series_config['stack'] = 'total'
            config['series'].append(series_config)
        
        return config
    
    @staticmethod
    def generate_pie_config(
        data: Dict[str, Any], 
        customizations: Dict[str, Any] = None
    ) -> Dict:
        """Generate pie chart configuration"""
        customizations = customizations or {}
        chart_style = customizations.get('chartStyle', 'pie')
        show_data_labels = customizations.get('showDataLabels', True)
        label_format = customizations.get('labelFormat', 'percentage')
        legend_position = customizations.get('legendPosition', 'right')
        
        # Determine label formatter
        formatter_map = {
            'percentage': '{b}: {d}%',
            'value': '{b}: {c}',
            'name_percentage': '{b}\n{d}%',
            'name_value': '{b}\n{c}'
        }
        
        config = {
            'title': {'text': customizations.get('title', '')},
            'tooltip': {
                'trigger': 'item',
                'formatter': '{a} <br/>{b}: {c} ({d}%)'
            },
            'legend': {
                'orient': 'vertical' if legend_position in ['left', 'right'] else 'horizontal',
                legend_position: 10 if legend_position in ['left', 'right'] else 'center',
                'data': [item['name'] for item in data.get('pieData', [])]
            },
            'series': [{
                'name': data.get('seriesName', 'Data'),
                'type': 'pie',
                'radius': ['40%', '70%'] if chart_style == 'donut' else '70%',
                'avoidLabelOverlap': False,
                'label': {
                    'show': show_data_labels,
                    'formatter': formatter_map.get(label_format, '{b}: {d}%')
                },
                'labelLine': {
                    'show': show_data_labels
                },
                'data': data.get('pieData', [])
            }]
        }
        
        return config
    
    @staticmethod
    def generate_line_config(
        data: Dict[str, Any], 
        customizations: Dict[str, Any] = None
    ) -> Dict:
        """Generate line chart configuration"""
        customizations = customizations or {}
        line_style = customizations.get('lineStyle', 'straight')
        show_data_labels = customizations.get('showDataLabels', False)
        show_data_points = customizations.get('showDataPoints', True)
        x_axis_title = customizations.get('xAxisTitle', '')
        y_axis_title = customizations.get('yAxisTitle', '')
        
        config = {
            'title': {'text': customizations.get('title', '')},
            'tooltip': {'trigger': 'axis'},
            'legend': {'data': data.get('legend', [])},
            'grid': {
                'left': '3%',
                'right': '4%',
                'bottom': '3%',
                'containLabel': True
            },
            'xAxis': {
                'type': 'category',
                'data': data.get('xAxisData', []),
                'name': x_axis_title,
                'boundaryGap': False
            },
            'yAxis': {
                'type': 'value',
                'name': y_axis_title
            },
            'series': []
        }
        
        # Build series
        for series_data in data.get('series', []):
            series_config = {
                'name': series_data.get('name', ''),
                'type': 'line',
                'smooth': line_style == 'smooth',
                'data': series_data.get('data', []),
                'label': {
                    'show': show_data_labels,
                    'position': 'top'
                },
                'showSymbol': show_data_points
            }
            config['series'].append(series_config)
        
        return config
```

#### 1.4 Enhanced Chart Data API

**File:** `DDP_backend/ddpui/api/charts_api.py`

Add new endpoints:

```python
from typing import Optional
from ninja import Schema
from ddpui.core.echarts_config_generator import EChartsConfigGenerator

class ChartDataPayload(Schema):
    chart_type: str
    computation_type: str
    schema_name: str
    table: str
    
    # For raw data
    xAxis: Optional[str] = None
    yAxis: Optional[str] = None
    
    # For aggregated data
    dimension_col: Optional[str] = None
    aggregate_col: Optional[str] = None
    aggregate_func: Optional[str] = None
    extra_dimension: Optional[str] = None
    
    # Customizations
    customizations: Optional[dict] = None
    
    # Pagination
    offset: int = 0
    limit: int = 100

class ChartDataResponse(Schema):
    data: dict
    echarts_config: dict

@charts_router.post("/chart-data/", response_model=ChartDataResponse)
def get_chart_data(request, payload: ChartDataPayload, orguser: OrgUser = Depends(org_user_auth)):
    """Get chart data with ECharts configuration"""
    
    # Validate user has access to schema/table
    if not has_schema_access(orguser, payload.schema_name):
        raise HttpError(403, "Access denied to schema")
    
    # Build query based on computation type
    query_builder = AggQueryBuilder()
    warehouse = get_warehouse_for_org(orguser.org)
    
    if payload.computation_type == 'raw':
        # Build raw data query
        query_builder.fetch_from(f"{payload.schema_name}.{payload.table}")
        query_builder.add_column(payload.xAxis)
        query_builder.add_column(payload.yAxis)
        
        if payload.extra_dimension:
            query_builder.add_column(payload.extra_dimension)
            
    else:  # aggregated
        # Build aggregated query
        query_builder.fetch_from(f"{payload.schema_name}.{payload.table}")
        query_builder.add_column(payload.dimension_col)
        query_builder.add_aggregate_column(
            payload.aggregate_col, 
            payload.aggregate_func,
            f"{payload.aggregate_func}_{payload.aggregate_col}"
        )
        query_builder.group_cols_by([payload.dimension_col])
        
        if payload.extra_dimension:
            query_builder.add_column(payload.extra_dimension)
            query_builder.group_cols_by([payload.dimension_col, payload.extra_dimension])
    
    # Add pagination
    query_builder.limit_rows(payload.limit)
    query_builder.offset_rows(payload.offset)
    
    # Execute query
    sql = query_builder.build()
    results = warehouse.execute(sql)
    
    # Transform data for chart
    chart_data = transform_data_for_chart(
        results, 
        payload.chart_type,
        payload.computation_type,
        payload
    )
    
    # Generate ECharts config
    config_generator = {
        'bar': EChartsConfigGenerator.generate_bar_config,
        'pie': EChartsConfigGenerator.generate_pie_config,
        'line': EChartsConfigGenerator.generate_line_config
    }
    
    echarts_config = config_generator[payload.chart_type](
        chart_data,
        payload.customizations
    )
    
    return ChartDataResponse(
        data=chart_data,
        echarts_config=echarts_config
    )

@charts_router.post("/chart-data-preview/", response_model=DataPreviewResponse)
def get_chart_data_preview(request, payload: ChartDataPayload, orguser: OrgUser = Depends(org_user_auth)):
    """Get paginated data preview for chart"""
    # Similar implementation but return tabular format
    # Include column names and data types
    pass

def transform_data_for_chart(results, chart_type, computation_type, payload):
    """Transform query results to chart-specific data format"""
    
    if chart_type == 'bar':
        if computation_type == 'raw':
            return {
                'xAxisData': [row[payload.xAxis] for row in results],
                'series': [{
                    'name': payload.yAxis,
                    'data': [row[payload.yAxis] for row in results]
                }],
                'legend': [payload.yAxis]
            }
        else:  # aggregated
            if payload.extra_dimension:
                # Group by extra dimension
                grouped_data = {}
                for row in results:
                    dimension = row[payload.extra_dimension]
                    if dimension not in grouped_data:
                        grouped_data[dimension] = []
                    grouped_data[dimension].append(row)
                
                return {
                    'xAxisData': list(set(row[payload.dimension_col] for row in results)),
                    'series': [
                        {
                            'name': dimension,
                            'data': [row.get(f"{payload.aggregate_func}_{payload.aggregate_col}", 0) 
                                    for row in data]
                        }
                        for dimension, data in grouped_data.items()
                    ],
                    'legend': list(grouped_data.keys())
                }
            else:
                return {
                    'xAxisData': [row[payload.dimension_col] for row in results],
                    'series': [{
                        'name': f"{payload.aggregate_func}({payload.aggregate_col})",
                        'data': [row[f"{payload.aggregate_func}_{payload.aggregate_col}"] 
                                for row in results]
                    }],
                    'legend': [f"{payload.aggregate_func}({payload.aggregate_col})"]
                }
    
    elif chart_type == 'pie':
        if computation_type == 'raw':
            # For raw data, count occurrences
            value_counts = {}
            for row in results:
                key = row[payload.xAxis]
                value_counts[key] = value_counts.get(key, 0) + 1
            
            return {
                'pieData': [
                    {'value': count, 'name': str(name)}
                    for name, count in value_counts.items()
                ],
                'seriesName': payload.xAxis
            }
        else:  # aggregated
            return {
                'pieData': [
                    {
                        'value': row[f"{payload.aggregate_func}_{payload.aggregate_col}"],
                        'name': str(row[payload.dimension_col])
                    }
                    for row in results
                ],
                'seriesName': f"{payload.aggregate_func}({payload.aggregate_col})"
            }
    
    elif chart_type == 'line':
        # Similar transformation logic for line charts
        pass
    
    return {}
```

### Phase 2: Frontend Implementation

#### 2.1 Chart Listing Page

**File:** `webapp_v2/app/charts/page.tsx`

```typescript
'use client';

import { useState } from 'react';
import { useCharts } from '@/hooks/api/useChart';
import { ChartCard } from '@/components/charts/ChartCard';
import { SearchBar } from '@/components/ui/search-bar';
import { Button } from '@/components/ui/button';
import { Plus } from 'lucide-react';
import Link from 'next/link';

export default function ChartsPage() {
  const [searchTerm, setSearchTerm] = useState('');
  const { data: charts, isLoading, error } = useCharts();
  
  const filteredCharts = charts?.filter(chart => 
    chart.title.toLowerCase().includes(searchTerm.toLowerCase()) ||
    chart.description?.toLowerCase().includes(searchTerm.toLowerCase())
  );
  
  return (
    <div className="container mx-auto p-6">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-2xl font-bold">Charts</h1>
        <Link href="/charts/new">
          <Button>
            <Plus className="mr-2 h-4 w-4" />
            Create Chart
          </Button>
        </Link>
      </div>
      
      <SearchBar 
        value={searchTerm}
        onChange={setSearchTerm}
        placeholder="Search charts..."
        className="mb-6"
      />
      
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {filteredCharts?.map(chart => (
          <ChartCard key={chart.id} chart={chart} />
        ))}
      </div>
    </div>
  );
}
```

#### 2.2 Enhanced Chart Builder

**File:** `webapp_v2/components/charts/ChartBuilder.tsx` (Enhanced version)

The existing ChartBuilder needs significant enhancements:

1. Replace Recharts with ECharts
2. Add dynamic customization panel based on chart type
3. Implement proper form field structure per chart type
4. Add data preview table alongside chart preview

Key sections to modify:

```typescript
// Add chart-specific customization components
const ChartCustomizations = ({ chartType, formData, onChange }) => {
  switch (chartType) {
    case 'bar':
      return <BarChartCustomizations formData={formData} onChange={onChange} />;
    case 'pie':
      return <PieChartCustomizations formData={formData} onChange={onChange} />;
    case 'line':
      return <LineChartCustomizations formData={formData} onChange={onChange} />;
    default:
      return null;
  }
};

// Bar chart customizations component
const BarChartCustomizations = ({ formData, onChange }) => {
  return (
    <div className="space-y-4">
      <div>
        <Label>Orientation</Label>
        <RadioGroup
          value={formData.customizations?.orientation || 'vertical'}
          onValueChange={(value) => 
            onChange({ 
              ...formData, 
              customizations: { ...formData.customizations, orientation: value } 
            })
          }
        >
          <div className="flex items-center space-x-2">
            <RadioGroupItem value="vertical" id="vertical" />
            <Label htmlFor="vertical">Vertical</Label>
          </div>
          <div className="flex items-center space-x-2">
            <RadioGroupItem value="horizontal" id="horizontal" />
            <Label htmlFor="horizontal">Horizontal</Label>
          </div>
        </RadioGroup>
      </div>
      
      {formData.extra_dimension && (
        <div className="flex items-center space-x-2">
          <Checkbox
            id="stacked"
            checked={formData.customizations?.stacked || false}
            onCheckedChange={(checked) =>
              onChange({
                ...formData,
                customizations: { ...formData.customizations, stacked: checked }
              })
            }
          />
          <Label htmlFor="stacked">Stacked Bars</Label>
        </div>
      )}
      
      <div className="flex items-center space-x-2">
        <Checkbox
          id="showDataLabels"
          checked={formData.customizations?.showDataLabels || false}
          onCheckedChange={(checked) =>
            onChange({
              ...formData,
              customizations: { ...formData.customizations, showDataLabels: checked }
            })
          }
        />
        <Label htmlFor="showDataLabels">Show Data Labels</Label>
      </div>
      
      <div>
        <Label htmlFor="xAxisTitle">X-Axis Title</Label>
        <Input
          id="xAxisTitle"
          value={formData.customizations?.xAxisTitle || ''}
          onChange={(e) =>
            onChange({
              ...formData,
              customizations: { ...formData.customizations, xAxisTitle: e.target.value }
            })
          }
          placeholder="Enter X-axis title"
        />
      </div>
      
      <div>
        <Label htmlFor="yAxisTitle">Y-Axis Title</Label>
        <Input
          id="yAxisTitle"
          value={formData.customizations?.yAxisTitle || ''}
          onChange={(e) =>
            onChange({
              ...formData,
              customizations: { ...formData.customizations, yAxisTitle: e.target.value }
            })
          }
          placeholder="Enter Y-axis title"
        />
      </div>
    </div>
  );
};
```

#### 2.3 ECharts Integration

**File:** `webapp_v2/components/charts/EChartsWrapper.tsx`

```typescript
'use client';

import React, { useEffect, useRef } from 'react';
import * as echarts from 'echarts/core';
import { BarChart, LineChart, PieChart } from 'echarts/charts';
import {
  TitleComponent,
  TooltipComponent,
  LegendComponent,
  GridComponent,
  DatasetComponent
} from 'echarts/components';
import { CanvasRenderer } from 'echarts/renderers';

// Register required components
echarts.use([
  BarChart,
  LineChart,
  PieChart,
  TitleComponent,
  TooltipComponent,
  LegendComponent,
  GridComponent,
  DatasetComponent,
  CanvasRenderer
]);

interface EChartsWrapperProps {
  option: echarts.EChartsOption;
  style?: React.CSSProperties;
  className?: string;
  onChartReady?: (chart: echarts.ECharts) => void;
}

export const EChartsWrapper = React.forwardRef<echarts.ECharts, EChartsWrapperProps>(
  ({ option, style, className, onChartReady }, ref) => {
    const chartRef = useRef<HTMLDivElement>(null);
    const chartInstance = useRef<echarts.ECharts>();
    
    useEffect(() => {
      if (chartRef.current) {
        chartInstance.current = echarts.init(chartRef.current);
        chartInstance.current.setOption(option);
        
        if (ref) {
          (ref as React.MutableRefObject<echarts.ECharts>).current = chartInstance.current;
        }
        
        if (onChartReady) {
          onChartReady(chartInstance.current);
        }
        
        const handleResize = () => {
          chartInstance.current?.resize();
        };
        
        window.addEventListener('resize', handleResize);
        
        return () => {
          window.removeEventListener('resize', handleResize);
          chartInstance.current?.dispose();
        };
      }
    }, []);
    
    useEffect(() => {
      chartInstance.current?.setOption(option);
    }, [option]);
    
    return <div ref={chartRef} style={style} className={className} />;
  }
);

EChartsWrapper.displayName = 'EChartsWrapper';
```

#### 2.4 Data Preview Component

**File:** `webapp_v2/components/charts/DataPreview.tsx`

```typescript
interface DataPreviewProps {
  data: any[];
  columns: string[];
  isLoading: boolean;
  error?: Error;
  pagination?: {
    page: number;
    pageSize: number;
    total: number;
    onPageChange: (page: number) => void;
  };
}

export function DataPreview({ data, columns, isLoading, error, pagination }: DataPreviewProps) {
  if (isLoading) return <LoadingState />;
  if (error) return <ErrorState error={error} />;
  if (!data || data.length === 0) return <EmptyState message="No data available" />;
  
  return (
    <div className="space-y-4">
      <div className="overflow-x-auto border rounded-lg">
        <table className="min-w-full divide-y divide-gray-200">
          <thead className="bg-gray-50">
            <tr>
              {columns.map(col => (
                <th 
                  key={col} 
                  className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider"
                >
                  {col}
                </th>
              ))}
            </tr>
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {data.map((row, idx) => (
              <tr key={idx} className="hover:bg-gray-50">
                {columns.map(col => (
                  <td key={col} className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                    {formatCellValue(row[col])}
                  </td>
                ))}
              </tr>
            ))}
          </tbody>
        </table>
      </div>
      
      {pagination && (
        <div className="flex items-center justify-between">
          <div className="text-sm text-gray-700">
            Showing {((pagination.page - 1) * pagination.pageSize) + 1} to{' '}
            {Math.min(pagination.page * pagination.pageSize, pagination.total)} of{' '}
            {pagination.total} results
          </div>
          <div className="flex gap-2">
            <Button
              variant="outline"
              size="sm"
              onClick={() => pagination.onPageChange(pagination.page - 1)}
              disabled={pagination.page === 1}
            >
              Previous
            </Button>
            <Button
              variant="outline"
              size="sm"
              onClick={() => pagination.onPageChange(pagination.page + 1)}
              disabled={pagination.page * pagination.pageSize >= pagination.total}
            >
              Next
            </Button>
          </div>
        </div>
      )}
    </div>
  );
}

function formatCellValue(value: any): string {
  if (value === null || value === undefined) return '-';
  if (typeof value === 'number') return value.toLocaleString();
  if (value instanceof Date) return value.toLocaleDateString();
  return String(value);
}
```

### Phase 3: Integration and Testing

#### 3.1 API Hooks

**File:** `webapp_v2/hooks/api/useChart.ts`

Add new hooks:

```typescript
export interface ChartDataPayload {
  chart_type: string;
  computation_type: 'raw' | 'aggregated';
  schema_name: string;
  table: string;
  
  // For raw data
  xAxis?: string;
  yAxis?: string;
  
  // For aggregated data
  dimension_col?: string;
  aggregate_col?: string;
  aggregate_func?: string;
  extra_dimension?: string;
  
  // Customizations
  customizations?: Record<string, any>;
  
  // Pagination
  offset?: number;
  limit?: number;
}

export function useChartData(payload: ChartDataPayload | null) {
  return useSWR(
    payload ? ['/api/charts/chart-data/', payload] : null,
    ([url, data]) => api.post(url, data),
    {
      revalidateOnFocus: false,
      revalidateOnReconnect: false,
      dedupingInterval: 2000,
    }
  );
}

export function useChartDataPreview(payload: ChartDataPayload | null, page: number = 1, pageSize: number = 50) {
  const enrichedPayload = payload ? {
    ...payload,
    offset: (page - 1) * pageSize,
    limit: pageSize
  } : null;
  
  return useSWR(
    enrichedPayload ? ['/api/charts/chart-data-preview/', enrichedPayload] : null,
    ([url, data]) => api.post(url, data)
  );
}
```

#### 3.2 Testing Strategy

##### Backend Tests

**File:** `DDP_backend/ddpui/tests/test_charts_api.py`

```python
import pytest
from django.test import TestCase
from ddpui.models.org import Org, OrgUser
from ddpui.models.visualization import Chart

class TestChartsAPI(TestCase):
    def setUp(self):
        self.org = Org.objects.create(name="Test Org")
        self.user = OrgUser.objects.create(
            email="test@example.com",
            org=self.org
        )
        
    def test_create_bar_chart(self):
        """Test chart creation with bar type"""
        payload = {
            "chart_type": "bar",
            "computation_type": "aggregated",
            "schema_name": "public",
            "table": "sales",
            "dimension_col": "region",
            "aggregate_col": "revenue",
            "aggregate_func": "sum",
            "customizations": {
                "orientation": "vertical",
                "showDataLabels": True
            }
        }
        response = self.client.post("/api/charts/chart-data/", payload)
        self.assertEqual(response.status_code, 200)
        self.assertIn("echarts_config", response.json())
        
    def test_chart_data_aggregation(self):
        """Test data aggregation for charts"""
        # Test various aggregation functions
        pass
    
    def test_echarts_config_generation(self):
        """Test ECharts config generation"""
        from ddpui.core.echarts_config_generator import EChartsConfigGenerator
        
        data = {
            'xAxisData': ['Q1', 'Q2', 'Q3', 'Q4'],
            'series': [{
                'name': 'Revenue',
                'data': [100, 200, 150, 300]
            }],
            'legend': ['Revenue']
        }
        
        config = EChartsConfigGenerator.generate_bar_config(data, {
            'title': 'Quarterly Revenue',
            'orientation': 'vertical',
            'showDataLabels': True
        })
        
        self.assertEqual(config['title']['text'], 'Quarterly Revenue')
        self.assertEqual(config['xAxis']['type'], 'category')
        self.assertEqual(config['series'][0]['label']['show'], True)
    
    def test_chart_data_preview(self):
        """Test data preview endpoint"""
        payload = {
            "chart_type": "bar",
            "computation_type": "raw",
            "schema_name": "public",
            "table": "sales",
            "xAxis": "date",
            "yAxis": "amount",
            "offset": 0,
            "limit": 10
        }
        response = self.client.post("/api/charts/chart-data-preview/", payload)
        self.assertEqual(response.status_code, 200)
        self.assertIn("columns", response.json())
        self.assertIn("data", response.json())
```

##### Frontend Tests

**File:** `webapp_v2/components/charts/__tests__/ChartBuilder.test.tsx`

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import ChartBuilder from '../ChartBuilder';
import { useSchemas, useTables, useColumns } from '@/hooks/api/useChart';

// Mock the hooks
jest.mock('@/hooks/api/useChart');

describe('ChartBuilder', () => {
  beforeEach(() => {
    (useSchemas as jest.Mock).mockReturnValue({
      data: ['public', 'analytics'],
      isLoading: false,
      error: null
    });
    
    (useTables as jest.Mock).mockReturnValue({
      data: [{ name: 'sales' }, { name: 'customers' }],
      isLoading: false,
      error: null
    });
    
    (useColumns as jest.Mock).mockReturnValue({
      data: [
        { name: 'id', data_type: 'integer' },
        { name: 'amount', data_type: 'numeric' },
        { name: 'region', data_type: 'text' }
      ],
      isLoading: false,
      error: null
    });
  });
  
  it('renders chart type selection', () => {
    render(<ChartBuilder onSave={jest.fn()} />);
    
    expect(screen.getByText('Bar Chart')).toBeInTheDocument();
    expect(screen.getByText('Pie Chart')).toBeInTheDocument();
    expect(screen.getByText('Line Chart')).toBeInTheDocument();
  });
  
  it('shows customization options for bar chart', async () => {
    render(<ChartBuilder onSave={jest.fn()} />);
    
    // Select bar chart
    fireEvent.click(screen.getByText('Bar Chart'));
    
    // Should show bar-specific options
    await waitFor(() => {
      expect(screen.getByText('Orientation')).toBeInTheDocument();
    });
  });
  
  it('generates chart preview', async () => {
    render(<ChartBuilder onSave={jest.fn()} />);
    
    // Select chart type and fill required fields
    fireEvent.click(screen.getByText('Bar Chart'));
    
    // Select schema and table
    // ... additional test implementation
  });
  
  it('saves chart configuration', async () => {
    const onSave = jest.fn();
    render(<ChartBuilder onSave={onSave} />);
    
    // Fill out the form
    // ... 
    
    // Click save
    fireEvent.click(screen.getByText('Save Chart'));
    
    await waitFor(() => {
      expect(onSave).toHaveBeenCalledWith(
        expect.objectContaining({
          chart_type: 'bar',
          title: expect.any(String)
        })
      );
    });
  });
});
```

### Phase 4: Form Flow Implementation

#### Detailed Form Flow Components

**File:** `webapp_v2/components/charts/ChartBuilderSteps.tsx`

```typescript
interface ChartBuilderStep {
  id: string;
  title: string;
  description: string;
  isComplete: (formData: ChartFormData) => boolean;
}

export const chartBuilderSteps: ChartBuilderStep[] = [
  {
    id: 'chart-type',
    title: 'Select Chart Type',
    description: 'Choose the type of visualization',
    isComplete: (data) => !!data.chart_type
  },
  {
    id: 'data-source',
    title: 'Configure Data Source',
    description: 'Select schema and table',
    isComplete: (data) => !!data.schema_name && !!data.table
  },
  {
    id: 'columns',
    title: 'Map Columns',
    description: 'Select data columns for your chart',
    isComplete: (data) => {
      if (data.computation_type === 'raw') {
        return !!data.xAxis && !!data.yAxis;
      } else {
        return !!data.dimension_col && !!data.aggregate_col && !!data.aggregate_func;
      }
    }
  },
  {
    id: 'customize',
    title: 'Customize Chart',
    description: 'Configure chart appearance',
    isComplete: () => true // Optional step
  },
  {
    id: 'metadata',
    title: 'Add Details',
    description: 'Name and describe your chart',
    isComplete: (data) => !!data.title
  }
];
```

## Critical Implementation Notes

### Backend Considerations

1. **Query Performance**: 
   - Implement query result caching using Redis
   - Add query timeout limits (30 seconds default)
   - Limit result sets to 10,000 rows for initial load

2. **Security**:
   - Validate user has access to requested schema/table through org membership
   - Use SQLAlchemy parameterized queries to prevent SQL injection
   - Implement rate limiting on chart data endpoints

3. **Error Handling**:
   - Return specific error codes for different failure scenarios
   - Log all query failures with context for debugging
   - Provide user-friendly error messages

### Frontend Considerations

1. **ECharts Bundle Size**:
   - Use dynamic imports for chart components
   - Only import required ECharts modules
   - Consider using ECharts CDN for production with local fallback

2. **User Experience**:
   - Show skeleton loaders during data fetching
   - Debounce form changes by 500ms before fetching new data
   - Auto-save chart drafts to localStorage
   - Provide keyboard shortcuts for common actions

3. **Performance**:
   - Implement virtual scrolling for data preview tables > 100 rows
   - Use React.memo for chart customization components
   - Throttle chart resize events

## Testing Checklist

### Unit Tests
- [ ] Query builder aggregation functions for all supported functions
- [ ] ECharts config generation for each chart type and customization
- [ ] Data transformation logic for all chart type/computation combinations
- [ ] API endpoint validation and error handling

### Integration Tests
- [ ] End-to-end chart creation flow for all chart types
- [ ] Data preview with pagination and column sorting
- [ ] Chart customization persistence across sessions
- [ ] Multi-user access control and org isolation

### Frontend Tests
- [ ] Component rendering for all chart types
- [ ] Form validation and error states
- [ ] Responsive design on mobile/tablet/desktop
- [ ] Accessibility compliance (WCAG 2.1 AA)

## Documentation Updates

1. **API Documentation**: Update OpenAPI specs for new endpoints
2. **User Guide**: Create step-by-step chart builder tutorial with screenshots
3. **Developer Guide**: Document ECharts configuration patterns and customization hooks

## External Resources

### Apache ECharts Documentation
- Main docs: https://echarts.apache.org/en/option.html
- Examples gallery: https://echarts.apache.org/examples/en/index.html
- API reference: https://echarts.apache.org/en/api.html

### ECharts React Integration
- echarts-for-react: https://github.com/hustcc/echarts-for-react
- Best practices: https://dev.to/manufac/using-apache-echarts-with-react-and-typescript-353k
- Bundle optimization: https://dev.to/manufac/using-apache-echarts-with-react-and-typescript-optimizing-bundle-size-29l8

### Query Builder References
- SQLAlchemy docs: https://docs.sqlalchemy.org/en/20/core/selectable.html
- Aggregation functions: https://docs.sqlalchemy.org/en/20/core/functions.html

## Success Criteria

1. **Functional Requirements**:
   - All three chart types (bar, pie, line) working with specified customizations
   - Live preview updates within 1 second of configuration changes
   - Data preview shows paginated query results with proper formatting
   - Charts persist with all customizations and can be edited

2. **Performance Requirements**:
   - Chart renders within 2 seconds for datasets < 10k rows
   - Configuration changes reflect in < 500ms
   - No memory leaks after 100+ chart preview updates
   - Page load time < 3 seconds on 3G connection

3. **Quality Requirements**:
   - All tests passing with > 80% code coverage
   - No console errors or warnings in production build
   - Responsive design works on devices >= 320px width
   - Accessibility score > 90 in Lighthouse

## Implementation Confidence Score: 9/10

This comprehensive plan provides:
- Complete form field specifications for each chart type
- Detailed implementation steps with code examples
- Full API contract definitions
- Extensive testing strategies
- Performance optimization guidelines
- Security considerations

The plan addresses all requirements from the spec with clear, actionable implementation details that enable one-pass development success.