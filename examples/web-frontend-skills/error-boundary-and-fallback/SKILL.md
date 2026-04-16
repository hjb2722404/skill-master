---
name: error-boundary-and-fallback
description: >
  本技能用于实现错误边界与降级处理，包括错误捕获、错误上报、降级UI、重试机制等。
  当需要处理组件错误、设计错误降级UI、实现错误监控时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 错误边界与降级

## 一、概述

### 1.1 这是什么

错误边界与降级技能提供完整的前端错误处理方案，包括错误捕获、错误上报、降级UI展示、错误恢复机制等。

### 1.2 适用场景

- ✅ 捕获组件渲染错误
- ✅ 设计错误降级UI
- ✅ 实现错误监控和上报
- ✅ 提供错误重试机制
- ✅ 处理异步错误边界
- ❌ 事件处理错误（使用try-catch）
- ❌ 异步代码错误（使用Promise.catch）

### 1.3 核心原则

1. **优雅降级** —— 局部错误不影响整体应用
2. **用户友好** —— 提供清晰的错误提示和恢复指引
3. **可监控** —— 错误信息及时上报
4. **可恢复** —— 提供重试或刷新机制

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要防止组件错误导致整个应用崩溃
- 需要设计错误状态的UI展示
- 需要实现错误日志收集
- 需要提供用户错误恢复机制
- 需要处理第三方组件的错误

### 2.2 输入

- 错误类型分类
- 降级UI设计稿
- 错误上报接口
- 重试策略需求

### 2.3 输出

- 错误边界实现
- 降级UI组件
- 错误上报服务
- 重试机制实现

---

## 三、核心实现

### 3.1 基础错误边界

```typescript
// 错误边界配置
interface ErrorBoundaryConfig {
  fallback?: HTMLElement | ((error: Error) => HTMLElement);
  onError?: (error: Error, errorInfo: ErrorInfo) => void;
  resetKeys?: unknown[];
}

interface ErrorInfo {
  componentStack?: string;
  timestamp: number;
  url: string;
}

// 错误边界管理器
class ErrorBoundaryManager {
  private container: HTMLElement;
  private config: ErrorBoundaryConfig;
  private hasError = false;
  private error: Error | null = null;
  private childContainer: HTMLElement;

  constructor(container: HTMLElement, config: ErrorBoundaryConfig = {}) {
    this.container = container;
    this.config = config;
    
    // 创建子容器
    this.childContainer = document.createElement('div');
    this.childContainer.style.width = '100%';
    this.childContainer.style.height = '100%';
    this.container.appendChild(this.childContainer);

    // 监听错误
    this.setupErrorHandling();
  }

  private setupErrorHandling(): void {
    // 捕获子元素中的错误
    this.childContainer.addEventListener('error', (e) => {
      if (e.target !== this.childContainer) {
        this.handleError(e.error);
      }
    }, true);
  }

  handleError(error: Error, errorInfo?: Partial<ErrorInfo>): void {
    if (this.hasError) return;

    this.hasError = true;
    this.error = error;

    const fullErrorInfo: ErrorInfo = {
      componentStack: errorInfo?.componentStack,
      timestamp: Date.now(),
      url: window.location.href,
    };

    // 上报错误
    this.config.onError?.(error, fullErrorInfo);

    // 显示降级UI
    this.showFallback();
  }

  private showFallback(): void {
    if (!this.error) return;

    // 清空子容器
    this.childContainer.innerHTML = '';
    this.childContainer.style.display = 'none';

    // 创建降级UI
    const fallback = this.createFallback(this.error);
    this.container.appendChild(fallback);
  }

  private createFallback(error: Error): HTMLElement {
    if (typeof this.config.fallback === 'function') {
      return this.config.fallback(error);
    }

    // 默认降级UI
    const element = document.createElement('div');
    element.className = 'error-fallback';
    element.innerHTML = `
      <div class="error-icon">⚠️</div>
      <h2>出错了</h2>
      <p>页面加载时发生错误，请尝试刷新页面或返回首页。</p>
      <div class="error-actions">
        <button class="btn-retry">重试</button>
        <button class="btn-refresh">刷新页面</button>
        <a href="/" class="btn-home">返回首页</a>
      </div>
    `;

    // 绑定事件
    element.querySelector('.btn-retry')?.addEventListener('click', () => this.reset());
    element.querySelector('.btn-refresh')?.addEventListener('click', () => window.location.reload());

    return element;
  }

  reset(): void {
    this.hasError = false;
    this.error = null;
    
    // 移除降级UI
    const fallback = this.container.querySelector('.error-fallback');
    fallback?.remove();
    
    // 显示子容器
    this.childContainer.style.display = '';
    this.childContainer.innerHTML = '';
  }

  getContainer(): HTMLElement {
    return this.childContainer;
  }

  destroy(): void {
    this.container.innerHTML = '';
  }
}
```

### 3.2 全局错误处理

```typescript
// 全局错误处理器
class GlobalErrorHandler {
  private errorReporter: ErrorReporter;
  private boundaries: Set<ErrorBoundaryManager> = new Set();

  constructor(errorReporter: ErrorReporter) {
    this.errorReporter = errorReporter;
    this.setupHandlers();
  }

  private setupHandlers(): void {
    // 捕获未处理的Promise错误
    window.addEventListener('unhandledrejection', (event) => {
      this.handleError(event.reason, { type: 'unhandledrejection' });
    });

    // 捕获全局错误
    window.addEventListener('error', (event) => {
      this.handleError(event.error, { 
        type: 'error',
        filename: event.filename,
        lineno: event.lineno,
        colno: event.colno,
      });
    });
  }

  private handleError(error: Error, context: Record<string, any>): void {
    console.error('Global error:', error, context);
    this.errorReporter.report(error, context);
  }

  registerBoundary(boundary: ErrorBoundaryManager): void {
    this.boundaries.add(boundary);
  }
}
```

### 3.3 错误上报服务

```typescript
interface ErrorReport {
  message: string;
  stack?: string;
  componentStack?: string;
  timestamp: number;
  url: string;
  userAgent: string;
  userId?: string;
  context?: Record<string, unknown>;
}

class ErrorReporter {
  private endpoint: string;
  private enabled: boolean;
  private queue: ErrorReport[] = [];
  private flushInterval: number;

  constructor(endpoint: string, options: { enabled?: boolean; flushInterval?: number } = {}) {
    this.endpoint = endpoint;
    this.enabled = options.enabled ?? true;
    this.flushInterval = options.flushInterval ?? 5000;

    // 页面卸载前批量上报
    window.addEventListener('beforeunload', () => {
      this.flush();
    });

    // 定时刷新
    setInterval(() => this.flush(), this.flushInterval);
  }

  report(error: Error, context?: Record<string, unknown>): void {
    if (!this.enabled) return;

    const report: ErrorReport = {
      message: error.message,
      stack: error.stack,
      timestamp: Date.now(),
      url: window.location.href,
      userAgent: navigator.userAgent,
      context,
    };

    this.queue.push(report);

    if (this.queue.length >= 5) {
      this.flush();
    }
  }

  private async flush(): Promise<void> {
    if (this.queue.length === 0) return;

    const reports = [...this.queue];
    this.queue = [];

    try {
      await fetch(this.endpoint, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ errors: reports }),
        keepalive: true,
      });
    } catch (e) {
      // 上报失败，重新加入队列
      this.queue.unshift(...reports);
    }
  }
}
```

### 3.4 重试机制

```typescript
interface RetryableConfig {
  maxRetries?: number;
  retryDelay?: number;
  backoffMultiplier?: number;
  onRetry?: (error: Error, attempt: number) => void;
}

class RetryableOperation<T> {
  private operation: () => Promise<T>;
  private config: Required<RetryableConfig>;
  private attempt = 0;

  constructor(operation: () => Promise<T>, config: RetryableConfig = {}) {
    this.operation = operation;
    this.config = {
      maxRetries: config.maxRetries ?? 3,
      retryDelay: config.retryDelay ?? 1000,
      backoffMultiplier: config.backoffMultiplier ?? 2,
      onRetry: config.onRetry ?? (() => {}),
    };
  }

  async execute(): Promise<T> {
    while (this.attempt <= this.config.maxRetries) {
      try {
        return await this.operation();
      } catch (error) {
        if (this.attempt === this.config.maxRetries) {
          throw error;
        }

        this.config.onRetry(error as Error, this.attempt);
        
        const delay = this.config.retryDelay * Math.pow(
          this.config.backoffMultiplier,
          this.attempt
        );
        
        await this.sleep(delay);
        this.attempt++;
      }
    }

    throw new Error('Max retries exceeded');
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

// 使用
const retryable = new RetryableOperation(
  () => fetchData(),
  { maxRetries: 3, retryDelay: 1000 }
);

retryable.execute()
  .then(data => console.log(data))
  .catch(error => console.error('Failed after retries:', error));
```

---

## 四、决策检查清单

- [ ] 实现了多层级错误边界
- [ ] 设计了用户友好的降级UI
- [ ] 配置了错误上报服务
- [ ] 提供了错误重试机制
- [ ] 区分了不同类型的错误处理
- [ ] 开发环境显示详细错误信息
- [ ] 生产环境隐藏敏感信息

---

## 五、相关技能

- [API数据获取与缓存](./api-data-fetching-and-caching/SKILL.md) —— 请求错误处理
- [前端组件设计模式](./frontend-component-design-patterns/SKILL.md) —— 组件错误边界
- [性能优化实战](./performance-optimization/SKILL.md) —— 错误边界性能影响

---

## 六、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，框架无关实现 |
