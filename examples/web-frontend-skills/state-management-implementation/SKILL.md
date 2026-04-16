---
name: state-management-implementation
description: >
  本技能用于实现前端状态管理方案，包括状态管理选型、状态切片设计、持久化等。
  当需要选择状态管理方案、设计状态结构、实现状态持久化时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 状态管理实现

## 一、概述

### 1.1 这是什么

状态管理实现技能提供系统性的前端状态管理方案，涵盖技术选型、状态结构设计、状态持久化等关键环节。

### 1.2 适用场景

- ✅ 全局状态管理方案选型
- ✅ 复杂应用状态结构设计
- ✅ 跨组件状态共享
- ✅ 状态持久化（localStorage/sessionStorage）
- ✅ 服务端状态与客户端状态分离
- ❌ 简单组件内部状态
- ❌ 仅父子组件通信

### 1.3 核心原则

1. **最小化全局状态** —— 只有真正需要全局共享的状态才放入全局store
2. **服务端状态分离** —— 使用专用库管理服务端状态
3. **状态扁平化** —— 避免深层嵌套，便于更新和查询
4. **不可变更新** —— 始终创建新对象而非修改原对象

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要选择合适的状态管理方案
- 需要设计全局状态结构
- 需要实现状态持久化
- 需要跨多层组件共享状态
- 需要管理复杂的异步状态流

### 2.2 输入

- 应用规模和复杂度
- 状态共享范围
- 状态持久化需求
- 团队技术栈偏好

### 2.3 输出

- 状态管理方案选型建议
- 状态结构设计文档
- Store配置和实现代码
- 状态操作封装

---

## 三、技术选型决策

### 3.1 选型决策树

```
是否需要跨组件共享状态？
├─ 否 → 组件内部状态管理
└─ 是 → 状态变化频率？
    ├─ 低频 → 简单事件总线
    └─ 高频 → 是否需要中间件？
        ├─ 是 → Redux
        └─ 否 → 轻量状态管理
```

### 3.2 方案对比

| 方案 | 适用场景 | 学习成本 | 性能 | 生态 |
|------|----------|----------|------|------|
| 组件内部状态 | 局部状态 | 低 | 高 | 原生 |
| 事件总线 | 简单跨组件通信 | 低 | 中 | 轻量 |
| Zustand/Valtio | 中小型应用 | 低 | 高 | 轻量 |
| Redux | 大型复杂应用 | 中 | 中 | 丰富 |
| Pinia/MobX | 响应式状态管理 | 中 | 高 | 中等 |

---

## 四、轻量状态管理实现

### 4.1 基础Store创建

```typescript
// 订阅者类型
type Subscriber<T> = (state: T, prevState: T) => void;
type Unsubscribe = () => void;

// 基础Store类
class Store<T extends Record<string, any>> {
  private state: T;
  private subscribers: Set<Subscriber<T>> = new Set();

  constructor(initialState: T) {
    this.state = { ...initialState };
  }

  // 获取当前状态
  getState(): T {
    return { ...this.state };
  }

  // 更新状态（支持部分更新）
  setState(updater: Partial<T> | ((prev: T) => Partial<T>)): void {
    const prevState = this.state;
    const updates = typeof updater === 'function' ? updater(prevState) : updater;
    
    this.state = { ...prevState, ...updates };
    this.notify(prevState);
  }

  // 订阅状态变化
  subscribe(subscriber: Subscriber<T>): Unsubscribe {
    this.subscribers.add(subscriber);
    return () => this.subscribers.delete(subscriber);
  }

  // 通知所有订阅者
  private notify(prevState: T): void {
    this.subscribers.forEach(subscriber => subscriber(this.state, prevState));
  }
}

// 使用示例
interface UserState {
  user: User | null;
  isAuthenticated: boolean;
  permissions: string[];
}

const userStore = new Store<UserState>({
  user: null,
  isAuthenticated: false,
  permissions: [],
});

// 订阅状态变化
const unsubscribe = userStore.subscribe((state, prevState) => {
  console.log('State changed:', state);
});

// 更新状态
userStore.setState({ isAuthenticated: true });
userStore.setState(prev => ({ user: { id: 1, name: 'John' } }));
```

### 4.2 多Store拆分

```typescript
// 按功能域拆分Store
class UserStore extends Store<UserState> {
  constructor() {
    super({
      user: null,
      isAuthenticated: false,
      permissions: [],
    });
  }

  login(user: User): void {
    this.setState({
      user,
      isAuthenticated: true,
      permissions: user.permissions,
    });
  }

  logout(): void {
    this.setState({
      user: null,
      isAuthenticated: false,
      permissions: [],
    });
  }

  updateProfile(profile: Partial<User>): void {
    this.setState(prev => ({
      user: prev.user ? { ...prev.user, ...profile } : null,
    }));
  }
}

class UIStore extends Store<UIState> {
  constructor() {
    super({
      sidebarCollapsed: false,
      theme: 'light',
      notifications: [],
    });
  }

  toggleSidebar(): void {
    this.setState(prev => ({ sidebarCollapsed: !prev.sidebarCollapsed }));
  }

  setTheme(theme: 'light' | 'dark'): void {
    this.setState({ theme });
  }

  addNotification(notification: Omit<Notification, 'id'>): void {
    this.setState(prev => ({
      notifications: [...prev.notifications, { id: Date.now().toString(), ...notification }],
    }));
  }
}

// 导出单例
export const userStore = new UserStore();
export const uiStore = new UIStore();
```

### 4.3 派生状态（Computed）

```typescript
class CartStore extends Store<CartState> {
  private computedCache: Map<string, any> = new Map();

  constructor() {
    super({ items: [] });
    
    // 状态变化时清除缓存
    this.subscribe(() => this.computedCache.clear());
  }

  // 派生属性：商品总数
  get totalCount(): number {
    if (!this.computedCache.has('totalCount')) {
      this.computedCache.set('totalCount', 
        this.getState().items.reduce((sum, item) => sum + item.quantity, 0)
      );
    }
    return this.computedCache.get('totalCount');
  }

  // 派生属性：总价
  get totalPrice(): number {
    if (!this.computedCache.has('totalPrice')) {
      this.computedCache.set('totalPrice',
        this.getState().items.reduce((sum, item) => sum + item.price * item.quantity, 0)
      );
    }
    return this.computedCache.get('totalPrice');
  }

  // 派生属性：是否为空
  get isEmpty(): boolean {
    return this.getState().items.length === 0;
  }

  addItem(item: CartItem): void {
    this.setState(prev => {
      const existing = prev.items.find(i => i.id === item.id);
      if (existing) {
        return {
          items: prev.items.map(i =>
            i.id === item.id ? { ...i, quantity: i.quantity + item.quantity } : i
          ),
        };
      }
      return { items: [...prev.items, item] };
    });
  }
}
```

---

## 五、Redux风格实现

### 5.1 Store配置

```typescript
// Action类型
interface Action<T = any> {
  type: string;
  payload?: T;
}

// Reducer类型
type Reducer<S> = (state: S, action: Action) => S;

// 创建Store
function createStore<S>(
  reducer: Reducer<S>,
  initialState: S,
  enhancer?: (createStore: any) => any
) {
  let state = initialState;
  const listeners: Set<() => void> = new Set();

  function getState(): S {
    return state;
  }

  function dispatch(action: Action): void {
    state = reducer(state, action);
    listeners.forEach(listener => listener());
  }

  function subscribe(listener: () => void): () => void {
    listeners.add(listener);
    return () => listeners.delete(listener);
  }

  // 初始化
  dispatch({ type: '@@INIT' });

  return { getState, dispatch, subscribe };
}

// 组合Reducer
function combineReducers<S>(reducers: { [K in keyof S]: Reducer<S[K]> }): Reducer<S> {
  return (state: S, action: Action) => {
    const nextState = {} as S;
    
    for (const key in reducers) {
      nextState[key] = reducers[key](state[key], action);
    }
    
    return nextState;
  };
}
```

### 5.2 Slice创建

```typescript
// 创建Slice辅助函数
function createSlice<S, R extends Record<string, (state: S, payload?: any) => S>>({
  name,
  initialState,
  reducers,
}: {
  name: string;
  initialState: S;
  reducers: R;
}) {
  const actions = {} as { [K in keyof R]: (payload?: any) => Action };
  
  const reducer = (state: S = initialState, action: Action): S => {
    if (action.type.startsWith(`${name}/`)) {
      const reducerFn = reducers[action.type.replace(`${name}/`, '') as keyof R];
      if (reducerFn) {
        return reducerFn(state, action.payload);
      }
    }
    return state;
  };

  for (const key in reducers) {
    actions[key] = (payload?: any) => ({
      type: `${name}/${key}`,
      payload,
    });
  }

  return { name, reducer, actions };
}

// 使用
const userSlice = createSlice({
  name: 'user',
  initialState: { user: null as User | null, loading: false },
  reducers: {
    setUser: (state, user: User) => ({ ...state, user }),
    clearUser: (state) => ({ ...state, user: null }),
    setLoading: (state, loading: boolean) => ({ ...state, loading }),
  },
});

const rootReducer = combineReducers({
  user: userSlice.reducer,
});

const store = createStore(rootReducer, { user: { user: null, loading: false } });

// 使用Action
store.dispatch(userSlice.actions.setUser({ id: 1, name: 'John' }));
```

---

## 六、状态结构设计原则

### 6.1 规范化状态结构

```typescript
// ❌ 不推荐：嵌套结构
interface BadState {
  posts: Array<{
    id: string;
    title: string;
    author: { id: string; name: string };
    comments: Array<{ id: string; content: string }>;
  }>;
}

// ✅ 推荐：规范化结构
interface GoodState {
  posts: {
    byId: Record<string, Post>;
    allIds: string[];
  };
  users: {
    byId: Record<string, User>;
    allIds: string[];
  };
  comments: {
    byId: Record<string, Comment>;
    byPost: Record<string, string[]>;
  };
}

// 使用示例
const post = state.posts.byId[postId];
const author = state.users.byId[post.authorId];
const commentIds = state.comments.byPost[postId] || [];
const comments = commentIds.map(id => state.comments.byId[id]);
```

### 6.2 状态分类

```typescript
// 1. 服务端状态 - 使用专用库管理
// 2. 客户端全局状态 - 使用Store管理
interface ClientGlobalState {
  user: User | null;
  permissions: string[];
  theme: 'light' | 'dark';
  language: string;
  cart: CartItem[];
}

// 3. 客户端局部状态 - 组件内部管理
// 4. URL状态 - 路由管理
```

---

## 七、状态持久化

```typescript
class PersistentStore<T extends Record<string, any>> extends Store<T> {
  private storageKey: string;
  private storage: Storage;

  constructor(initialState: T, storageKey: string, storage: Storage = localStorage) {
    // 尝试从存储恢复
    const saved = storage.getItem(storageKey);
    const restoredState = saved ? { ...initialState, ...JSON.parse(saved) } : initialState;
    
    super(restoredState);
    this.storageKey = storageKey;
    this.storage = storage;

    // 订阅变化并保存
    this.subscribe((state) => {
      this.storage.setItem(this.storageKey, JSON.stringify(state));
    });
  }

  clear(): void {
    this.storage.removeItem(this.storageKey);
  }
}

// 使用
const persistentUserStore = new PersistentStore(
  { user: null, preferences: {} },
  'user-storage'
);
```

---

## 八、决策检查清单

- [ ] 区分了服务端状态和客户端状态
- [ ] 选择了合适的状态管理方案
- [ ] 状态结构设计扁平化
- [ ] 实现了不可变更新
- [ ] 配置了状态持久化（如需要）
- [ ] 异步状态处理完善

---

## 九、相关技能

- [API数据获取与缓存](./api-data-fetching-and-caching/SKILL.md) —— 服务端状态管理
- [前端组件设计模式](./frontend-component-design-patterns/SKILL.md) —— 组件状态设计
- [性能优化实战](./performance-optimization/SKILL.md) —— 状态更新优化

---

## 十、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，框架无关实现 |
