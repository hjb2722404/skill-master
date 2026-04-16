---
name: micro-frontend-architecture
description: >
  本技能用于实现微前端架构，包括qiankun/module-federation、应用通信、样式隔离等。
  当需要设计微前端架构、实现应用集成、处理应用间通信时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 微前端架构

## 一、概述

### 1.1 这是什么

微前端架构技能提供完整的微前端解决方案，涵盖qiankun/module-federation等方案、应用通信、样式隔离等。

### 1.2 适用场景

- ✅ 大型应用拆分
- ✅ 多团队协作
- ✅ 技术栈异构
- ✅ 独立部署
- ✅ 遗留系统整合
- ❌ 小型项目（<5个模块）
- ❌ 单一团队维护

### 1.3 核心原则

1. **技术无关** —— 子应用技术栈独立
2. **独立开发** —— 团队独立开发和部署
3. **运行时集成** —— 浏览器端动态加载
4. **样式隔离** —— 避免样式冲突

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要拆分大型应用
- 多团队并行开发
- 需要技术栈升级过渡
- 需要整合遗留系统
- 需要独立部署

### 2.2 输入

- 应用拆分方案
- 技术栈选型
- 通信需求
- 部署策略

### 2.3 输出

- 微前端架构设计
- 主应用配置
- 子应用改造
- 通信机制

---

## 三、方案选择

| 方案 | 适用场景 | 优点 | 缺点 |
|------|----------|------|------|
| qiankun | 传统应用改造 | 成熟、文档丰富 | 需要改造 |
| Module Federation | 新应用 | 原生支持、性能好 | 需要webpack5 |
| iframe | 简单集成 | 完全隔离 | 体验差 |
| Web Components | 组件级 | 标准化 | 浏览器支持 |

---

## 四、qiankun实现

### 4.1 主应用配置

```typescript
// main-app/src/main.ts
import { registerMicroApps, start, setDefaultMountApp } from 'qiankun';

registerMicroApps([
  {
    name: 'app1',
    entry: '//localhost:8081',
    container: '#subapp-container',
    activeRule: '/app1',
    props: { token: getToken() },
  },
  {
    name: 'app2',
    entry: '//localhost:8082',
    container: '#subapp-container',
    activeRule: '/app2',
  },
]);

setDefaultMountApp('/app1');

start({
  sandbox: {
    strictStyleIsolation: true,
    experimentalStyleIsolation: true,
  },
});
```

### 4.2 子应用改造

```typescript
// sub-app/src/main.ts
import { createApp } from 'vue';
import App from './App.vue';
import router from './router';

let instance: any = null;

function render(props: any = {}) {
  const { container } = props;
  
  instance = createApp(App);
  instance.use(router);
  instance.mount(container ? container.querySelector('#app') : '#app');
}

// 独立运行
if (!window.__POWERED_BY_QIANKUN__) {
  render();
}

// qiankun生命周期
export async function bootstrap() {
  console.log('app bootstraped');
}

export async function mount(props: any) {
  render(props);
}

export async function unmount() {
  instance.unmount();
  instance = null;
}
```

### 4.3 子应用webpack配置

```javascript
// sub-app/vue.config.js
const { name } = require('./package.json');

module.exports = {
  devServer: {
    port: 8081,
    headers: {
      'Access-Control-Allow-Origin': '*',
    },
  },
  configureWebpack: {
    output: {
      library: `${name}-[name]`,
      libraryTarget: 'umd',
      jsonpFunction: `webpackJsonp_${name}`,
    },
  },
};
```

---

## 五、应用通信

### 5.1 全局状态

```typescript
// shared/store.ts
import { initGlobalState, MicroAppStateActions } from 'qiankun';

const initialState = {
  user: null,
  token: '',
  permissions: [],
};

const actions: MicroAppStateActions = initGlobalState(initialState);

export default actions;

// 主应用设置状态
actions.setGlobalState({ user: { name: 'John' } });

// 子应用监听状态
actions.onGlobalStateChange((state, prev) => {
  console.log('state changed:', state);
});
```

### 5.2 事件总线

```typescript
// shared/eventBus.ts
type EventHandler = (data: any) => void;

class EventBus {
  private events: Map<string, EventHandler[]> = new Map();

  on(event: string, handler: EventHandler) {
    if (!this.events.has(event)) {
      this.events.set(event, []);
    }
    this.events.get(event)!.push(handler);
  }

  off(event: string, handler: EventHandler) {
    const handlers = this.events.get(event);
    if (handlers) {
      const index = handlers.indexOf(handler);
      if (index > -1) handlers.splice(index, 1);
    }
  }

  emit(event: string, data?: any) {
    const handlers = this.events.get(event);
    handlers?.forEach((handler) => handler(data));
  }
}

export const eventBus = new EventBus();

// 使用
// 子应用A
import { eventBus } from '@shared/eventBus';
eventBus.emit('user:login', { userId: 1 });

// 子应用B
eventBus.on('user:login', (data) => {
  console.log('User logged in:', data);
});
```

---

## 六、样式隔离

```typescript
// qiankun配置
start({
  sandbox: {
    // 严格样式隔离（Shadow DOM）
    strictStyleIsolation: true,
    // 实验性样式隔离（前缀）
    experimentalStyleIsolation: true,
  },
});

// CSS Module
// sub-app/src/App.vue
<style module>
.container {
  padding: 20px;
}
</style>

// BEM命名规范
// sub-app/src/components/Button/index.css
.app1-button {
  background: blue;
}
.app1-button--primary {
  background: green;
}
```

---

## 七、Module Federation

```javascript
// host-app/webpack.config.js
const { ModuleFederationPlugin } = require('webpack').container;

module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'host',
      remotes: {
        app1: 'app1@http://localhost:3001/remoteEntry.js',
        app2: 'app2@http://localhost:3002/remoteEntry.js',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
      },
    }),
  ],
};

// remote-app/webpack.config.js
module.exports = {
  plugins: [
    new ModuleFederationPlugin({
      name: 'app1',
      filename: 'remoteEntry.js',
      exposes: {
        './Button': './src/components/Button',
        './utils': './src/utils',
      },
      shared: {
        react: { singleton: true },
        'react-dom': { singleton: true },
      },
    }),
  ],
};

// 使用远程组件
const RemoteButton = React.lazy(() => import('app1/Button'));

function App() {
  return (
    <React.Suspense fallback="Loading...">
      <RemoteButton />
    </React.Suspense>
  );
}
```

---

## 八、决策检查清单

- [ ] 微前端方案选择合理
- [ ] 主应用配置正确
- [ ] 子应用生命周期完善
- [ ] 应用间通信机制建立
- [ ] 样式隔离配置正确
- [ ] 路由跳转正常
- [ ] 公共依赖共享
- [ ] 错误边界处理

---

## 九、相关技能

- [路由与导航管理](./routing-and-navigation/SKILL.md) —— 微前端路由
- [状态管理实现](./state-management-implementation/SKILL.md) —— 全局状态
- [懒加载与代码分割](./lazy-loading-and-code-splitting/SKILL.md) —— 动态加载

---

## 十、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含qiankun和Module Federation实现 |
