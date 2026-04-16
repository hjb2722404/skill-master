---
name: performance-optimization
description: >
  本技能用于实现前端性能优化，包括渲染优化、内存泄漏排查、长任务优化、虚拟列表等。
  当需要分析性能瓶颈、优化渲染性能、解决内存泄漏时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 性能优化实战

## 一、概述

### 1.1 这是什么

性能优化实战技能提供系统性的前端性能优化方法论，涵盖渲染优化、内存管理、长任务优化、虚拟列表等关键技术。

### 1.2 适用场景

- ✅ 解决页面卡顿、掉帧问题
- ✅ 优化首屏加载时间
- ✅ 排查和修复内存泄漏
- ✅ 实现大数据列表渲染
- ✅ 优化复杂组件渲染性能
- ❌ 过早优化（未测量就优化）
- ❌ 简单页面性能优化（收益不明显）

### 1.3 核心原则

1. **测量先于优化** —— 使用DevTools建立性能基线
2. **关注用户感知** —— 优先优化影响用户体验的指标
3. **渐进式优化** —— 从最大瓶颈开始逐步优化
4. **权衡取舍** —— 代码复杂度 vs 性能收益

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 页面出现明显卡顿或掉帧
- 首屏加载时间过长
- 内存占用持续增长
- 需要渲染大量数据列表
- 复杂组件渲染性能差

### 2.2 输入

- 性能测量数据（Lighthouse、DevTools）
- 性能瓶颈分析
- 优化目标指标
- 浏览器兼容性要求

### 2.3 输出

- 性能优化方案
- 优化前后对比数据
- 代码实现
- 性能监控方案

---

## 三、性能测量

### 3.1 核心指标

| 指标 | 说明 | 目标值 |
|------|------|--------|
| FCP | 首次内容绘制 | < 1.8s |
| LCP | 最大内容绘制 | < 2.5s |
| TTI | 可交互时间 | < 3.8s |
| TBT | 总阻塞时间 | < 200ms |
| CLS | 累积布局偏移 | < 0.1 |
| FPS | 帧率 | 60fps |

### 3.2 测量工具

```typescript
// Web Vitals测量
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function reportWebVitals(metric: Metric) {
  analytics.track('WebVitals', {
    name: metric.name,
    value: metric.value,
    id: metric.id,
  });
}

getCLS(reportWebVitals);
getFID(reportWebVitals);
getFCP(reportWebVitals);
getLCP(reportWebVitals);
getTTFB(reportWebVitals);

// 自定义性能标记
class PerformanceMonitor {
  private marks: Map<string, number> = new Map();

  mark(name: string): void {
    this.marks.set(name, performance.now());
    performance.mark(name);
  }

  measure(name: string, startMark: string, endMark?: string): number {
    const start = this.marks.get(startMark);
    const end = endMark ? this.marks.get(endMark) : performance.now();
    
    if (start && end) {
      const duration = end - start;
      console.log(`${name}: ${duration}ms`);
      return duration;
    }
    return 0;
  }
}
```

---

## 四、渲染优化

### 4.1 避免不必要重渲染

```typescript
// 使用类管理组件状态
class OptimizedComponent {
  private state: Record<string, any> = {};
  private prevState: Record<string, any> = {};
  private element: HTMLElement;
  private renderScheduled = false;

  constructor(element: HTMLElement) {
    this.element = element;
  }

  setState(newState: Record<string, any>): void {
    this.prevState = { ...this.state };
    this.state = { ...this.state, ...newState };
    
    // 浅比较决定是否需要重渲染
    if (this.shouldUpdate(this.prevState, this.state)) {
      this.scheduleRender();
    }
  }

  private shouldUpdate(prev: Record<string, any>, next: Record<string, any>): boolean {
    for (const key in next) {
      if (prev[key] !== next[key]) return true;
    }
    return false;
  }

  private scheduleRender(): void {
    if (this.renderScheduled) return;
    this.renderScheduled = true;
    
    requestAnimationFrame(() => {
      this.render();
      this.renderScheduled = false;
    });
  }

  protected render(): void {
    // 子类实现
  }
}

// 缓存计算结果
class MemoizedCalculator<T, R> {
  private cache = new Map<string, R>();

  calculate(inputs: T[], calculator: (items: T[]) => R): R {
    const key = JSON.stringify(inputs);
    
    if (!this.cache.has(key)) {
      this.cache.set(key, calculator(inputs));
    }
    
    return this.cache.get(key)!;
  }

  clear(): void {
    this.cache.clear();
  }
}
```

### 4.2 状态拆分与粒度控制

```typescript
// 状态管理器
class StateManager<T extends Record<string, any>> {
  private states: Map<string, T> = new Map();
  private listeners: Map<string, Set<(state: T) => void>> = new Map();

  createSlice<K extends string>(key: K, initialState: T): void {
    this.states.set(key, initialState);
    this.listeners.set(key, new Set());
  }

  getState<K extends string>(key: K): T | undefined {
    return this.states.get(key);
  }

  setState<K extends string>(key: K, updater: Partial<T> | ((prev: T) => Partial<T>)): void {
    const prevState = this.states.get(key);
    if (!prevState) return;

    const updates = typeof updater === 'function' ? updater(prevState) : updater;
    const newState = { ...prevState, ...updates };
    
    this.states.set(key, newState);
    this.notify(key, newState);
  }

  subscribe<K extends string>(key: K, listener: (state: T) => void): () => void {
    const listeners = this.listeners.get(key);
    if (listeners) {
      listeners.add(listener);
      return () => listeners.delete(listener);
    }
    return () => {};
  }

  private notify(key: string, state: T): void {
    const listeners = this.listeners.get(key);
    listeners?.forEach(listener => listener(state));
  }
}
```

### 4.3 虚拟列表实现

```typescript
interface VirtualListConfig<T> {
  container: HTMLElement;
  items: T[];
  itemHeight: number;
  renderItem: (item: T, index: number) => HTMLElement;
  overscan?: number;
}

class VirtualList<T> {
  private container: HTMLElement;
  private content: HTMLElement;
  private config: VirtualListConfig<T>;
  private visibleItems: Map<number, HTMLElement> = new Map();
  private scrollTop = 0;

  constructor(config: VirtualListConfig<T>) {
    this.config = { overscan: 3, ...config };
    this.container = config.container;
    this.container.style.overflow = 'auto';
    
    this.content = document.createElement('div');
    this.content.style.position = 'relative';
    this.container.appendChild(this.content);

    this.container.addEventListener('scroll', this.handleScroll.bind(this));
    this.update();
  }

  private handleScroll(): void {
    this.scrollTop = this.container.scrollTop;
    this.update();
  }

  private update(): void {
    const { items, itemHeight, overscan } = this.config;
    const containerHeight = this.container.clientHeight;
    
    const startIndex = Math.max(0, Math.floor(this.scrollTop / itemHeight) - overscan!);
    const visibleCount = Math.ceil(containerHeight / itemHeight);
    const endIndex = Math.min(items.length, startIndex + visibleCount + overscan! * 2);

    // 设置内容高度
    this.content.style.height = `${items.length * itemHeight}px`;

    // 移除不可见项
    this.visibleItems.forEach((el, index) => {
      if (index < startIndex || index >= endIndex) {
        el.remove();
        this.visibleItems.delete(index);
      }
    });

    // 添加/更新可见项
    for (let i = startIndex; i < endIndex; i++) {
      if (!this.visibleItems.has(i)) {
        const el = this.config.renderItem(items[i], i);
        el.style.position = 'absolute';
        el.style.top = `${i * itemHeight}px`;
        el.style.left = '0';
        el.style.right = '0';
        el.style.height = `${itemHeight}px`;
        
        this.content.appendChild(el);
        this.visibleItems.set(i, el);
      }
    }
  }

  destroy(): void {
    this.container.removeEventListener('scroll', this.handleScroll.bind(this));
    this.content.remove();
  }
}
```

---

## 五、长任务优化

### 5.1 任务分割

```typescript
// 将长任务分割为小块
async function processInChunks<T, R>(
  items: T[],
  processor: (item: T) => R,
  chunkSize = 100,
  onProgress?: (progress: number) => void
): Promise<R[]> {
  const results: R[] = [];

  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    
    // 处理当前块
    for (const item of chunk) {
      results.push(processor(item));
    }

    // 报告进度
    onProgress?.(Math.min(100, ((i + chunkSize) / items.length) * 100));

    // 让出主线程
    await new Promise(resolve => setTimeout(resolve, 0));
  }

  return results;
}

// 使用 requestIdleCallback
function scheduleIdleTask(task: () => void, timeout?: number): void {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(task, { timeout });
  } else {
    setTimeout(task, 1);
  }
}

// Web Worker 包装
class WorkerPool {
  private workers: Worker[] = [];
  private queue: Array<{ task: any; resolve: (value: any) => void }> = [];
  private maxWorkers: number;

  constructor(workerScript: string, maxWorkers = 4) {
    this.maxWorkers = maxWorkers;
    
    for (let i = 0; i < maxWorkers; i++) {
      const worker = new Worker(workerScript);
      worker.onmessage = (e) => this.handleMessage(worker, e.data);
      this.workers.push(worker);
    }
  }

  execute(task: any): Promise<any> {
    return new Promise((resolve) => {
      this.queue.push({ task, resolve });
      this.processQueue();
    });
  }

  private processQueue(): void {
    const availableWorker = this.workers.find(w => !(w as any).busy);
    if (!availableWorker || this.queue.length === 0) return;

    const { task, resolve } = this.queue.shift()!;
    (availableWorker as any).busy = true;
    (availableWorker as any).currentResolve = resolve;
    availableWorker.postMessage(task);
  }

  private handleMessage(worker: Worker, result: any): void {
    const resolve = (worker as any).currentResolve;
    if (resolve) {
      resolve(result);
      (worker as any).busy = false;
      (worker as any).currentResolve = null;
      this.processQueue();
    }
  }
}
```

### 5.2 防抖与节流

```typescript
// 防抖
function debounce<T extends (...args: any[]) => void>(
  fn: T,
  delay: number
): (...args: Parameters<T>) => void {
  let timeoutId: ReturnType<typeof setTimeout>;

  return (...args: Parameters<T>) => {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => fn(...args), delay);
  };
}

// 节流
function throttle<T extends (...args: any[]) => void>(
  fn: T,
  limit: number
): (...args: Parameters<T>) => void {
  let inThrottle = false;

  return (...args: Parameters<T>) => {
    if (!inThrottle) {
      fn(...args);
      inThrottle = true;
      setTimeout(() => (inThrottle = false), limit);
    }
  };
}

// 使用示例
class SearchInput {
  private input: HTMLInputElement;
  private debouncedSearch: (value: string) => void;

  constructor(container: HTMLElement, onSearch: (value: string) => void) {
    this.input = document.createElement('input');
    this.input.type = 'text';
    this.input.placeholder = '搜索...';
    
    this.debouncedSearch = debounce(onSearch, 300);
    this.input.addEventListener('input', (e) => {
      this.debouncedSearch((e.target as HTMLInputElement).value);
    });
    
    container.appendChild(this.input);
  }
}
```

---

## 六、内存优化

### 6.1 内存泄漏排查

```typescript
// 常见内存泄漏场景及解决方案

// 1. 事件监听未清理
class EventManager {
  private listeners: Array<{ target: EventTarget; type: string; handler: EventListener }> = [];

  addEventListener(target: EventTarget, type: string, handler: EventListener): void {
    target.addEventListener(type, handler);
    this.listeners.push({ target, type, handler });
  }

  cleanup(): void {
    this.listeners.forEach(({ target, type, handler }) => {
      target.removeEventListener(type, handler);
    });
    this.listeners = [];
  }
}

// 2. 定时器管理
class TimerManager {
  private timers: Set<number | NodeJS.Timeout> = new Set();

  setTimeout(callback: () => void, delay: number): number {
    const id = window.setTimeout(() => {
      this.timers.delete(id);
      callback();
    }, delay);
    this.timers.add(id);
    return id;
  }

  setInterval(callback: () => void, delay: number): number {
    const id = window.setInterval(callback, delay);
    this.timers.add(id);
    return id;
  }

  clearAll(): void {
    this.timers.forEach(id => {
      clearTimeout(id as number);
      clearInterval(id as number);
    });
    this.timers.clear();
  }
}

// 3. 图片懒加载
class LazyImageLoader {
  private observer: IntersectionObserver;
  private images: Map<HTMLImageElement, string> = new Map();

  constructor(options: IntersectionObserverInit = {}) {
    this.observer = new IntersectionObserver((entries) => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          const img = entry.target as HTMLImageElement;
          this.loadImage(img);
          this.observer.unobserve(img);
        }
      });
    }, { rootMargin: '50px', ...options });
  }

  observe(img: HTMLImageElement, src: string): void {
    img.dataset.src = src;
    this.images.set(img, src);
    this.observer.observe(img);
  }

  private loadImage(img: HTMLImageElement): void {
    const src = this.images.get(img);
    if (src) {
      img.src = src;
      this.images.delete(img);
    }
  }

  disconnect(): void {
    this.observer.disconnect();
    this.images.clear();
  }
}
```

---

## 七、首屏优化

### 7.1 代码分割

```typescript
// 动态导入
class ModuleLoader {
  private cache: Map<string, any> = new Map();

  async load<T>(path: string): Promise<T> {
    if (this.cache.has(path)) {
      return this.cache.get(path);
    }

    const module = await import(/* @vite-ignore */ path);
    this.cache.set(path, module);
    return module;
  }

  preload(path: string): void {
    if (!this.cache.has(path)) {
      this.load(path).catch(() => {});
    }
  }
}

// 预加载
function prefetchComponent(path: string): void {
  const link = document.createElement('link');
  link.rel = 'prefetch';
  link.href = path;
  document.head.appendChild(link);
}

// 智能预加载
class SmartPrefetcher {
  private loader: ModuleLoader;
  private prefetchMap: Map<string, string[]>;

  constructor(loader: ModuleLoader, prefetchMap: Map<string, string[]>) {
    this.loader = loader;
    this.prefetchMap = prefetchMap;
  }

  onRouteChange(currentRoute: string): void {
    const routesToPrefetch = this.prefetchMap.get(currentRoute) || [];
    
    // 延迟预加载
    setTimeout(() => {
      routesToPrefetch.forEach(route => {
        this.loader.preload(route);
      });
    }, 2000);
  }
}
```

---

## 八、决策检查清单

- [ ] 建立了性能测量基线
- [ ] 识别了主要性能瓶颈
- [ ] 实现了组件更新优化
- [ ] 合理使用缓存
- [ ] 实现了虚拟列表（大数据场景）
- [ ] 长任务已分割或移至Worker
- [ ] 清理了所有副作用
- [ ] 图片已优化和懒加载
- [ ] 代码已按需分割

---

## 九、相关技能

- [列表/表格渲染优化](./list-table-rendering-optimization/SKILL.md) —— 大数据列表优化
- [懒加载与代码分割](./lazy-loading-and-code-splitting/SKILL.md) —— 加载性能优化
- [前端组件设计模式](./frontend-component-design-patterns/SKILL.md) —— 组件性能优化

---

## 十、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，框架无关实现 |
