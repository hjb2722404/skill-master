---
name: routing-and-navigation
description: >
  本技能用于实现前端路由与导航管理，包括路由守卫、动态路由、面包屑导航、权限路由等。
  当需要设计路由结构、实现导航守卫、处理动态权限路由时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 路由与导航管理

## 一、概述

### 1.1 这是什么

路由与导航管理技能涵盖单页应用(SPA)中的路由配置、导航守卫、动态路由加载、面包屑导航等完整解决方案。

### 1.2 适用场景

- ✅ 多页面SPA路由配置
- ✅ 路由权限控制（路由守卫）
- ✅ 动态路由加载（基于权限）
- ✅ 面包屑导航实现
- ✅ 路由过渡动画
- ✅ 路由元信息管理
- ❌ 多页应用(MPA)导航
- ❌ 纯静态页面网站

### 1.3 核心原则

1. **声明式路由** —— 使用配置而非硬编码
2. **权限前置** —— 路由层面拦截无权限访问
3. **懒加载优化** —— 路由级别代码分割
4. **状态同步** —— URL与应用状态保持一致

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要设计应用的路由结构
- 需要实现登录/权限路由守卫
- 需要根据用户角色动态生成路由
- 需要实现面包屑或导航菜单
- 需要处理路由参数和查询字符串

### 2.2 输入

- 应用页面结构
- 权限系统规则
- 导航菜单需求
- 路由嵌套层级

### 2.3 输出

- 路由配置文件
- 路由守卫实现
- 动态路由加载逻辑
- 面包屑/导航组件

---

## 三、核心实现

### 3.1 路由配置设计

```typescript
// 路由元信息接口
interface RouteMeta {
  title: string;
  icon?: string;
  requiresAuth?: boolean;
  permissions?: string[];
  hidden?: boolean;
  keepAlive?: boolean;
  breadcrumb?: boolean;
}

// 路由配置接口
interface RouteConfig {
  path: string;
  component?: () => Promise<any>;
  redirect?: string;
  meta?: RouteMeta;
  children?: RouteConfig[];
  beforeEnter?: (to: Route, from: Route | null) => boolean | Promise<boolean>;
}

// 路由实例接口
interface Route {
  path: string;
  fullPath: string;
  params: Record<string, string>;
  query: Record<string, string>;
  meta?: RouteMeta;
  matched: RouteConfig[];
}

// 路由配置示例
const routes: RouteConfig[] = [
  {
    path: '/login',
    component: () => import('./pages/Login'),
    meta: { title: '登录', requiresAuth: false },
  },
  {
    path: '/',
    component: () => import('./pages/Layout'),
    meta: { title: '首页', requiresAuth: true },
    children: [
      {
        path: '',
        component: () => import('./pages/Dashboard'),
        meta: { title: '仪表盘', icon: 'Dashboard' },
      },
      {
        path: 'users',
        meta: { title: '用户管理', icon: 'Users', permissions: ['user:view'] },
        children: [
          {
            path: '',
            component: () => import('./pages/Users/List'),
            meta: { title: '用户列表' },
          },
          {
            path: ':id',
            component: () => import('./pages/Users/Detail'),
            meta: { title: '用户详情', hidden: true },
          },
        ],
      },
    ],
  },
  {
    path: '*',
    component: () => import('./pages/NotFound'),
    meta: { title: '404', requiresAuth: false },
  },
];
```

### 3.2 路由管理器实现

```typescript
class Router {
  private routes: RouteConfig[];
  private currentRoute: Route | null = null;
  private beforeGuards: Array<(to: Route, from: Route | null) => boolean | Promise<boolean>> = [];
  private afterHooks: Array<(to: Route, from: Route | null) => void> = [];
  private listeners: Set<(route: Route) => void> = new Set();

  constructor(routes: RouteConfig[]) {
    this.routes = this.flattenRoutes(routes);
    this.init();
  }

  private init(): void {
    window.addEventListener('popstate', this.handlePopState.bind(this));
    this.navigate(window.location.pathname + window.location.search);
  }

  private flattenRoutes(routes: RouteConfig[], parentPath = ''): RouteConfig[] {
    const flattened: RouteConfig[] = [];

    for (const route of routes) {
      const fullPath = parentPath + route.path;
      flattened.push({ ...route, path: fullPath });

      if (route.children) {
        flattened.push(...this.flattenRoutes(route.children, fullPath));
      }
    }

    return flattened;
  }

  private handlePopState(): void {
    this.navigate(window.location.pathname + window.location.search, false);
  }

  async navigate(path: string, pushState = true): Promise<boolean> {
    const route = this.resolveRoute(path);
    if (!route) return false;

    // 执行前置守卫
    for (const guard of this.beforeGuards) {
      const result = await guard(route, this.currentRoute);
      if (!result) return false;
    }

    // 执行路由独享守卫
    if (route.matched[route.matched.length - 1]?.beforeEnter) {
      const result = await route.matched[route.matched.length - 1].beforeEnter!(route, this.currentRoute);
      if (!result) return false;
    }

    this.currentRoute = route;

    if (pushState) {
      window.history.pushState({}, '', path);
    }

    // 通知监听器
    this.listeners.forEach(listener => listener(route));

    // 执行后置钩子
    this.afterHooks.forEach(hook => hook(route, this.currentRoute));

    return true;
  }

  private resolveRoute(path: string): Route | null {
    const [pathname, search = ''] = path.split('?');
    const query = this.parseQuery(search);

    for (const route of this.routes) {
      const match = this.matchPath(pathname, route.path);
      if (match) {
        return {
          path: route.path,
          fullPath: path,
          params: match.params,
          query,
          meta: route.meta,
          matched: [route],
        };
      }
    }

    return null;
  }

  private matchPath(path: string, routePath: string): { params: Record<string, string> } | null {
    const pathParts = path.split('/').filter(Boolean);
    const routeParts = routePath.split('/').filter(Boolean);

    if (pathParts.length !== routeParts.length && !routePath.includes('*')) {
      return null;
    }

    const params: Record<string, string> = {};

    for (let i = 0; i < routeParts.length; i++) {
      const routePart = routeParts[i];
      const pathPart = pathParts[i];

      if (routePart.startsWith(':')) {
        params[routePart.slice(1)] = pathPart;
      } else if (routePart === '*') {
        break;
      } else if (routePart !== pathPart) {
        return null;
      }
    }

    return { params };
  }

  private parseQuery(search: string): Record<string, string> {
    const query: Record<string, string> = {};
    if (!search) return query;

    const params = new URLSearchParams(search);
    params.forEach((value, key) => {
      query[key] = value;
    });

    return query;
  }

  // 全局前置守卫
  beforeEach(guard: (to: Route, from: Route | null) => boolean | Promise<boolean>): void {
    this.beforeGuards.push(guard);
  }

  // 全局后置钩子
  afterEach(hook: (to: Route, from: Route | null) => void): void {
    this.afterHooks.push(hook);
  }

  // 订阅路由变化
  subscribe(listener: (route: Route) => void): () => void {
    this.listeners.add(listener);
    if (this.currentRoute) listener(this.currentRoute);
    return () => this.listeners.delete(listener);
  }

  getCurrentRoute(): Route | null {
    return this.currentRoute;
  }
}
```

### 3.3 路由守卫实现

```typescript
interface AuthState {
  isAuthenticated: boolean;
  user: User | null;
  permissions: string[];
}

class AuthGuard {
  private authState: AuthState;
  private loginPath = '/login';
  private forbiddenPath = '/403';

  constructor(authState: AuthState) {
    this.authState = authState;
  }

  createGuard(): (to: Route, from: Route | null) => boolean {
    return (to, from) => {
      const meta = to.meta;

      // 1. 检查是否需要登录
      if (meta?.requiresAuth !== false && !this.authState.isAuthenticated) {
        this.redirect(this.loginPath, { redirect: to.fullPath });
        return false;
      }

      // 2. 检查权限
      if (meta?.permissions && this.authState.user) {
        const hasPermission = meta.permissions.some(perm =>
          this.authState.permissions.includes(perm)
        );

        if (!hasPermission) {
          this.redirect(this.forbiddenPath);
          return false;
        }
      }

      // 3. 已登录用户访问登录页，重定向到首页
      if (to.path === this.loginPath && this.authState.isAuthenticated) {
        this.redirect('/');
        return false;
      }

      return true;
    };
  }

  private redirect(path: string, query?: Record<string, string>): void {
    let url = path;
    if (query) {
      const params = new URLSearchParams(query);
      url += '?' + params.toString();
    }
    window.location.href = url;
  }
}

// 使用
const authState: AuthState = {
  isAuthenticated: false,
  user: null,
  permissions: [],
};

const router = new Router(routes);
const authGuard = new AuthGuard(authState);
router.beforeEach(authGuard.createGuard());
```

### 3.4 面包屑导航

```typescript
interface BreadcrumbItem {
  title: string;
  path?: string;
}

class BreadcrumbGenerator {
  private routes: RouteConfig[];

  constructor(routes: RouteConfig[]) {
    this.routes = routes;
  }

  generate(path: string): BreadcrumbItem[] {
    const breadcrumbs: BreadcrumbItem[] = [{ title: '首页', path: '/' }];

    const segments = path.split('/').filter(Boolean);
    let currentPath = '';

    for (const segment of segments) {
      currentPath += `/${segment}`;
      const route = this.findRoute(currentPath);

      if (route?.meta?.title && route.meta.breadcrumb !== false) {
        breadcrumbs.push({
          title: route.meta.title,
          path: currentPath,
        });
      }
    }

    return breadcrumbs;
  }

  private findRoute(path: string): RouteConfig | null {
    for (const route of this.routes) {
      if (this.matchPath(path, route.path)) {
        return route;
      }
    }
    return null;
  }

  private matchPath(path: string, routePath: string): boolean {
    const pathParts = path.split('/').filter(Boolean);
    const routeParts = routePath.split('/').filter(Boolean);

    if (pathParts.length !== routeParts.length) return false;

    for (let i = 0; i < routeParts.length; i++) {
      if (routeParts[i].startsWith(':')) continue;
      if (routeParts[i] !== pathParts[i]) return false;
    }

    return true;
  }
}
```

### 3.5 导航菜单

```typescript
interface MenuItem {
  key: string;
  label: string;
  icon?: string;
  path?: string;
  children?: MenuItem[];
}

class MenuGenerator {
  private routes: RouteConfig[];

  constructor(routes: RouteConfig[]) {
    this.routes = routes;
  }

  generate(userPermissions: string[]): MenuItem[] {
    return this.filterAndTransform(this.routes, userPermissions);
  }

  private filterAndTransform(routes: RouteConfig[], permissions: string[]): MenuItem[] {
    const items: MenuItem[] = [];

    for (const route of routes) {
      if (route.meta?.hidden) continue;

      // 检查权限
      if (route.meta?.permissions) {
        const hasPermission = route.meta.permissions.some(p => permissions.includes(p));
        if (!hasPermission) continue;
      }

      const item: MenuItem = {
        key: route.path,
        label: route.meta?.title || '',
        icon: route.meta?.icon,
        path: route.path,
      };

      if (route.children) {
        item.children = this.filterAndTransform(route.children, permissions);
      }

      items.push(item);
    }

    return items;
  }
}
```

### 3.6 URL状态同步

```typescript
class UrlStateManager<T extends Record<string, string>> {
  private defaultState: T;

  constructor(defaultState: T) {
    this.defaultState = defaultState;
  }

  getState(): T {
    const params = new URLSearchParams(window.location.search);
    const state = { ...this.defaultState };

    for (const key of Object.keys(this.defaultState)) {
      const value = params.get(key);
      if (value !== null) {
        (state as any)[key] = value;
      }
    }

    return state;
  }

  setState(newState: Partial<T>): void {
    const params = new URLSearchParams(window.location.search);

    for (const [key, value] of Object.entries(newState)) {
      if (value === undefined || value === null || value === '') {
        params.delete(key);
      } else {
        params.set(key, String(value));
      }
    }

    const newUrl = `${window.location.pathname}?${params.toString()}`;
    window.history.pushState({}, '', newUrl);
  }
}

// 使用
const urlState = new UrlStateManager({
  keyword: '',
  status: '',
  page: '1',
});

// 读取状态
const filters = urlState.getState();

// 更新状态
urlState.setState({ keyword: 'search term' });
```

---

## 四、路由配置检查清单

- [ ] 路由路径设计清晰、语义化
- [ ] 元信息完整（title、icon、permissions等）
- [ ] 实现了路由守卫（登录、权限检查）
- [ ] 动态路由按需加载
- [ ] 面包屑导航正确显示
- [ ] 菜单根据权限动态过滤
- [ ] 404页面配置完善

---

## 五、相关技能

- [权限控制实现](./access-control-implementation/SKILL.md) —— 权限系统与路由结合
- [懒加载与代码分割](./lazy-loading-and-code-splitting/SKILL.md) —— 路由级别懒加载
- [前端组件设计模式](./frontend-component-design-patterns/SKILL.md) —— 导航组件设计

---

## 六、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，框架无关实现 |
