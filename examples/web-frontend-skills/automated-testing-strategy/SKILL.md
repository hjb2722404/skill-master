---
name: automated-testing-strategy
description: >
  本技能用于实现自动化测试策略，包括单元测试、集成测试、E2E测试、测试覆盖率等。
  当需要建立测试体系、编写测试用例、配置测试覆盖率时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 自动化测试策略

## 一、概述

### 1.1 这是什么

自动化测试策略技能提供完整的测试解决方案，涵盖单元测试、集成测试、E2E测试、测试覆盖率等。

### 1.2 适用场景

- ✅ 核心功能验证
- ✅ 回归测试
- ✅ 持续集成
- ✅ 代码质量保证
- ✅ 重构保障
- ❌ 一次性原型
- ❌ 快速验证想法

### 1.3 核心原则

1. **测试金字塔** —— 更多单元测试，更少E2E测试
2. **可维护性** —— 测试代码也是代码
3. **确定性** —— 测试应该稳定可靠
4. **快速反馈** —— 测试执行要快

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要建立测试体系
- 需要编写测试用例
- 需要配置测试覆盖率
- 需要集成CI/CD
- 需要保证代码质量

### 2.2 输入

- 测试范围定义
- 测试框架选型
- 覆盖率目标
- CI/CD配置

### 2.3 输出

- 测试配置
- 测试用例
- 覆盖率报告
- CI集成

---

## 三、测试金字塔

```
    /\
   /  \     E2E测试 (10%)
  /----\
 /      \   集成测试 (30%)
/--------\
/          \ 单元测试 (60%)
------------
```

| 测试类型 | 工具 | 范围 | 速度 | 成本 |
|----------|------|------|------|------|
| 单元测试 | Jest/Vitest | 函数/组件 | 快 | 低 |
| 集成测试 | React Testing Library | 组件交互 | 中 | 中 |
| E2E测试 | Playwright/Cypress | 完整流程 | 慢 | 高 |

---

## 四、单元测试

### 4.1 Jest/Vitest配置

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './src/test/setup.ts',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/test/',
      ],
    },
  },
});

// src/test/setup.ts
import '@testing-library/jest-dom';
import { vi } from 'vitest';

// 全局mock
global.fetch = vi.fn();
```

### 4.2 组件测试

```typescript
// Button.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { Button } from './Button';

describe('Button', () => {
  it('renders correctly', () => {
    render(<Button>Click me</Button>);
    expect(screen.getByText('Click me')).toBeInTheDocument();
  });

  it('handles click events', () => {
    const handleClick = vi.fn();
    render(<Button onClick={handleClick}>Click me</Button>);
    
    fireEvent.click(screen.getByText('Click me'));
    expect(handleClick).toHaveBeenCalledTimes(1);
  });

  it('is disabled when loading', () => {
    render(<Button loading>Click me</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });

  it('applies variant styles', () => {
    const { rerender } = render(<Button variant="primary">Button</Button>);
    expect(screen.getByRole('button')).toHaveClass('btn-primary');
    
    rerender(<Button variant="danger">Button</Button>);
    expect(screen.getByRole('button')).toHaveClass('btn-danger');
  });
});
```

### 4.3 Hook测试

```typescript
// useCounter.test.ts
import { renderHook, act } from '@testing-library/react';
import { describe, it, expect } from 'vitest';
import { useCounter } from './useCounter';

describe('useCounter', () => {
  it('initializes with default value', () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  it('initializes with provided value', () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  it('increments count', () => {
    const { result } = renderHook(() => useCounter());
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });

  it('decrements count', () => {
    const { result } = renderHook(() => useCounter(5));
    
    act(() => {
      result.current.decrement();
    });
    
    expect(result.current.count).toBe(4);
  });
});
```

---

## 五、集成测试

```typescript
// UserForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { UserForm } from './UserForm';

const createTestQueryClient = () => new QueryClient({
  defaultOptions: {
    queries: { retry: false },
    mutations: { retry: false },
  },
});

const renderWithProviders = (ui: React.ReactElement) => {
  const queryClient = createTestQueryClient();
  return render(
    <QueryClientProvider client={queryClient}>
      {ui}
    </QueryClientProvider>
  );
};

describe('UserForm', () => {
  it('submits form with valid data', async () => {
    const onSubmit = vi.fn();
    renderWithProviders(<UserForm onSubmit={onSubmit} />);

    fireEvent.change(screen.getByLabelText('姓名'), {
      target: { value: 'John' },
    });
    fireEvent.change(screen.getByLabelText('邮箱'), {
      target: { value: 'john@example.com' },
    });

    fireEvent.click(screen.getByText('提交'));

    await waitFor(() => {
      expect(onSubmit).toHaveBeenCalledWith({
        name: 'John',
        email: 'john@example.com',
      });
    });
  });

  it('shows validation errors', async () => {
    renderWithProviders(<UserForm />);

    fireEvent.click(screen.getByText('提交'));

    await waitFor(() => {
      expect(screen.getByText('姓名不能为空')).toBeInTheDocument();
      expect(screen.getByText('邮箱格式不正确')).toBeInTheDocument();
    });
  });
});
```

---

## 六、E2E测试

### 6.1 Playwright配置

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### 6.2 E2E测试用例

```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Login', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });

  test('successful login', async ({ page }) => {
    await page.fill('[name="email"]', 'user@example.com');
    await page.fill('[name="password"]', 'password123');
    await page.click('button[type="submit"]');

    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('text=欢迎回来')).toBeVisible();
  });

  test('shows error for invalid credentials', async ({ page }) => {
    await page.fill('[name="email"]', 'wrong@example.com');
    await page.fill('[name="password"]', 'wrongpassword');
    await page.click('button[type="submit"]');

    await expect(page.locator('text=邮箱或密码错误')).toBeVisible();
  });

  test('validates required fields', async ({ page }) => {
    await page.click('button[type="submit"]');

    await expect(page.locator('text=邮箱不能为空')).toBeVisible();
    await expect(page.locator('text=密码不能为空')).toBeVisible();
  });
});

// e2e/checkout.spec.ts
import { test, expect } from '@playwright/test';

test('complete checkout flow', async ({ page }) => {
  // 添加商品到购物车
  await page.goto('/products');
  await page.click('[data-testid="add-to-cart-1"]');
  await page.click('[data-testid="add-to-cart-2"]');

  // 进入购物车
  await page.click('[data-testid="cart-icon"]');
  await expect(page.locator('[data-testid="cart-count"]')).toHaveText('2');

  // 结算
  await page.click('text=去结算');
  await page.fill('[name="address"]', '测试地址');
  await page.click('text=提交订单');

  // 验证订单成功
  await expect(page).toHaveURL(/\/order\/\d+/);
  await expect(page.locator('text=订单提交成功')).toBeVisible();
});
```

---

## 七、测试最佳实践

### 7.1 测试原则

```typescript
// ✅ 好的测试：关注行为而非实现
test('shows loading state while fetching', async () => {
  render(<UserList />);
  expect(screen.getByText('加载中...')).toBeInTheDocument();
  await waitFor(() => {
    expect(screen.getByText('John')).toBeInTheDocument();
  });
});

// ❌ 坏的测试：测试实现细节
test('calls setState with correct arguments', () => {
  const setState = vi.fn();
  // 测试实现细节，重构时容易失败
});
```

### 7.2 Mock策略

```typescript
// MSW (Mock Service Worker)
// src/test/mocks/handlers.ts
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/users', (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json([
        { id: 1, name: 'John' },
        { id: 2, name: 'Jane' },
      ])
    );
  }),

  rest.post('/api/users', async (req, res, ctx) => {
    const body = await req.json();
    return res(
      ctx.status(201),
      ctx.json({ id: 3, ...body })
    );
  }),
];

// src/test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// src/test/setup.ts
import { server } from './mocks/server';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

---

## 八、CI集成

```yaml
# .github/workflows/test.yml
name: Test

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      
      - name: Run integration tests
        run: npm run test:integration
      
      - name: Run E2E tests
        run: npm run test:e2e
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/lcov.info
```

---

## 九、决策检查清单

- [ ] 测试金字塔结构合理
- [ ] 单元测试覆盖率达标（>80%）
- [ ] 集成测试覆盖关键流程
- [ ] E2E测试覆盖用户旅程
- [ ] Mock策略合理
- [ ] CI集成完成
- [ ] 测试运行速度快
- [ ] 测试稳定可靠

---

## 十、相关技能

- [React组件设计模式](./react-component-design-patterns/SKILL.md) —— 可测试组件设计
- [性能优化实战](./performance-optimization/SKILL.md) —— 性能测试
- [错误边界与降级](./error-boundary-and-fallback/SKILL.md) —— 错误测试

---

## 十一、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含单元测试、集成测试、E2E测试 |
