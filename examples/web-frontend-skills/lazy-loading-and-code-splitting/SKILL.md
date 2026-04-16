---
name: lazy-loading-and-code-splitting
description: >
  本技能用于实现懒加载与代码分割，包括路由懒加载、组件懒加载、预加载策略等。
  当需要优化首屏加载、实现按需加载、设计预加载策略时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 懒加载与代码分割

## 一、概述

### 1.1 这是什么

懒加载与代码分割技能提供系统性的前端加载优化方案，涵盖路由懒加载、组件懒加载、资源预加载等技术。

### 1.2 适用场景

- ✅ 优化首屏加载时间
- ✅ 大型单页应用代码分割
- ✅ 按需加载非关键资源
- ✅ 实现渐进式加载
- ✅ 优化打包体积
- ❌ 小型应用（<100KB）
- ❌ 关键路径资源

### 1.3 核心原则

1. **首屏优先** —— 优先加载首屏必需资源
2. **按需加载** —— 非关键资源延迟加载
3. **预加载策略** —— 预测用户行为提前加载
4. **加载反馈** —— 提供加载状态提示

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 首屏加载时间超过3秒
- 打包体积超过200KB
- 需要实现路由级别代码分割
- 需要延迟加载大型组件
- 需要优化资源加载顺序

### 2.2 输入

- 应用路由结构
- 打包分析报告
- 首屏必需资源清单
- 用户行为数据

### 2.3 输出

- 代码分割配置
- 懒加载实现
- 预加载策略
- 加载状态组件

---

## 三、路由懒加载

### 3.1 React Router懒加载

```typescript
import { lazy, Suspense } from 'react';
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

// 懒加载页面组件
const Home = lazy(() => import('./pages/Home'));
const Dashboard = lazy(() => import('./pages/Dashboard'));
const UserList = lazy(() => import('./pages/User/List'));
const UserDetail = lazy(() => import('./pages/User/Detail'));
const Settings = lazy(() => import('./pages/Settings'));

// 加载状态组件
function PageLoading() {
  return (
    <div className="page-loading">
      <div className="spinner" />
      <p>页面加载中...</p>
    </div>
  );
}

// 路由配置
const router = createBrowserRouter([
  {
    path: '/',
    element: (
      <Suspense fallback={<PageLoading />}>
        <Home />
      </Suspense>
    ),
  },
  {
    path: '/dashboard',
    element: (
      <Suspense fallback={<PageLoading />}>
        <Dashboard />
      </Suspense>
    ),
  },
  {
    path: '/users',
    children: [
      {
        index: true,
        element: (
          <Suspense fallback={<PageLoading />}>
            <UserList />
          </Suspense>
        ),
      },
      {
        path: ':id',
        element: (
          <Suspense fallback={<PageLoading />}>
            <UserDetail />
          </Suspense>
        ),
      },
    ],
  },
]);

function App() {
  return <RouterProvider router={router} />;
}
```

### 3.2 带错误处理的懒加载

```typescript
import { lazy, Suspense, Component, type ReactNode } from 'react';

// 错误边界
class LazyLoadErrorBoundary extends Component<
  { children: ReactNode; fallback?: ReactNode },
  { hasError: boolean }
> {
  state = { hasError: false };

  static getDerivedStateFromError() {
    return { hasError: true };
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || <div>加载失败，请刷新页面</div>;
    }
    return this.props.children;
  }
}

// 包装懒加载组件
function lazyWithRetry<T extends React.ComponentType<any>>(
  factory: () => Promise<{ default: T }>
) {
  const LazyComponent = lazy(factory);

  return function LazyWrapper(props: React.ComponentProps<T>) {
    return (
      <LazyLoadErrorBoundary>
        <Suspense fallback={<PageLoading />}>
          <LazyComponent {...props} />
        </Suspense>
      </LazyLoadErrorBoundary>
    );
  };
}

// 带重试的懒加载
function lazyWithRetryLogic<T extends React.ComponentType<any>>(
  factory: () => Promise<{ default: T }>,
  retries = 3
) {
  return lazy(() => retry(factory, retries));
}

async function retry<T>(fn: () => Promise<T>, retries: number): Promise<T> {
  try {
    return await fn();
  } catch (error) {
    if (retries > 0) {
      await new Promise(resolve => setTimeout(resolve, 1000));
      return retry(fn, retries - 1);
    }
    throw error;
  }
}
```

---

## 四、组件懒加载

### 4.1 条件懒加载

```typescript
import { lazy, Suspense, useState } from 'react';

// 大型组件懒加载
const HeavyChart = lazy(() => import('./components/HeavyChart'));
const RichEditor = lazy(() => import('./components/RichEditor'));
const DataGrid = lazy(() => import('./components/DataGrid'));

function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <div>
      <h1>仪表盘</h1>
      
      <button onClick={() => setShowChart(true)}>
        显示图表
      </button>
      
      {showChart && (
        <Suspense fallback={<div>图表加载中...</div>}>
          <HeavyChart data={chartData} />
        </Suspense>
      )}
    </div>
  );
}
```

### 4.2 模态框懒加载

```typescript
import { lazy, Suspense, useState } from 'react';

const UserFormModal = lazy(() => import('./components/UserFormModal'));
const ImportModal = lazy(() => import('./components/ImportModal'));

function UserList() {
  const [modalType, setModalType] = useState<string | null>(null);

  return (
    <div>
      <button onClick={() => setModalType('create')}>新增用户</button>
      <button onClick={() => setModalType('import')}>批量导入</button>

      <Suspense fallback={null}>
        {modalType === 'create' && (
          <UserFormModal onClose={() => setModalType(null)} />
        )}
        {modalType === 'import' && (
          <ImportModal onClose={() => setModalType(null)} />
        )}
      </Suspense>
    </div>
  );
}
```

---

## 五、预加载策略

### 5.1 路由预加载

```typescript
import { useEffect } from 'react';
import { useLocation } from 'react-router-dom';

// 组件预加载映射
const componentPrefetchMap: Record<string, () => Promise<any>> = {
  '/dashboard': () => import('./pages/Dashboard'),
  '/users': () => import('./pages/User/List'),
  '/settings': () => import('./pages/Settings'),
};

// 预加载Hook
export function usePrefetch() {
  const location = useLocation();

  const prefetch = (path: string) => {
    const loader = componentPrefetchMap[path];
    if (loader) {
      loader().catch(() => {});
    }
  };

  return { prefetch };
}

// 预加载链接组件
function PrefetchLink({ to, children }: { to: string; children: React.ReactNode }) {
  const { prefetch } = usePrefetch();

  return (
    <Link
      to={to}
      onMouseEnter={() => prefetch(to)}
      onFocus={() => prefetch(to)}
    >
      {children}
    </Link>
  );
}

// 智能预加载（基于当前路由）
function useSmartPrefetch() {
  const location = useLocation();

  useEffect(() => {
    // 根据当前路由预加载可能访问的页面
    const prefetchMap: Record<string, string[]> = {
      '/': ['/dashboard', '/about'],
      '/dashboard': ['/users', '/settings'],
      '/users': ['/users/:id'],
    };

    const routesToPrefetch = prefetchMap[location.pathname] || [];
    
    // 延迟预加载，避免影响当前页面加载
    const timer = setTimeout(() => {
      routesToPrefetch.forEach(route => {
        const loader = componentPrefetchMap[route];
        if (loader) {
          loader().catch(() => {});
        }
      });
    }, 2000);

    return () => clearTimeout(timer);
  }, [location.pathname]);
}
```

### 5.2 资源预加载

```typescript
// 图片预加载
function preloadImage(src: string): Promise<void> {
  return new Promise((resolve, reject) => {
    const img = new Image();
    img.onload = () => resolve();
    img.onerror = reject;
    img.src = src;
  });
}

// 组件预加载Hook
export function useComponentPrefetch() {
  const prefetchQueue = useRef<Set<() => Promise<any>>>(new Set());
  const isProcessing = useRef(false);

  const processQueue = useCallback(async () => {
    if (isProcessing.current) return;
    isProcessing.current = true;

    while (prefetchQueue.current.size > 0) {
      const loader = prefetchQueue.current.values().next().value;
      prefetchQueue.current.delete(loader);
      
      try {
        await loader();
      } catch {
        // 忽略预加载错误
      }
      
      // 让出主线程
      await new Promise(resolve => setTimeout(resolve, 50));
    }

    isProcessing.current = false;
  }, []);

  const prefetch = useCallback((loader: () => Promise<any>) => {
    if (!prefetchQueue.current.has(loader)) {
      prefetchQueue.current.add(loader);
      processQueue();
    }
  }, [processQueue]);

  return { prefetch };
}
```

---

## 六、构建配置

### 6.1 Vite配置

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    react(),
    visualizer({
      open: true,
      gzipSize: true,
      brotliSize: true,
    }),
  ],
  build: {
    rollupOptions: {
      output: {
        // 代码分割配置
        manualChunks: {
          // 第三方库分割
          vendor: ['react', 'react-dom', 'react-router-dom'],
          ui: ['@mui/material', '@emotion/react', '@emotion/styled'],
          utils: ['lodash', 'dayjs', 'axios'],
        },
        // 动态导入分割
        chunkFileNames: 'assets/js/[name]-[hash].js',
        entryFileNames: 'assets/js/[name]-[hash].js',
        assetFileNames: (assetInfo) => {
          const info = assetInfo.name.split('.');
          const ext = info[info.length - 1];
          if (/\.(png|jpe?g|gif|svg|webp|ico)$/i.test(assetInfo.name)) {
            return 'assets/images/[name]-[hash][extname]';
          }
          if (/\.(css)$/i.test(assetInfo.name)) {
            return 'assets/css/[name]-[hash][extname]';
          }
          return 'assets/[name]-[hash][extname]';
        },
      },
    },
    // 压缩配置
    minify: 'terser',
    terserOptions: {
      compress: {
        drop_console: true,
        drop_debugger: true,
      },
    },
    // 报告超大chunk
    chunkSizeWarningLimit: 500,
  },
});
```

### 6.2 Webpack配置

```javascript
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
        common: {
          minChunks: 2,
          chunks: 'all',
          enforce: true,
        },
      },
    },
    runtimeChunk: {
      name: 'runtime',
    },
  },
};
```

---

## 七、加载状态设计

```typescript
// 骨架屏组件
function Skeleton({ rows = 5 }: { rows?: number }) {
  return (
    <div className="skeleton">
      {Array.from({ length: rows }).map((_, i) => (
        <div key={i} className="skeleton-row">
          <div className="skeleton-cell" style={{ width: `${Math.random() * 40 + 30}%` }} />
        </div>
      ))}
    </div>
  );
}

// 渐进式加载
function ProgressiveImage({ src, alt, placeholder }: ImageProps) {
  const [loaded, setLoaded] = useState(false);

  return (
    <div className="progressive-image">
      <img
        src={placeholder}
        alt={alt}
        className={`placeholder ${loaded ? 'hidden' : ''}`}
      />
      <img
        src={src}
        alt={alt}
        className={`full ${loaded ? 'visible' : ''}`}
        onLoad={() => setLoaded(true)}
      />
    </div>
  );
}

// 加载错误重试
function LazyComponentWithRetry({ loader }: { loader: () => Promise<any> }) {
  const [retryCount, setRetryCount] = useState(0);
  const Component = useMemo(
    () => lazy(() => retry(loader, 3)),
    [retryCount]
  );

  return (
    <Suspense
      fallback={<div>加载中...</div>}
    >
      <Component />
    </Suspense>
  );
}
```

---

## 八、决策检查清单

- [ ] 路由已配置懒加载
- [ ] 大型组件已延迟加载
- [ ] 实现了预加载策略
- [ ] 加载状态设计完善
- [ ] 错误处理机制完善
- [ ] 打包分析已执行
- [ ] 代码分割策略合理
- [ ] 首屏加载时间达标

---

## 九、相关技能

- [路由与导航管理](./routing-and-navigation/SKILL.md) —— 路由懒加载
- [性能优化实战](./performance-optimization/SKILL.md) —— 加载性能优化
- [列表/表格渲染优化](./list-table-rendering-optimization/SKILL.md) —— 大数据加载

---

## 十、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含路由懒加载、组件懒加载、预加载策略 |
