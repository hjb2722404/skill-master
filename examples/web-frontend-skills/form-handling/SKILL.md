---
name: form-handling
description: >
  本技能用于实现表单处理，包括状态管理、验证、提交和错误处理。
  当需要创建或重构表单（登录、注册、配置等）时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 表单处理技能

## 一、概述

### 1.1 这是什么

表单处理的系统化方法，涵盖表单状态设计、验证逻辑、提交处理和用户反馈的完整实现指南。

### 1.2 适用场景

- ✅ 用户登录/注册表单
- ✅ 数据配置/设置表单
- ✅ 搜索/筛选表单
- ✅ 多步骤向导表单
- ❌ 复杂富文本编辑（需要专门技能）
- ❌ 文件上传表单（需要专门技能）

### 1.3 核心原则

1. **状态单一来源** —— 表单状态必须明确归属
2. **验证即时反馈** —— 用户功能可用优先，错误及时提示
3. **副作用清晰管理** —— 提交逻辑明确，清理完备
4. **渐进式复杂度** —— 简单表单不引入过度抽象

---

## 二、前置条件

### 2.1 知识要求

- 熟悉 JavaScript/TypeScript
- 理解 DOM 操作
- 了解异步操作处理

---

## 三、执行步骤

### 步骤1：确定表单状态范围

**决策问题**：表单状态应该放在哪里？

| 场景 | 推荐位置 | 理由 |
|------|----------|------|
| 简单表单，独立使用 | 类内部状态 | 封装性好，无需外部依赖 |
| 表单数据需要父级使用 | 事件回调传递 | 数据提升，便于统一管理 |
| 复杂表单，多字段关联 | 使用表单状态管理类 | 减少样板代码，优化性能 |

### 步骤2：设计表单状态结构

```typescript
interface FormState<T> {
  values: T;
  errors: Partial<Record<keyof T, string>>;
  touched: Partial<Record<keyof T, boolean>>;
  isSubmitting: boolean;
  submitError: string | null;
  isValid: boolean;
}
```

---

## 四、核心实现

### 4.1 表单管理器

```typescript
type ValidationRule<T> = {
  required?: boolean;
  minLength?: number;
  maxLength?: number;
  pattern?: RegExp;
  validate?: (value: any, values: T) => string | undefined;
};

type ValidationSchema<T> = Partial<Record<keyof T, ValidationRule<T>>>;

interface FormConfig<T> {
  initialValues: T;
  validationSchema?: ValidationSchema<T>;
  onSubmit: (values: T) => Promise<void> | void;
}

class FormManager<T extends Record<string, any>> {
  private state: FormState<T>;
  private config: FormConfig<T>;
  private listeners: Set<(state: FormState<T>) => void> = new Set();

  constructor(config: FormConfig<T>) {
    this.config = config;
    this.state = {
      values: { ...config.initialValues },
      errors: {},
      touched: {},
      isSubmitting: false,
      submitError: null,
      isValid: true,
    };
  }

  // 订阅状态变化
  subscribe(listener: (state: FormState<T>) => void): () => void {
    this.listeners.add(listener);
    listener(this.state);
    return () => this.listeners.delete(listener);
  }

  private notify(): void {
    this.listeners.forEach(listener => listener(this.state));
  }

  // 设置字段值
  setValue<K extends keyof T>(field: K, value: T[K]): void {
    const newValues = { ...this.state.values, [field]: value };
    const errors = this.validateField(field, value, newValues);
    
    this.state = {
      ...this.state,
      values: newValues,
      errors: { ...this.state.errors, [field]: errors },
      isValid: this.validateAll(newValues),
    };
    
    this.notify();
  }

  // 设置触摸状态
  setTouched<K extends keyof T>(field: K, touched: boolean = true): void {
    this.state = {
      ...this.state,
      touched: { ...this.state.touched, [field]: touched },
    };
    this.notify();
  }

  // 验证单个字段
  private validateField<K extends keyof T>(
    field: K,
    value: any,
    values: T
  ): string | undefined {
    const rules = this.config.validationSchema?.[field];
    if (!rules) return undefined;

    if (rules.required && !value) {
      return '此字段为必填项';
    }

    if (rules.minLength && String(value).length < rules.minLength) {
      return `最少需要 ${rules.minLength} 个字符`;
    }

    if (rules.maxLength && String(value).length > rules.maxLength) {
      return `最多允许 ${rules.maxLength} 个字符`;
    }

    if (rules.pattern && !rules.pattern.test(String(value))) {
      return '格式不正确';
    }

    if (rules.validate) {
      return rules.validate(value, values);
    }

    return undefined;
  }

  // 验证所有字段
  private validateAll(values: T): boolean {
    if (!this.config.validationSchema) return true;

    for (const field in this.config.validationSchema) {
      const error = this.validateField(field, values[field], values);
      if (error) return false;
    }

    return true;
  }

  // 提交表单
  async submit(): Promise<void> {
    // 标记所有字段为已触摸
    const allTouched = Object.keys(this.state.values).reduce((acc, key) => {
      acc[key as keyof T] = true;
      return acc;
    }, {} as Record<keyof T, boolean>);

    this.state = {
      ...this.state,
      touched: allTouched,
      isSubmitting: true,
      submitError: null,
    };
    this.notify();

    try {
      await this.config.onSubmit(this.state.values);
      this.state = { ...this.state, isSubmitting: false };
    } catch (error) {
      this.state = {
        ...this.state,
        isSubmitting: false,
        submitError: error instanceof Error ? error.message : '提交失败',
      };
    }
    
    this.notify();
  }

  // 重置表单
  reset(): void {
    this.state = {
      values: { ...this.config.initialValues },
      errors: {},
      touched: {},
      isSubmitting: false,
      submitError: null,
      isValid: true,
    };
    this.notify();
  }

  getState(): FormState<T> {
    return { ...this.state };
  }
}
```

### 4.2 表单渲染器

```typescript
class FormRenderer<T extends Record<string, any>> {
  private form: FormManager<T>;
  private container: HTMLElement;
  private fields: Map<keyof T, HTMLElement> = new Map();
  private unsubscribe: () => void;

  constructor(container: HTMLElement, form: FormManager<T>) {
    this.container = container;
    this.form = form;
    this.unsubscribe = form.subscribe(() => this.update());
  }

  createField<K extends keyof T>(
    field: K,
    config: {
      label: string;
      type?: string;
      placeholder?: string;
    }
  ): HTMLElement {
    const wrapper = document.createElement('div');
    wrapper.className = 'form-field';

    const label = document.createElement('label');
    label.textContent = config.label;
    wrapper.appendChild(label);

    const input = document.createElement('input');
    input.type = config.type || 'text';
    input.placeholder = config.placeholder || '';
    input.name = String(field);
    
    input.addEventListener('input', (e) => {
      this.form.setValue(field, (e.target as HTMLInputElement).value as T[K]);
    });
    
    input.addEventListener('blur', () => {
      this.form.setTouched(field);
    });

    wrapper.appendChild(input);
    this.fields.set(field, wrapper);

    const errorEl = document.createElement('span');
    errorEl.className = 'error-message';
    wrapper.appendChild(errorEl);

    return wrapper;
  }

  private update(): void {
    const state = this.form.getState();

    this.fields.forEach((wrapper, field) => {
      const input = wrapper.querySelector('input') as HTMLInputElement;
      const errorEl = wrapper.querySelector('.error-message') as HTMLElement;

      // 更新值
      if (input.value !== state.values[field]) {
        input.value = String(state.values[field] || '');
      }

      // 显示错误
      const error = state.errors[field];
      const touched = state.touched[field];
      
      if (error && touched) {
        errorEl.textContent = error;
        wrapper.classList.add('has-error');
      } else {
        errorEl.textContent = '';
        wrapper.classList.remove('has-error');
      }
    });
  }

  createSubmitButton(text: string): HTMLButtonElement {
    const button = document.createElement('button');
    button.type = 'submit';
    button.textContent = text;
    
    button.addEventListener('click', (e) => {
      e.preventDefault();
      this.form.submit();
    });

    return button;
  }

  destroy(): void {
    this.unsubscribe();
    this.fields.clear();
  }
}
```

### 4.3 使用示例

```typescript
// 创建表单
const loginForm = new FormManager({
  initialValues: {
    email: '',
    password: '',
  },
  validationSchema: {
    email: {
      required: true,
      pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/,
    },
    password: {
      required: true,
      minLength: 6,
    },
  },
  async onSubmit(values) {
    const response = await fetch('/api/login', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(values),
    });
    
    if (!response.ok) {
      throw new Error('登录失败');
    }
  },
});

// 渲染表单
const container = document.getElementById('login-form')!;
const renderer = new FormRenderer(container, loginForm);

renderer.createField('email', {
  label: '邮箱',
  type: 'email',
  placeholder: '请输入邮箱',
});

renderer.createField('password', {
  label: '密码',
  type: 'password',
  placeholder: '请输入密码',
});

container.appendChild(renderer.createSubmitButton('登录'));
```

---

## 五、决策检查清单

- [ ] 表单状态结构设计合理
- [ ] 验证规则配置完善
- [ ] 错误提示时机恰当
- [ ] 提交状态处理完善
- [ ] 表单可重置
- [ ] 内存泄漏已处理

---

## 六、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，框架无关实现 |
