---
name: access-control-implementation
description: >
  本技能用于实现前端权限控制，包括路由权限、按钮权限、数据权限、动态菜单等。
  当需要设计权限系统、实现权限控制逻辑、处理动态权限时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 权限控制实现

## 一、概述

### 1.1 这是什么

权限控制实现技能提供完整的前端权限管理方案，涵盖路由权限、按钮权限、数据权限、动态菜单等。

### 1.2 适用场景

- ✅ 基于角色的访问控制(RBAC)
- ✅ 路由级别权限控制
- ✅ 按钮/功能级别权限控制
- ✅ 数据字段级别权限
- ✅ 动态菜单生成
- ❌ 纯静态网站
- ❌ 无用户系统的应用

### 1.3 核心原则

1. **权限前置** —— 在入口处拦截无权限访问
2. **最小权限** —— 默认无权限，按需授予
3. **前后端一致** —— 前端权限控制不能替代后端验证
4. **可扩展性** —— 支持多种权限模型

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要设计权限系统架构
- 需要实现路由权限控制
- 需要控制按钮/功能可见性
- 需要根据权限动态生成菜单
- 需要实现数据权限过滤

### 2.2 输入

- 权限模型设计（RBAC/ABAC）
- 用户角色定义
- 权限资源列表
- 权限数据接口

### 2.3 输出

- 权限控制组件
- 权限验证Hook
- 动态菜单生成逻辑
- 权限指令/装饰器

---

## 三、权限模型

### 3.1 RBAC模型（基于角色）

```typescript
// 权限实体定义
interface Permission {
  id: string;
  code: string;        // 权限标识，如 'user:create'
  name: string;        // 权限名称
  resource: string;    // 资源类型，如 'user'
  action: string;      // 操作类型，如 'create'
}

interface Role {
  id: string;
  name: string;
  permissions: string[]; // 权限code列表
}

interface User {
  id: string;
  username: string;
  roles: string[];       // 角色id列表
  permissions: string[]; // 直接分配的权限
}

// 权限判断
function hasPermission(user: User, permissionCode: string): boolean {
  // 1. 检查直接权限
  if (user.permissions.includes(permissionCode)) {
    return true;
  }
  
  // 2. 检查角色权限（需要获取角色详情）
  // 实际应用中通常后端返回合并后的权限列表
  return false;
}
```

### 3.2 权限码设计规范

```typescript
// 权限码格式: {resource}:{action}
// resource: 模块/资源名称
// action: 操作类型

const PERMISSIONS = {
  // 用户管理
  USER_VIEW: 'user:view',
  USER_CREATE: 'user:create',
  USER_UPDATE: 'user:update',
  USER_DELETE: 'user:delete',
  USER_EXPORT: 'user:export',
  
  // 订单管理
  ORDER_VIEW: 'order:view',
  ORDER_CREATE: 'order:create',
  ORDER_UPDATE: 'order:update',
  ORDER_DELETE: 'order:delete',
  ORDER_APPROVE: 'order:approve',
  
  // 系统管理
  SYSTEM_SETTING: 'system:setting',
  SYSTEM_LOG: 'system:log',
} as const;

type PermissionCode = typeof PERMISSIONS[keyof typeof PERMISSIONS];
```

---

## 四、权限Hook实现

```typescript
import { useMemo } from 'react';
import { useUserStore } from '@/stores/userStore';

// 权限Hook
export function usePermission() {
  const user = useUserStore(state => state.user);
  const permissions = useUserStore(state => state.permissions);

  // 检查单个权限
  const hasPermission = useMemo(() => {
    return (code: string): boolean => {
      if (!user) return false;
      return permissions.includes(code);
    };
  }, [user, permissions]);

  // 检查多个权限（任一）
  const hasAnyPermission = useMemo(() => {
    return (codes: string[]): boolean => {
      if (!user) return false;
      return codes.some(code => permissions.includes(code));
    };
  }, [user, permissions]);

  // 检查多个权限（全部）
  const hasAllPermissions = useMemo(() => {
    return (codes: string[]): boolean => {
      if (!user) return false;
      return codes.every(code => permissions.includes(code));
    };
  }, [user, permissions]);

  // 检查角色
  const hasRole = useMemo(() => {
    return (roleCode: string): boolean => {
      if (!user) return false;
      return user.roles.includes(roleCode);
    };
  }, [user]);

  return {
    hasPermission,
    hasAnyPermission,
    hasAllPermissions,
    hasRole,
    isAuthenticated: !!user,
  };
}

// 使用示例
function UserManagement() {
  const { hasPermission } = usePermission();

  return (
    <div>
      <h1>用户管理</h1>
      {hasPermission('user:create') && (
        <button>新增用户</button>
      )}
      {hasPermission('user:export') && (
        <button>导出数据</button>
      )}
    </div>
  );
}
```

---

## 五、权限组件

### 5.1 权限包装组件

```typescript
interface PermissionGuardProps {
  children: React.ReactNode;
  permission?: string;
  permissions?: string[];
  matchMode?: 'any' | 'all';
  fallback?: React.ReactNode;
}

export function PermissionGuard({
  children,
  permission,
  permissions,
  matchMode = 'any',
  fallback = null,
}: PermissionGuardProps) {
  const { hasPermission, hasAnyPermission, hasAllPermissions } = usePermission();

  const hasAccess = useMemo(() => {
    if (permission) {
      return hasPermission(permission);
    }
    if (permissions) {
      return matchMode === 'any'
        ? hasAnyPermission(permissions)
        : hasAllPermissions(permissions);
    }
    return true;
  }, [permission, permissions, matchMode, hasPermission, hasAnyPermission, hasAllPermissions]);

  if (!hasAccess) {
    return <>{fallback}</>;
  }

  return <>{children}</>;
}

// 使用示例
function UserList() {
  return (
    <div>
      <PermissionGuard permission="user:create">
        <Button type="primary">新增用户</Button>
      </PermissionGuard>
      
      <PermissionGuard permissions={['user:update', 'user:delete']} matchMode="any">
        <Table columns={actionColumns} />
      </PermissionGuard>
      
      {/* 无权限时显示提示 */}
      <PermissionGuard
        permission="user:export"
        fallback={<span>无导出权限</span>}
      >
        <Button>导出</Button>
      </PermissionGuard>
    </div>
  );
}
```

### 5.2 权限指令（高阶组件）

```typescript
// 按钮权限高阶组件
interface WithPermissionOptions {
  permission?: string;
  permissions?: string[];
  matchMode?: 'any' | 'all';
}

export function withPermission<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  options: WithPermissionOptions
) {
  return function WithPermissionComponent(props: P) {
    const { hasPermission, hasAnyPermission, hasAllPermissions } = usePermission();

    const hasAccess = useMemo(() => {
      if (options.permission) {
        return hasPermission(options.permission);
      }
      if (options.permissions) {
        return options.matchMode === 'any'
          ? hasAnyPermission(options.permissions)
          : hasAllPermissions(options.permissions);
      }
      return true;
    }, [hasPermission, hasAnyPermission, hasAllPermissions]);

    if (!hasAccess) {
      return null;
    }

    return <WrappedComponent {...props} />;
  };
}

// 使用
const DeleteButton = withPermission(Button, {
  permission: 'user:delete',
});
```

---

## 六、动态菜单

```typescript
interface MenuItem {
  id: string;
  name: string;
  path?: string;
  icon?: string;
  children?: MenuItem[];
  permissions?: string[]; // 访问该菜单需要的权限
  hidden?: boolean;
}

// 过滤菜单
export function filterMenusByPermission(
  menus: MenuItem[],
  userPermissions: string[]
): MenuItem[] {
  return menus
    .filter(menu => {
      // 如果菜单没有配置权限，默认可见
      if (!menu.permissions || menu.permissions.length === 0) {
        return !menu.hidden;
      }
      // 检查是否有任一权限
      return menu.permissions.some(p => userPermissions.includes(p));
    })
    .map(menu => ({
      ...menu,
      children: menu.children
        ? filterMenusByPermission(menu.children, userPermissions)
        : undefined,
    }));
}

// 获取第一个有权限的路由
export function getFirstAccessibleRoute(
  menus: MenuItem[],
  userPermissions: string[]
): string | null {
  for (const menu of menus) {
    if (menu.path && hasMenuPermission(menu, userPermissions)) {
      return menu.path;
    }
    if (menu.children) {
      const childRoute = getFirstAccessibleRoute(menu.children, userPermissions);
      if (childRoute) return childRoute;
    }
  }
  return null;
}

function hasMenuPermission(menu: MenuItem, userPermissions: string[]): boolean {
  if (!menu.permissions || menu.permissions.length === 0) return true;
  return menu.permissions.some(p => userPermissions.includes(p));
}
```

---

## 七、数据权限

```typescript
// 数据权限过滤Hook
export function useDataPermission<T extends Record<string, any>>(
  data: T[],
  fieldPermissions: Record<string, string>
): {
  filteredData: Partial<T>[];
  visibleFields: string[];
} {
  const { hasPermission } = usePermission();

  return useMemo(() => {
    // 确定可见字段
    const visibleFields = Object.entries(fieldPermissions)
      .filter(([_, permission]) => hasPermission(permission))
      .map(([field]) => field);

    // 过滤数据
    const filteredData = data.map(item => {
      const filtered: Partial<T> = {};
      visibleFields.forEach(field => {
        filtered[field as keyof T] = item[field];
      });
      return filtered;
    });

    return { filteredData, visibleFields };
  }, [data, fieldPermissions, hasPermission]);
}

// 使用示例
function UserTable({ users }: { users: User[] }) {
  const fieldPermissions = {
    name: 'user:view:name',
    email: 'user:view:email',
    phone: 'user:view:phone',
    salary: 'user:view:salary',
  };

  const { filteredData, visibleFields } = useDataPermission(users, fieldPermissions);

  const columns = useMemo(() => {
    return [
      visibleFields.includes('name') && { title: '姓名', dataIndex: 'name' },
      visibleFields.includes('email') && { title: '邮箱', dataIndex: 'email' },
      visibleFields.includes('phone') && { title: '电话', dataIndex: 'phone' },
      visibleFields.includes('salary') && { title: '薪资', dataIndex: 'salary' },
    ].filter(Boolean);
  }, [visibleFields]);

  return <Table columns={columns} dataSource={filteredData} />;
}
```

---

## 八、权限刷新

```typescript
// 权限刷新Hook
export function usePermissionRefresh() {
  const queryClient = useQueryClient();
  const setPermissions = useUserStore(state => state.setPermissions);

  const refreshPermissions = useCallback(async () => {
    try {
      // 重新获取用户权限
      const { permissions } = await fetchUserPermissions();
      setPermissions(permissions);
      
      // 刷新相关查询
      queryClient.invalidateQueries({ queryKey: ['user'] });
      
      return true;
    } catch (error) {
      console.error('刷新权限失败:', error);
      return false;
    }
  }, [queryClient, setPermissions]);

  return { refreshPermissions };
}
```

---

## 九、决策检查清单

- [ ] 权限码设计规范统一
- [ ] 实现了路由权限控制
- [ ] 实现了按钮权限控制
- [ ] 实现了数据权限过滤
- [ ] 动态菜单根据权限生成
- [ ] 权限刷新机制完善
- [ ] 前后端权限校验一致
- [ ] 无权限时UI处理友好

---

## 十、相关技能

- [路由与导航管理](./routing-and-navigation/SKILL.md) —— 路由权限控制
- [状态管理实现](./state-management-implementation/SKILL.md) —— 权限状态管理
- [React组件设计模式](./react-component-design-patterns/SKILL.md) —— 权限组件设计

---

## 十一、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含RBAC权限模型、权限组件、动态菜单 |
