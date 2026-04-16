---
name: modal-dialog-management
description: >
  本技能用于实现弹窗/对话框管理，包括全局弹窗栈、命令式调用、动画、层级管理等。
  当需要设计弹窗系统、实现命令式弹窗调用、管理弹窗层级时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 弹窗/对话框管理

## 一、概述

### 1.1 这是什么

弹窗/对话框管理技能提供完整的弹窗系统解决方案，涵盖全局弹窗栈、命令式调用、动画效果、层级管理等。

### 1.2 适用场景

- ✅ 全局弹窗管理
- ✅ 命令式弹窗调用
- ✅ 弹窗动画效果
- ✅ 多弹窗层级管理
- ✅ 确认/提示对话框
- ❌ 简单内联展开/收起
- ❌ 页面跳转替代方案

### 1.3 核心原则

1. **统一入口** —— 全局管理弹窗状态
2. **命令式+声明式** —— 支持两种调用方式
3. **层级管理** —— 正确处理多弹窗叠加
4. **可访问性** —— 焦点管理和键盘导航

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要统一管理应用弹窗
- 需要命令式调用弹窗
- 需要弹窗动画效果
- 需要处理多弹窗层级
- 需要全局确认对话框

### 2.2 输入

- 弹窗类型定义
- 动画需求
- 层级管理需求
- 可访问性要求

### 2.3 输出

- 弹窗管理器
- 命令式API
- 动画组件
- 基础弹窗组件

---

## 三、全局弹窗管理

### 3.1 弹窗状态管理

```typescript
import { create } from 'zustand';

interface ModalItem {
  id: string;
  component: React.ComponentType<any>;
  props: Record<string, any>;
  options: ModalOptions;
}

interface ModalOptions {
  closable?: boolean;
  maskClosable?: boolean;
  zIndex?: number;
  animation?: 'fade' | 'slide' | 'zoom';
}

interface ModalState {
  modals: ModalItem[];
  open: <T>(component: React.ComponentType<T>, props?: T, options?: ModalOptions) => string;
  close: (id: string) => void;
  closeAll: () => void;
  update: (id: string, props: Record<string, any>) => void;
}

export const useModalStore = create<ModalState>((set, get) => ({
  modals: [],

  open: (component, props = {}, options = {}) => {
    const id = `modal-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
    
    set((state) => ({
      modals: [
        ...state.modals,
        { id, component, props, options },
      ],
    }));

    return id;
  },

  close: (id) => {
    set((state) => ({
      modals: state.modals.filter((m) => m.id !== id),
    }));
  },

  closeAll: () => {
    set({ modals: [] });
  },

  update: (id, props) => {
    set((state) => ({
      modals: state.modals.map((m) =>
        m.id === id ? { ...m, props: { ...m.props, ...props } } : m
      ),
    }));
  },
}));
```

### 3.2 弹窗容器组件

```typescript
export function ModalContainer() {
  const modals = useModalStore((state) => state.modals);
  const close = useModalStore((state) => state.close);

  return (
    <>
      {modals.map((modal, index) => (
        <ModalWrapper
          key={modal.id}
          modal={modal}
          onClose={() => close(modal.id)}
          zIndex={1000 + index * 10}
        />
      ))}
    </>
  );
}

function ModalWrapper({
  modal,
  onClose,
  zIndex,
}: {
  modal: ModalItem;
  onClose: () => void;
  zIndex: number;
}) {
  const { component: Component, props, options } = modal;

  return (
    <div className="modal-wrapper" style={{ zIndex }}>
      <div
        className="modal-mask"
        onClick={() => options.maskClosable !== false && onClose()}
      />
      <div className={`modal-content modal-animation-${options.animation || 'fade'}`}>
        <Component {...props} onClose={onClose} />
      </div>
    </div>
  );
}
```

---

## 四、命令式API

```typescript
// 命令式弹窗API
export const modal = {
  open: <T,>(component: React.ComponentType<T>, props?: T, options?: ModalOptions) => {
    return useModalStore.getState().open(component, props, options);
  },

  close: (id: string) => {
    useModalStore.getState().close(id);
  },

  closeAll: () => {
    useModalStore.getState().closeAll();
  },

  // 确认对话框
  confirm: (options: ConfirmOptions): Promise<boolean> => {
    return new Promise((resolve) => {
      const id = useModalStore.getState().open(
        ConfirmDialog,
        {
          ...options,
          onConfirm: () => {
            resolve(true);
            useModalStore.getState().close(id);
          },
          onCancel: () => {
            resolve(false);
            useModalStore.getState().close(id);
          },
        },
        { closable: false }
      );
    });
  },

  // 信息提示
  info: (options: MessageOptions) => {
    const id = useModalStore.getState().open(
      MessageDialog,
      {
        ...options,
        type: 'info',
        onClose: () => useModalStore.getState().close(id),
      },
      { maskClosable: true }
    );
    
    // 自动关闭
    if (options.duration) {
      setTimeout(() => {
        useModalStore.getState().close(id);
      }, options.duration);
    }
  },

  // 成功提示
  success: (message: string) => {
    modal.info({ message, type: 'success', duration: 3000 });
  },

  // 错误提示
  error: (message: string) => {
    modal.info({ message, type: 'error', duration: 5000 });
  },
};

// 使用示例
async function handleDelete() {
  const confirmed = await modal.confirm({
    title: '确认删除',
    content: '删除后无法恢复，是否继续？',
    okText: '删除',
    okType: 'danger',
  });

  if (confirmed) {
    await deleteItem();
    modal.success('删除成功');
  }
}
```

---

## 五、基础弹窗组件

### 5.1 确认对话框

```typescript
interface ConfirmDialogProps {
  title: string;
  content: React.ReactNode;
  okText?: string;
  cancelText?: string;
  okType?: 'primary' | 'danger' | 'default';
  onConfirm: () => void;
  onCancel: () => void;
}

export function ConfirmDialog({
  title,
  content,
  okText = '确定',
  cancelText = '取消',
  okType = 'primary',
  onConfirm,
  onCancel,
}: ConfirmDialogProps) {
  return (
    <div className="confirm-dialog">
      <div className="confirm-header">
        <h3>{title}</h3>
      </div>
      <div className="confirm-body">{content}</div>
      <div className="confirm-footer">
        <button onClick={onCancel}>{cancelText}</button>
        <button onClick={onConfirm} className={`btn-${okType}`}>
          {okText}
        </button>
      </div>
    </div>
  );
}
```

### 5.2 可复用Modal组件

```typescript
interface ModalProps {
  open: boolean;
  title?: React.ReactNode;
  children: React.ReactNode;
  footer?: React.ReactNode;
  width?: number | string;
  closable?: boolean;
  onClose: () => void;
  afterClose?: () => void;
}

export function Modal({
  open,
  title,
  children,
  footer,
  width = 520,
  closable = true,
  onClose,
  afterClose,
}: ModalProps) {
  const [visible, setVisible] = useState(open);
  const [animationClass, setAnimationClass] = useState('');

  useEffect(() => {
    if (open) {
      setVisible(true);
      requestAnimationFrame(() => {
        setAnimationClass('modal-enter');
      });
    } else {
      setAnimationClass('modal-exit');
      const timer = setTimeout(() => {
        setVisible(false);
        afterClose?.();
      }, 300);
      return () => clearTimeout(timer);
    }
  }, [open, afterClose]);

  if (!visible) return null;

  return (
    <div className={`modal-root ${animationClass}`}>
      <div className="modal-mask" onClick={closable ? onClose : undefined} />
      <div className="modal-wrap" onClick={closable ? onClose : undefined}>
        <div
          className="modal"
          style={{ width }}
          onClick={(e) => e.stopPropagation()}
        >
          {closable && (
            <button className="modal-close" onClick={onClose}>
              ×
            </button>
          )}
          {title && (
            <div className="modal-header">
              <div className="modal-title">{title}</div>
            </div>
          )}
          <div className="modal-body">{children}</div>
          {footer !== undefined && (
            <div className="modal-footer">{footer}</div>
          )}
        </div>
      </div>
    </div>
  );
}
```

---

## 六、动画效果

```css
/* 淡入淡出 */
.modal-enter .modal-mask {
  opacity: 0;
  animation: fadeIn 0.3s forwards;
}

.modal-enter .modal {
  opacity: 0;
  transform: scale(0.9);
  animation: zoomIn 0.3s forwards;
}

.modal-exit .modal-mask {
  animation: fadeOut 0.3s forwards;
}

.modal-exit .modal {
  animation: zoomOut 0.3s forwards;
}

@keyframes fadeIn {
  to { opacity: 1; }
}

@keyframes fadeOut {
  to { opacity: 0; }
}

@keyframes zoomIn {
  to {
    opacity: 1;
    transform: scale(1);
  }
}

@keyframes zoomOut {
  to {
    opacity: 0;
    transform: scale(0.9);
  }
}

/* 滑动 */
.modal-animation-slide.modal-enter .modal {
  transform: translateY(-50px);
  animation: slideDown 0.3s forwards;
}

@keyframes slideDown {
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
```

---

## 七、决策检查清单

- [ ] 实现了全局弹窗管理
- [ ] 支持命令式调用
- [ ] 弹窗动画效果完善
- [ ] 多弹窗层级管理正确
- [ ] 焦点管理完善
- [ ] 支持ESC关闭
- [ ] 点击遮罩关闭可配置
- [ ] 弹窗内容可滚动

---

## 八、相关技能

- [React组件设计模式](./react-component-design-patterns/SKILL.md) —— 弹窗组件设计
- [状态管理实现](./state-management-implementation/SKILL.md) —— 弹窗状态管理
- [性能优化实战](./performance-optimization/SKILL.md) —— 弹窗性能优化

---

## 九、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含全局弹窗管理、命令式API、动画效果 |
