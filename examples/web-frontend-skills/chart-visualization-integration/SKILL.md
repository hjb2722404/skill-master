---
name: chart-visualization-integration
description: >
  本技能用于实现图表可视化集成，包括ECharts/D3集成、响应式图表、大数据可视化等。
  当需要集成图表库、实现数据可视化、处理大数据图表时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 图表可视化集成

## 一、概述

### 1.1 这是什么

图表可视化集成技能提供完整的图表解决方案，涵盖ECharts/D3等库的集成、响应式图表、大数据可视化等。

### 1.2 适用场景

- ✅ 数据可视化展示
- ✅ 仪表盘/报表
- ✅ 实时数据图表
- ✅ 大数据量图表
- ✅ 交互式图表
- ❌ 简单静态图表（使用图片即可）
- ❌ 高度定制化图形（需要专业设计）

### 1.3 核心原则

1. **按需加载** —— 图表库较大，需要懒加载
2. **响应式适配** —— 自动适应容器尺寸
3. **性能优先** —— 大数据量时优化渲染
4. **可访问性** —— 提供替代文本

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要展示数据图表
- 需要集成图表库
- 需要响应式图表
- 需要大数据量图表
- 需要实时更新图表

### 2.2 输入

- 图表类型需求
- 数据源格式
- 交互需求
- 性能要求

### 2.3 输出

- 图表组件实现
- 图表配置
- 数据处理逻辑
- 响应式适配

---

## 三、ECharts集成

### 3.1 基础封装

```typescript
import { useEffect, useRef, useCallback } from 'react';
import * as echarts from 'echarts/core';
import { LineChart, BarChart, PieChart } from 'echarts/charts';
import {
  GridComponent,
  TooltipComponent,
  LegendComponent,
  TitleComponent,
} from 'echarts/components';
import { CanvasRenderer } from 'echarts/renderers';

// 按需引入
echarts.use([
  LineChart,
  BarChart,
  PieChart,
  GridComponent,
  TooltipComponent,
  LegendComponent,
  TitleComponent,
  CanvasRenderer,
]);

interface EChartsProps {
  option: echarts.EChartsOption;
  style?: React.CSSProperties;
  onClick?: (params: any) => void;
  onResize?: (instance: echarts.ECharts) => void;
}

export function EChartsComponent({
  option,
  style,
  onClick,
  onResize,
}: EChartsProps) {
  const chartRef = useRef<HTMLDivElement>(null);
  const chartInstance = useRef<echarts.ECharts>();

  // 初始化图表
  useEffect(() => {
    if (!chartRef.current) return;

    chartInstance.current = echarts.init(chartRef.current);

    if (onClick) {
      chartInstance.current.on('click', onClick);
    }

    return () => {
      chartInstance.current?.dispose();
    };
  }, []);

  // 更新配置
  useEffect(() => {
    chartInstance.current?.setOption(option, true);
  }, [option]);

  // 响应式
  useEffect(() => {
    const handleResize = () => {
      chartInstance.current?.resize();
      onResize?.(chartInstance.current!);
    };

    window.addEventListener('resize', handleResize);
    return () => window.removeEventListener('resize', handleResize);
  }, [onResize]);

  return (
    <div
      ref={chartRef}
      style={{ width: '100%', height: '400px', ...style }}
    />
  );
}
```

### 3.2 常用图表封装

```typescript
// 折线图
interface LineChartProps {
  data: { name: string; value: number }[];
  xAxisData: string[];
  title?: string;
}

export function LineChartComponent({ data, xAxisData, title }: LineChartProps) {
  const option: echarts.EChartsOption = {
    title: title ? { text: title } : undefined,
    tooltip: { trigger: 'axis' },
    xAxis: {
      type: 'category',
      data: xAxisData,
    },
    yAxis: { type: 'value' },
    series: [
      {
        type: 'line',
        data: data.map((d) => d.value),
        smooth: true,
        areaStyle: { opacity: 0.1 },
      },
    ],
  };

  return <EChartsComponent option={option} />;
}

// 柱状图
interface BarChartProps {
  data: { name: string; value: number }[];
  title?: string;
}

export function BarChartComponent({ data, title }: BarChartProps) {
  const option: echarts.EChartsOption = {
    title: title ? { text: title } : undefined,
    tooltip: { trigger: 'axis' },
    xAxis: {
      type: 'category',
      data: data.map((d) => d.name),
    },
    yAxis: { type: 'value' },
    series: [
      {
        type: 'bar',
        data: data.map((d) => d.value),
      },
    ],
  };

  return <EChartsComponent option={option} />;
}

// 饼图
interface PieChartProps {
  data: { name: string; value: number }[];
  title?: string;
}

export function PieChartComponent({ data, title }: PieChartProps) {
  const option: echarts.EChartsOption = {
    title: title ? { text: title } : undefined,
    tooltip: { trigger: 'item' },
    legend: { orient: 'vertical', left: 'left' },
    series: [
      {
        type: 'pie',
        radius: ['40%', '70%'],
        avoidLabelOverlap: false,
        itemStyle: {
          borderRadius: 10,
          borderColor: '#fff',
          borderWidth: 2,
        },
        label: { show: false },
        emphasis: {
          label: { show: true, fontSize: 20, fontWeight: 'bold' },
        },
        data,
      },
    ],
  };

  return <EChartsComponent option={option} />;
}
```

---

## 四、大数据优化

```typescript
interface LargeDataChartProps {
  data: [number, number][]; // [x, y] 数据点
  dataLimit?: number;
}

export function LargeDataChart({ data, dataLimit = 10000 }: LargeDataChartProps) {
  // 数据采样
  const sampledData = useMemo(() => {
    if (data.length <= dataLimit) return data;

    const step = Math.ceil(data.length / dataLimit);
    return data.filter((_, index) => index % step === 0);
  }, [data, dataLimit]);

  const option: echarts.EChartsOption = {
    tooltip: { trigger: 'axis' },
    dataZoom: [
      { type: 'inside', start: 0, end: 100 },
      { start: 0, end: 100 },
    ],
    xAxis: { type: 'value' },
    yAxis: { type: 'value' },
    series: [
      {
        type: 'line',
        data: sampledData,
        large: true, // 启用大数据优化
        largeThreshold: 500,
        sampling: 'lttb', //  Largest Triangle Three Bucket 采样算法
        showSymbol: false,
        lineStyle: { width: 1 },
      },
    ],
  };

  return <EChartsComponent option={option} />;
}
```

---

## 五、实时数据图表

```typescript
export function RealtimeChart() {
  const [data, setData] = useState<{ time: string; value: number }[]>([]);
  const chartRef = useRef<echarts.ECharts>();

  useEffect(() => {
    // 模拟实时数据
    const interval = setInterval(() => {
      const now = new Date();
      const time = now.toLocaleTimeString();
      const value = Math.random() * 100;

      setData((prev) => {
        const newData = [...prev, { time, value }];
        if (newData.length > 50) newData.shift();
        return newData;
      });
    }, 1000);

    return () => clearInterval(interval);
  }, []);

  const option: echarts.EChartsOption = {
    tooltip: { trigger: 'axis' },
    xAxis: {
      type: 'category',
      data: data.map((d) => d.time),
      boundaryGap: false,
    },
    yAxis: { type: 'value', min: 0, max: 100 },
    series: [
      {
        type: 'line',
        data: data.map((d) => d.value),
        smooth: true,
        areaStyle: { opacity: 0.3 },
        animation: false, // 禁用动画提高性能
      },
    ],
  };

  return <EChartsComponent option={option} />;
}
```

---

## 六、决策检查清单

- [ ] 选择了合适的图表库
- [ ] 实现了按需加载
- [ ] 图表响应式适配
- [ ] 大数据量已优化
- [ ] 图表可访问性良好
- [ ] 交互事件处理完善
- [ ] 图表销毁处理正确
- [ ] 加载状态显示

---

## 七、相关技能

- [性能优化实战](./performance-optimization/SKILL.md) —— 图表性能优化
- [懒加载与代码分割](./lazy-loading-and-code-splitting/SKILL.md) —— 图表库懒加载
- [API数据获取与缓存](./api-data-fetching-and-caching/SKILL.md) —— 图表数据获取

---

## 八、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含ECharts集成、大数据优化、实时图表 |
