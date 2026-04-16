---
name: api-data-fetching-and-caching
description: >
  本技能用于处理前端API数据获取与缓存场景，包括REST/GraphQL请求、错误重试、缓存策略。
  当需要实现数据获取逻辑、设计缓存策略、处理请求错误时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# API数据获取与缓存

## 一、概述

### 1.1 这是什么

API数据获取与缓存技能涵盖前端与后端通信的完整流程，包括请求发起、响应处理、错误重试、数据缓存等关键环节。

### 1.2 适用场景

- ✅ RESTful API 数据获取
- ✅ GraphQL 查询与变更
- ✅ 请求错误重试机制
- ✅ 客户端缓存策略（内存/本地存储）
- ✅ 请求去重与竞态处理
- ❌ WebSocket 实时通信（使用专门的WebSocket技能）
- ❌ 文件上传下载（使用专门的文件处理技能）

### 1.3 核心原则

1. **缓存优先** —— 合理缓存减少重复请求
2. **错误可恢复** —— 设计优雅降级和重试机制
3. **竞态防护** —— 处理异步请求的时序问题
4. **取消支持** —— 组件卸载时取消未完成的请求

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要从后端API获取数据
- 需要设计请求缓存策略
- 需要实现错误重试逻辑
- 需要处理请求竞态条件
- 需要优化数据获取性能

### 2.2 输入

- API端点信息（URL、方法、参数）
- 数据类型定义（TypeScript接口）
- 缓存需求（是否需要缓存、缓存时长）
- 错误处理要求（重试次数、降级方案）

### 2.3 输出

- 数据获取函数/类实现
- 缓存配置和策略
- 错误处理逻辑
- 类型定义文件

---

## 三、请求工具选择

| 场景 | 推荐工具 | 理由 |
|------|----------|------|
| 简单REST请求 | fetch | 原生支持，无额外依赖 |
| 复杂请求管理 | axios | 拦截器、取消请求、自动JSON转换 |
| 服务端状态管理 | TanStack Query/SWR | 内置缓存、去重、重试、背景更新 |
| GraphQL | Apollo Client / urql | 专用GraphQL缓存和状态管理 |

---

## 四、核心实现

### 4.1 基础请求封装

```typescript
// 核心原则：统一错误处理和请求配置
interface ApiResponse<T> {
  data: T;
  status: number;
  message?: string;
}

interface ApiError {
  code: string;
  message: string;
  status: number;
}

interface RequestConfig extends RequestInit {
  timeout?: number;
  retries?: number;
  retryDelay?: number;
}

class ApiClient {
  private baseURL: string;
  private defaultConfig: RequestConfig;

  constructor(baseURL: string, config: RequestConfig = {}) {
    this.baseURL = baseURL;
    this.defaultConfig = {
      headers: {
        'Content-Type': 'application/json',
      },
      timeout: 10000,
      retries: 3,
      retryDelay: 1000,
      ...config,
    };
  }

  async request<T>(
    endpoint: string,
    config: RequestConfig = {}
  ): Promise<ApiResponse<T>> {
    const url = `${this.baseURL}${endpoint}`;
    const mergedConfig = this.mergeConfig(config);

    return this.executeWithRetry(url, mergedConfig);
  }

  private async executeWithRetry<T>(
    url: string,
    config: RequestConfig,
    attempt: number = 0
  ): Promise<ApiResponse<T>> {
    try {
      const controller = new AbortController();
      const timeoutId = setTimeout(
        () => controller.abort(),
        config.timeout
      );

      const response = await fetch(url, {
        ...config,
        signal: controller.signal,
      });

      clearTimeout(timeoutId);

      if (!response.ok) {
        throw await this.parseError(response);
      }

      const data = await response.json();
      return {
        data,
        status: response.status,
      };
    } catch (error) {
      if (attempt < (config.retries || 0)) {
        await this.delay(config.retryDelay! * Math.pow(2, attempt));
        return this.executeWithRetry(url, config, attempt + 1);
      }
      throw this.normalizeError(error);
    }
  }

  private mergeConfig(config: RequestConfig): RequestConfig {
    return {
      ...this.defaultConfig,
      ...config,
      headers: {
        ...this.defaultConfig.headers,
        ...config.headers,
      },
    };
  }

  private async parseError(response: Response): Promise<ApiError> {
    try {
      const errorData = await response.json();
      return {
        code: errorData.code || 'UNKNOWN_ERROR',
        message: errorData.message || '请求失败',
        status: response.status,
      };
    } catch {
      return {
        code: 'PARSE_ERROR',
        message: '无法解析错误响应',
        status: response.status,
      };
    }
  }

  private normalizeError(error: unknown): ApiError {
    if (error instanceof Error) {
      return {
        code: 'NETWORK_ERROR',
        message: error.message,
        status: 0,
      };
    }
    return error as ApiError;
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  // HTTP方法快捷方式
  get<T>(endpoint: string, config?: RequestConfig) {
    return this.request<T>(endpoint, { ...config, method: 'GET' });
  }

  post<T>(endpoint: string, data: unknown, config?: RequestConfig) {
    return this.request<T>(endpoint, {
      ...config,
      method: 'POST',
      body: JSON.stringify(data),
    });
  }

  put<T>(endpoint: string, data: unknown, config?: RequestConfig) {
    return this.request<T>(endpoint, {
      ...config,
      method: 'PUT',
      body: JSON.stringify(data),
    });
  }

  delete<T>(endpoint: string, config?: RequestConfig) {
    return this.request<T>(endpoint, { ...config, method: 'DELETE' });
  }
}
```

### 4.2 内存缓存实现

```typescript
class MemoryCache<T> {
  private cache = new Map<string, { data: T; timestamp: number }>();
  private ttl: number;

  constructor(ttl = 5 * 60 * 1000) {
    this.ttl = ttl;
  }

  get(key: string): T | null {
    const item = this.cache.get(key);
    if (!item) return null;

    if (Date.now() - item.timestamp > this.ttl) {
      this.cache.delete(key);
      return null;
    }

    return item.data;
  }

  set(key: string, data: T): void {
    this.cache.set(key, { data, timestamp: Date.now() });
  }

  invalidate(key?: string): void {
    if (key) {
      this.cache.delete(key);
    } else {
      this.cache.clear();
    }
  }
}

// 带缓存的API客户端
class CachedApiClient extends ApiClient {
  private cache: MemoryCache<unknown>;

  constructor(baseURL: string, config?: RequestConfig) {
    super(baseURL, config);
    this.cache = new MemoryCache();
  }

  async cachedRequest<T>(
    endpoint: string,
    config: RequestConfig = {},
    cacheKey?: string
  ): Promise<ApiResponse<T>> {
    const key = cacheKey || endpoint;
    const cached = this.cache.get(key) as T | null;

    if (cached) {
      return { data: cached, status: 200 };
    }

    const response = await this.request<T>(endpoint, config);
    this.cache.set(key, response.data);
    return response;
  }

  invalidateCache(key?: string): void {
    this.cache.invalidate(key);
  }
}
```

### 4.3 请求去重

```typescript
class RequestDeduplicator<T> {
  private pendingRequests = new Map<string, Promise<T>>();

  async execute(key: string, requestFn: () => Promise<T>): Promise<T> {
    if (this.pendingRequests.has(key)) {
      return this.pendingRequests.get(key)!;
    }

    const requestPromise = requestFn().finally(() => {
      this.pendingRequests.delete(key);
    });

    this.pendingRequests.set(key, requestPromise);
    return requestPromise;
  }
}
```

---

## 五、框架集成示例

### 5.1 与框架无关的数据获取类

```typescript
interface DataFetcherOptions<T> {
  fetcher: () => Promise<T>;
  onSuccess?: (data: T) => void;
  onError?: (error: ApiError) => void;
  onLoadingChange?: (loading: boolean) => void;
}

class DataFetcher<T> {
  private data: T | null = null;
  private loading = false;
  private error: ApiError | null = null;
  private abortController: AbortController | null = null;
  private options: DataFetcherOptions<T>;

  constructor(options: DataFetcherOptions<T>) {
    this.options = options;
  }

  async execute(): Promise<void> {
    this.loading = true;
    this.error = null;
    this.options.onLoadingChange?.(true);

    this.abortController = new AbortController();

    try {
      this.data = await this.options.fetcher();
      this.options.onSuccess?.(this.data);
    } catch (err) {
      this.error = err as ApiError;
      this.options.onError?.(this.error);
    } finally {
      this.loading = false;
      this.options.onLoadingChange?.(false);
    }
  }

  cancel(): void {
    this.abortController?.abort();
  }

  getData(): T | null {
    return this.data;
  }

  getLoading(): boolean {
    return this.loading;
  }

  getError(): ApiError | null {
    return this.error;
  }
}
```

---

## 六、决策检查清单

- [ ] 选择了合适的请求工具
- [ ] 实现了统一的错误处理
- [ ] 添加了请求取消支持
- [ ] 设计了合理的缓存策略
- [ ] 处理了竞态条件
- [ ] 配置了错误重试机制
- [ ] 定义了清晰的类型接口

---

## 七、相关技能

- [表单处理](./react-form-handling/SKILL.md) —— 表单提交与数据获取结合
- [错误边界与降级](./error-boundary-and-fallback/SKILL.md) —— 请求错误UI降级
- [WebSocket实时通信](./websocket-realtime-communication/SKILL.md) —— 实时数据推送

---

## 八、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，框架无关实现 |
