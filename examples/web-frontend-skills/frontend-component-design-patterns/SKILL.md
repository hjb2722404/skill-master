---
name: frontend-component-design-patterns
description: >
  本技能用于设计和实现高质量的前端组件，涵盖组件设计原则、状态管理、通信模式等。
  当需要设计可复用组件、选择组件通信方式、优化组件结构时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 前端组件设计模式

## 一、概述

### 1.1 这是什么

前端组件设计模式技能提供系统性的组件设计方法论，帮助开发者创建高内聚、低耦合、可复用、可维护的组件。

### 1.2 适用场景

- ✅ 设计可复用的UI组件库
- ✅ 实现复杂的复合组件（如Tabs、Modal、Select）
- ✅ 选择组件状态管理方式
- ✅ 设计组件间的通信机制
- ❌ 简单的一次性组件（过度设计）

### 1.3 核心原则

1. **单一职责** —— 每个组件只做一件事
2. **开闭原则** —— 对扩展开放，对修改关闭
3. **组合优于继承** —— 使用配置和插槽组合功能
4. **显式优于隐式** —— API设计清晰明确

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 设计需要在多个地方复用的组件
- 实现复杂的交互组件
- 需要解耦组件逻辑和UI表现
- 需要设计组件的扩展点

### 2.2 输入

- 组件功能需求
- 复用范围预期
- 状态管理复杂度
- 定制化需求程度

### 2.3 输出

- 组件接口设计
- 组件结构划分
- 状态管理方案
- 使用示例代码

---

## 三、组件设计模式

### 3.1 受控 vs 非受控组件

#### 决策矩阵

| 场景 | 推荐模式 | 理由 |
|------|----------|------|
| 简单表单输入 | 非受控 | 代码简洁，无需外部状态 |
| 需要即时验证 | 受控 | 可以实时监听值变化 |
| 需要程序化控制 | 受控 | 通过外部状态控制值 |
| 组件库设计 | 支持两种 | 提供灵活的使用方式 |

#### 非受控组件实现

```typescript
interface InputConfig {
  defaultValue?: string;
  placeholder?: string;
  onChange?: (value: string) => void;
}

class UncontrolledInput {
  private element: HTMLInputElement;
  private config: InputConfig;

  constructor(container: HTMLElement, config: InputConfig = {}) {
    this.config = config;
    this.element = document.createElement('input');
    this.element.type = 'text';
    this.element.value = config.defaultValue || '';
    this.element.placeholder = config.placeholder || '';
    
    this.element.addEventListener('input', this.handleInput.bind(this));
    container.appendChild(this.element);
  }

  private handleInput(e: Event) {
    const value = (e.target as HTMLInputElement).value;
    this.config.onChange?.(value);
  }

  getValue(): string {
    return this.element.value;
  }

  setValue(value: string): void {
    this.element.value = value;
  }

  focus(): void {
    this.element.focus();
  }

  destroy(): void {
    this.element.removeEventListener('input', this.handleInput.bind(this));
    this.element.remove();
  }
}
```

#### 受控组件实现

```typescript
interface ControlledInputConfig {
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
  disabled?: boolean;
}

class ControlledInput {
  private element: HTMLInputElement;
  private config: ControlledInputConfig;

  constructor(container: HTMLElement, config: ControlledInputConfig) {
    this.config = config;
    this.element = document.createElement('input');
    this.element.type = 'text';
    this.element.placeholder = config.placeholder || '';
    this.element.disabled = config.disabled || false;
    
    this.element.addEventListener('input', this.handleInput.bind(this));
    container.appendChild(this.element);
    this.update();
  }

  private handleInput(e: Event): void {
    const value = (e.target as HTMLInputElement).value;
    this.config.onChange(value);
  }

  update(): void {
    if (this.element.value !== this.config.value) {
      this.element.value = this.config.value;
    }
    this.element.disabled = this.config.disabled || false;
  }

  destroy(): void {
    this.element.removeEventListener('input', this.handleInput.bind(this));
    this.element.remove();
  }
}
```

### 3.2 复合组件模式

适用于：需要多个子组件协同工作的场景

```typescript
// 上下文管理
interface TabsContext {
  activeTab: string;
  setActiveTab: (id: string) => void;
}

class TabsManager {
  private context: TabsContext;
  private listeners: Set<(ctx: TabsContext) => void> = new Set();

  constructor(defaultTab: string = '') {
    this.context = {
      activeTab: defaultTab,
      setActiveTab: (id: string) => {
        this.context.activeTab = id;
        this.notify();
      },
    };
  }

  subscribe(listener: (ctx: TabsContext) => void): () => void {
    this.listeners.add(listener);
    listener(this.context);
    return () => this.listeners.delete(listener);
  }

  private notify(): void {
    this.listeners.forEach(listener => listener(this.context));
  }

  getContext(): TabsContext {
    return this.context;
  }
}

// Tabs组件
interface TabsConfig {
  defaultTab?: string;
  container: HTMLElement;
}

class Tabs {
  private manager: TabsManager;
  private container: HTMLElement;
  private unsubscribers: Array<() => void> = [];

  constructor(config: TabsConfig) {
    this.container = config.container;
    this.manager = new TabsManager(config.defaultTab);
    this.container.className = 'tabs';
  }

  createTabList(): HTMLElement {
    const list = document.createElement('div');
    list.className = 'tab-list';
    list.setAttribute('role', 'tablist');
    return list;
  }

  createTab(id: string, label: string, disabled: boolean = false): HTMLElement {
    const tab = document.createElement('button');
    tab.className = 'tab';
    tab.textContent = label;
    tab.setAttribute('role', 'tab');
    tab.disabled = disabled;

    const unsubscribe = this.manager.subscribe(ctx => {
      const isActive = ctx.activeTab === id;
      tab.setAttribute('aria-selected', String(isActive));
      tab.classList.toggle('active', isActive);
    });
    this.unsubscribers.push(unsubscribe);

    tab.addEventListener('click', () => {
      if (!disabled) this.manager.getContext().setActiveTab(id);
    });

    return tab;
  }

  createTabPanel(id: string, content: HTMLElement): HTMLElement {
    const panel = document.createElement('div');
    panel.className = 'tab-panel';
    panel.setAttribute('role', 'tabpanel');
    panel.appendChild(content);

    const unsubscribe = this.manager.subscribe(ctx => {
      const isActive = ctx.activeTab === id;
      panel.style.display = isActive ? 'block' : 'none';
    });
    this.unsubscribers.push(unsubscribe);

    return panel;
  }

  destroy(): void {
    this.unsubscribers.forEach(unsub => unsub());
    this.unsubscribers = [];
  }
}
```

### 3.3 观察者模式

适用于：需要共享逻辑但允许自定义渲染的场景

```typescript
interface ToggleState {
  on: boolean;
}

interface ToggleActions {
  toggle: () => void;
  setOn: (value: boolean) => void;
}

type ToggleListener = (state: ToggleState & ToggleActions) => void;

class ToggleController {
  private state: ToggleState = { on: false };
  private listeners: Set<ToggleListener> = new Set();

  constructor(defaultValue: boolean = false) {
    this.state = { on: defaultValue };
  }

  subscribe(listener: ToggleListener): () => void {
    this.listeners.add(listener);
    this.notify();
    return () => this.listeners.delete(listener);
  }

  private notify(): void {
    const actions: ToggleActions = {
      toggle: () => this.setState({ on: !this.state.on }),
      setOn: (value: boolean) => this.setState({ on: value }),
    };
    this.listeners.forEach(listener => listener({ ...this.state, ...actions }));
  }

  private setState(newState: Partial<ToggleState>): void {
    this.state = { ...this.state, ...newState };
    this.notify();
  }

  getState(): ToggleState {
    return { ...this.state };
  }
}

// 使用
const toggle = new ToggleController(false);
const unsubscribe = toggle.subscribe(({ on, toggle: doToggle }) => {
  console.log('Toggle state:', on);
  // 更新UI
});
```

### 3.4 插槽模式

适用于：需要灵活布局的组件

```typescript
interface CardSlots {
  header?: HTMLElement;
  footer?: HTMLElement;
  body: HTMLElement;
}

interface CardConfig {
  slots: CardSlots;
  className?: string;
}

class Card {
  private element: HTMLElement;

  constructor(config: CardConfig) {
    this.element = document.createElement('div');
    this.element.className = `card ${config.className || ''}`;

    if (config.slots.header) {
      const header = document.createElement('div');
      header.className = 'card-header';
      header.appendChild(config.slots.header);
      this.element.appendChild(header);
    }

    const body = document.createElement('div');
    body.className = 'card-body';
    body.appendChild(config.slots.body);
    this.element.appendChild(body);

    if (config.slots.footer) {
      const footer = document.createElement('div');
      footer.className = 'card-footer';
      footer.appendChild(config.slots.footer);
      this.element.appendChild(footer);
    }
  }

  mount(container: HTMLElement): void {
    container.appendChild(this.element);
  }

  unmount(): void {
    this.element.remove();
  }
}

// 使用
const card = new Card({
  slots: {
    header: document.createElement('h2'),
    body: document.createElement('p'),
    footer: document.createElement('button'),
  },
});
card.mount(document.body);
```

---

## 四、Props设计原则

### 4.1 命名规范

```typescript
// ✅ 好的命名
interface ButtonProps {
  variant: 'primary' | 'secondary' | 'danger';
  size: 'sm' | 'md' | 'lg';
  disabled: boolean;
  loading: boolean;
  onClick: () => void;
  children: Node;
}

// ❌ 避免
interface ButtonProps {
  type: string;
  isDisable: boolean;
  handleClick: () => void;
}
```

### 4.2 接口设计检查清单

- [ ] Props有清晰的类型定义
- [ ] 可选属性有合理的默认值
- [ ] 事件处理器使用标准命名（onXxx）
- [ ] 复杂对象使用接口而非any

---

## 五、常见问题

### 5.1 何时使用哪种模式？

| 模式 | 最佳场景 | 避免场景 |
|------|----------|----------|
| 受控组件 | 表单验证、即时反馈 | 简单一次性输入 |
| 复合组件 | Tabs、Modal、Select | 简单独立组件 |
| 观察者模式 | 逻辑共享+自定义渲染 | 简单状态管理 |
| 插槽模式 | 灵活布局卡片 | 固定结构组件 |

---

## 六、相关技能

- [表单处理](./react-form-handling/SKILL.md) —— 表单组件设计
- [状态管理实现](./state-management-implementation/SKILL.md) —— 组件状态与全局状态
- [性能优化实战](./performance-optimization/SKILL.md) —— 组件性能优化

---

## 七、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，框架无关实现 |
