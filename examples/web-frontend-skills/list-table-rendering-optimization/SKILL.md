---
name: list-table-rendering-optimization
description: >
  本技能用于优化列表和表格的渲染性能，包括虚拟滚动、分页、无限滚动、大数据渲染等。
  当需要渲染大量数据列表、优化表格性能、实现无限滚动时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 列表/表格渲染优化

## 一、概述

### 1.1 这是什么

列表/表格渲染优化技能提供系统性的数据列表性能优化方案，涵盖虚拟滚动、分页、无限滚动、大数据渲染等技术。

### 1.2 适用场景

- ✅ 渲染大量数据列表（>100条）
- ✅ 复杂表格性能优化
- ✅ 实现无限滚动列表
- ✅ 大数据表格渲染
- ✅ 树形数据展示
- ❌ 少量数据（<50条）
- ❌ 简单静态列表

### 1.3 核心原则

1. **只渲染可见项** —— 虚拟滚动减少DOM节点
2. **按需加载** —— 分页或无限滚动分批加载
3. **避免不必要重渲染** —— 合理使用key和memo
4. **优化滚动性能** —— 使用transform和will-change

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 列表数据量超过100条
- 表格出现滚动卡顿
- 需要实现无限滚动加载
- 需要展示树形大数据
- 列表项渲染复杂

### 2.2 输入

- 数据总量和单条大小
- 列表项复杂度
- 用户交互需求（滚动、筛选、排序）
- 性能目标指标

### 2.3 输出

- 列表渲染方案
- 虚拟滚动实现
- 分页/无限滚动逻辑
- 性能优化代码

---

## 三、方案选择

| 场景 | 推荐方案 | 数据量 |
|------|----------|--------|
| 固定高度列表 | react-window | 1k-100k |
| 动态高度列表 | react-virtualized | 1k-100k |
| 无限滚动 | react-window + 无限滚动 | 无上限 |
| 复杂表格 | react-table + 虚拟滚动 | 1k-50k |
| 树形数据 | rc-tree / 自定义虚拟树 | 1k-10k |
| 简单分页 | 传统分页 | <10k |

---

## 四、虚拟滚动实现

### 4.1 固定高度虚拟列表

```typescript
import { useRef, useState, useMemo, useCallback } from 'react';

interface VirtualListProps<T> {
  items: T[];
  itemHeight: number;
  height: number;
  renderItem: (item: T, index: number) => React.ReactNode;
  overscan?: number;
}

export function VirtualList<T>({
  items,
  itemHeight,
  height,
  renderItem,
  overscan = 3,
}: VirtualListProps<T>) {
  const containerRef = useRef<HTMLDivElement>(null);
  const [scrollTop, setScrollTop] = useState(0);

  // 计算可见范围
  const { virtualItems, totalHeight, startIndex } = useMemo(() => {
    const startIndex = Math.max(0, Math.floor(scrollTop / itemHeight) - overscan);
    const visibleCount = Math.ceil(height / itemHeight);
    const endIndex = Math.min(items.length, startIndex + visibleCount + overscan * 2);
    
    const virtualItems = items.slice(startIndex, endIndex).map((item, idx) => ({
      item,
      index: startIndex + idx,
      style: {
        position: 'absolute' as const,
        top: (startIndex + idx) * itemHeight,
        height: itemHeight,
        left: 0,
        right: 0,
      },
    }));

    return {
      virtualItems,
      totalHeight: items.length * itemHeight,
      startIndex,
    };
  }, [items, itemHeight, scrollTop, height, overscan]);

  const handleScroll = useCallback((e: React.UIEvent<HTMLDivElement>) => {
    setScrollTop(e.currentTarget.scrollTop);
  }, []);

  return (
    <div
      ref={containerRef}
      style={{ height, overflow: 'auto' }}
      onScroll={handleScroll}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        {virtualItems.map(({ item, index, style }) => (
          <div key={index} style={style}>
            {renderItem(item, index)}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### 4.2 动态高度虚拟列表

```typescript
import { useRef, useState, useEffect, useCallback } from 'react';

interface DynamicVirtualListProps<T> {
  items: T[];
  height: number;
  estimateHeight: number;
  renderItem: (item: T, index: number, ref: React.Ref<HTMLDivElement>) => React.ReactNode;
}

export function DynamicVirtualList<T>({
  items,
  height,
  estimateHeight,
  renderItem,
}: DynamicVirtualListProps<T>) {
  const containerRef = useRef<HTMLDivElement>(null);
  const itemRefs = useRef<Map<number, HTMLDivElement>>(new Map());
  const [scrollTop, setScrollTop] = useState(0);
  const [heights, setHeights] = useState<Map<number, number>>(new Map());

  // 计算累计高度
  const { offsets, totalHeight } = useMemo(() => {
    const offsets: number[] = [];
    let offset = 0;
    
    items.forEach((_, index) => {
      offsets[index] = offset;
      offset += heights.get(index) || estimateHeight;
    });

    return { offsets, totalHeight: offset };
  }, [items, heights, estimateHeight]);

  // 测量实际高度
  useEffect(() => {
    const newHeights = new Map(heights);
    let hasChange = false;

    itemRefs.current.forEach((el, index) => {
      const actualHeight = el.getBoundingClientRect().height;
      if (actualHeight !== heights.get(index)) {
        newHeights.set(index, actualHeight);
        hasChange = true;
      }
    });

    if (hasChange) {
      setHeights(newHeights);
    }
  });

  // 计算可见范围
  const visibleRange = useMemo(() => {
    let start = 0;
    let end = items.length;

    // 二分查找起始索引
    let left = 0, right = items.length - 1;
    while (left <= right) {
      const mid = Math.floor((left + right) / 2);
      if (offsets[mid] < scrollTop) {
        start = mid;
        left = mid + 1;
      } else {
        right = mid - 1;
      }
    }

    // 查找结束索引
    left = start;
    right = items.length - 1;
    const scrollBottom = scrollTop + height;
    while (left <= right) {
      const mid = Math.floor((left + right) / 2);
      if (offsets[mid] < scrollBottom) {
        end = mid;
        left = mid + 1;
      } else {
        right = mid - 1;
      }
    }

    return { start, end: Math.min(items.length, end + 3) };
  }, [offsets, scrollTop, height, items.length]);

  const handleScroll = useCallback((e: React.UIEvent<HTMLDivElement>) => {
    setScrollTop(e.currentTarget.scrollTop);
  }, []);

  return (
    <div
      ref={containerRef}
      style={{ height, overflow: 'auto' }}
      onScroll={handleScroll}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        {items.slice(visibleRange.start, visibleRange.end).map((item, idx) => {
          const index = visibleRange.start + idx;
          return (
            <div
              key={index}
              ref={el => {
                if (el) itemRefs.current.set(index, el);
              }}
              style={{
                position: 'absolute',
                top: offsets[index],
                left: 0,
                right: 0,
              }}
            >
              {renderItem(item, index)}
            </div>
          );
        })}
      </div>
    </div>
  );
}
```

### 4.3 使用react-window

```typescript
import { FixedSizeList as List } from 'react-window';
import AutoSizer from 'react-virtualized-auto-sizer';

function SimpleVirtualList({ items }: { items: Item[] }) {
  const Row = useCallback(({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style} className="list-item">
      <div className="item-content">
        <h4>{items[index].title}</h4>
        <p>{items[index].description}</p>
      </div>
    </div>
  ), [items]);

  return (
    <div style={{ height: '100%' }}>
      <AutoSizer>
        {({ height, width }) => (
          <List
            height={height}
            itemCount={items.length}
            itemSize={80}
            width={width}
          >
            {Row}
          </List>
        )}
      </AutoSizer>
    </div>
  );
}
```

---

## 五、无限滚动

```typescript
import { useRef, useCallback, useEffect, useState } from 'react';
import { useInfiniteQuery } from '@tanstack/react-query';

interface UseInfiniteScrollOptions<T> {
  fetchData: (page: number) => Promise<{ data: T[]; hasMore: boolean }>;
  threshold?: number;
}

export function useInfiniteScroll<T>({
  fetchData,
  threshold = 100,
}: UseInfiniteScrollOptions<T>) {
  const [items, setItems] = useState<T[]>([]);
  const [page, setPage] = useState(1);
  const [hasMore, setHasMore] = useState(true);
  const [loading, setLoading] = useState(false);
  const containerRef = useRef<HTMLDivElement>(null);

  const loadMore = useCallback(async () => {
    if (loading || !hasMore) return;

    setLoading(true);
    try {
      const result = await fetchData(page);
      setItems(prev => [...prev, ...result.data]);
      setHasMore(result.hasMore);
      setPage(p => p + 1);
    } finally {
      setLoading(false);
    }
  }, [fetchData, page, loading, hasMore]);

  useEffect(() => {
    const container = containerRef.current;
    if (!container) return;

    const handleScroll = () => {
      const { scrollTop, scrollHeight, clientHeight } = container;
      if (scrollHeight - scrollTop - clientHeight < threshold) {
        loadMore();
      }
    };

    container.addEventListener('scroll', handleScroll);
    return () => container.removeEventListener('scroll', handleScroll);
  }, [loadMore, threshold]);

  // 初始加载
  useEffect(() => {
    loadMore();
  }, []);

  return { items, loading, hasMore, containerRef, loadMore };
}

// 使用react-window的无限滚动
import { FixedSizeList, ListChildComponentProps } from 'react-window';
import InfiniteLoader from 'react-window-infinite-loader';

function InfiniteVirtualList({ loadMoreItems, hasNextPage, items }: Props) {
  const itemCount = hasNextPage ? items.length + 1 : items.length;
  const isItemLoaded = (index: number) => !hasNextPage || index < items.length;

  return (
    <InfiniteLoader
      isItemLoaded={isItemLoaded}
      itemCount={itemCount}
      loadMoreItems={loadMoreItems}
    >
      {({ onItemsRendered, ref }) => (
        <FixedSizeList
          height={500}
          itemCount={itemCount}
          itemSize={50}
          onItemsRendered={onItemsRendered}
          ref={ref}
          width="100%"
        >
          {({ index, style }: ListChildComponentProps) => (
            <div style={style}>
              {!isItemLoaded(index) ? (
                <div className="loading">加载中...</div>
              ) : (
                <div className="item">{items[index].name}</div>
              )}
            </div>
          )}
        </FixedSizeList>
      )}
    </InfiniteLoader>
  );
}
```

---

## 六、表格优化

```typescript
import { useMemo, useState, useCallback } from 'react';
import { useReactTable, getCoreRowModel, getSortedRowModel } from '@tanstack/react-table';

interface OptimizedTableProps<T> {
  data: T[];
  columns: ColumnDef<T>[];
  virtualScroll?: boolean;
  rowHeight?: number;
}

export function OptimizedTable<T>({
  data,
  columns,
  virtualScroll = false,
  rowHeight = 48,
}: OptimizedTableProps<T>) {
  const [sorting, setSorting] = useState<SortingState>([]);

  const table = useReactTable({
    data,
    columns,
    state: { sorting },
    onSortingChange: setSorting,
    getCoreRowModel: getCoreRowModel(),
    getSortedRowModel: getSortedRowModel(),
  });

  const rows = table.getRowModel().rows;

  if (virtualScroll && rows.length > 100) {
    return <VirtualTable rows={rows} rowHeight={rowHeight} />;
  }

  return (
    <table className="optimized-table">
      <thead>
        {table.getHeaderGroups().map(headerGroup => (
          <tr key={headerGroup.id}>
            {headerGroup.headers.map(header => (
              <th key={header.id} onClick={header.column.getToggleSortingHandler()}>
                {flexRender(header.column.columnDef.header, header.getContext())}
              </th>
            ))}
          </tr>
        ))}
      </thead>
      <tbody>
        {rows.map(row => (
          <tr key={row.id}>
            {row.getVisibleCells().map(cell => (
              <td key={cell.id}>
                {flexRender(cell.column.columnDef.cell, cell.getContext())}
              </td>
            ))}
          </tr>
        ))}
      </tbody>
    </table>
  );
}

// 虚拟表格
function VirtualTable<T>({ rows, rowHeight }: { rows: Row<T>[]; rowHeight: number }) {
  const [scrollTop, setScrollTop] = useState(0);
  const containerHeight = 500;

  const { virtualRows, totalHeight } = useMemo(() => {
    const startIndex = Math.floor(scrollTop / rowHeight);
    const visibleCount = Math.ceil(containerHeight / rowHeight);
    const endIndex = Math.min(rows.length, startIndex + visibleCount + 2);

    return {
      virtualRows: rows.slice(startIndex, endIndex),
      totalHeight: rows.length * rowHeight,
    };
  }, [rows, scrollTop, rowHeight, containerHeight]);

  return (
    <div
      style={{ height: containerHeight, overflow: 'auto' }}
      onScroll={e => setScrollTop(e.currentTarget.scrollTop)}
    >
      <div style={{ height: totalHeight, position: 'relative' }}>
        {virtualRows.map((row, idx) => (
          <div
            key={row.id}
            style={{
              position: 'absolute',
              top: (Math.floor(scrollTop / rowHeight) + idx) * rowHeight,
              height: rowHeight,
              left: 0,
              right: 0,
              display: 'flex',
            }}
          >
            {row.getVisibleCells().map(cell => (
              <div key={cell.id} style={{ flex: 1 }}>
                {flexRender(cell.column.columnDef.cell, cell.getContext())}
              </div>
            ))}
          </div>
        ))}
      </div>
    </div>
  );
}
```

---

## 七、决策检查清单

- [ ] 选择了合适的列表渲染方案
- [ ] 实现了虚拟滚动（数据量大时）
- [ ] 配置了合理的overscan值
- [ ] 列表项使用了稳定的key
- [ ] 滚动事件已节流
- [ ] 实现了无限滚动加载
- [ ] 加载状态处理完善
- [ ] 空状态处理完善

---

## 八、相关技能

- [性能优化实战](./performance-optimization/SKILL.md) —— 通用性能优化
- [懒加载与代码分割](./lazy-loading-and-code-splitting/SKILL.md) —— 按需加载
- [API数据获取与缓存](./api-data-fetching-and-caching/SKILL.md) —— 数据获取

---

## 九、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含虚拟滚动、无限滚动、表格优化 |
