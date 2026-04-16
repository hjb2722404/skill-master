---
name: internationalization-i18n
description: >
  本技能用于实现国际化(i18n)，包括语言切换、日期/数字格式化、RTL布局等。
  当需要实现多语言支持、格式化日期数字、处理RTL布局时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 国际化(i18n)实现

## 一、概述

### 1.1 这是什么

国际化(i18n)实现技能提供完整的多语言解决方案，涵盖语言切换、日期/数字格式化、RTL布局等。

### 1.2 适用场景

- ✅ 多语言网站/应用
- ✅ 日期/时间本地化
- ✅ 数字/货币格式化
- ✅ RTL（从右到左）语言支持
- ✅ 时区处理
- ❌ 单一语言应用
- ❌ 纯静态内容

### 1.3 核心原则

1. **文本外置** —— 所有用户可见文本放入语言包
2. **动态加载** —— 按需加载语言资源
3. **自动检测** —— 自动检测用户语言偏好
4. **持久化** —— 记住用户语言选择

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要支持多语言
- 需要本地化日期/数字
- 需要RTL布局支持
- 需要时区处理
- 需要货币格式化

### 2.2 输入

- 支持语言列表
- 翻译资源文件
- 默认语言设置
- 格式化需求

### 2.3 输出

- i18n配置
- 翻译Hook/组件
- 格式化工具
- 语言切换组件

---

## 三、i18n配置

### 3.1 使用react-i18next

```typescript
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';
import LanguageDetector from 'i18next-browser-languagedetector';

// 动态导入语言包
const loadResources = (lng: string) => {
  switch (lng) {
    case 'zh':
      return import('./locales/zh.json');
    case 'en':
      return import('./locales/en.json');
    case 'ja':
      return import('./locales/ja.json');
    default:
      return import('./locales/en.json');
  }
};

i18n
  .use(LanguageDetector)
  .use(initReactI18next)
  .init({
    fallbackLng: 'en',
    debug: process.env.NODE_ENV === 'development',
    
    interpolation: {
      escapeValue: false, // React already escapes
    },
    
    detection: {
      order: ['localStorage', 'navigator', 'htmlTag'],
      caches: ['localStorage'],
    },
  });

export default i18n;
```

### 3.2 语言包结构

```json
// locales/zh.json
{
  "common": {
    "welcome": "欢迎",
    "loading": "加载中...",
    "save": "保存",
    "cancel": "取消",
    "delete": "删除",
    "confirm": "确认"
  },
  "nav": {
    "home": "首页",
    "about": "关于",
    "settings": "设置"
  },
  "user": {
    "greeting": "你好，{{name}}",
    "profile": "个人资料",
    "logout": "退出登录"
  },
  "validation": {
    "required": "{{field}}不能为空",
    "email": "请输入有效的邮箱地址",
    "minLength": "{{field}}至少需要{{count}}个字符"
  }
}
```

---

## 四、翻译Hook

```typescript
import { useTranslation } from 'react-i18next';

// 基础使用
function WelcomeMessage({ name }: { name: string }) {
  const { t } = useTranslation();
  
  return <h1>{t('user.greeting', { name })}</h1>;
}

// 命名空间
function Navigation() {
  const { t } = useTranslation('nav');
  
  return (
    <nav>
      <a href="/">{t('home')}</a>
      <a href="/about">{t('about')}</a>
      <a href="/settings">{t('settings')}</a>
    </nav>
  );
}

// 切换语言
function LanguageSwitcher() {
  const { i18n } = useTranslation();
  
  const changeLanguage = (lng: string) => {
    i18n.changeLanguage(lng);
    document.documentElement.dir = lng === 'ar' ? 'rtl' : 'ltr';
  };

  return (
    <select
      value={i18n.language}
      onChange={(e) => changeLanguage(e.target.value)}
    >
      <option value="zh">中文</option>
      <option value="en">English</option>
      <option value="ja">日本語</option>
      <option value="ar">العربية</option>
    </select>
  );
}
```

---

## 五、格式化

### 5.1 日期时间格式化

```typescript
import { useTranslation } from 'react-i18next';

function DateDisplay({ date }: { date: Date }) {
  const { i18n } = useTranslation();
  
  const formattedDate = new Intl.DateTimeFormat(i18n.language, {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  }).format(date);

  const formattedTime = new Intl.DateTimeFormat(i18n.language, {
    hour: '2-digit',
    minute: '2-digit',
  }).format(date);

  return (
    <div>
      <p>{formattedDate}</p>
      <p>{formattedTime}</p>
    </div>
  );
}

// 相对时间
function RelativeTime({ date }: { date: Date }) {
  const { i18n } = useTranslation();
  
  const rtf = new Intl.RelativeTimeFormat(i18n.language, { numeric: 'auto' });
  
  const diff = Math.floor((date.getTime() - Date.now()) / (1000 * 60));
  
  return <span>{rtf.format(diff, 'minute')}</span>;
}
```

### 5.2 数字和货币格式化

```typescript
function NumberDisplay({ value }: { value: number }) {
  const { i18n } = useTranslation();
  
  const formatted = new Intl.NumberFormat(i18n.language, {
    minimumFractionDigits: 2,
    maximumFractionDigits: 2,
  }).format(value);

  return <span>{formatted}</span>;
}

function CurrencyDisplay({ amount, currency }: { amount: number; currency: string }) {
  const { i18n } = useTranslation();
  
  const formatted = new Intl.NumberFormat(i18n.language, {
    style: 'currency',
    currency,
  }).format(amount);

  return <span>{formatted}</span>;
}
```

---

## 六、RTL支持

```typescript
import { useEffect } from 'react';
import { useTranslation } from 'react-i18next';

const RTL_LANGUAGES = ['ar', 'he', 'fa', 'ur'];

export function useRTL() {
  const { i18n } = useTranslation();
  
  const isRTL = RTL_LANGUAGES.includes(i18n.language);
  
  useEffect(() => {
    document.documentElement.dir = isRTL ? 'rtl' : 'ltr';
    document.documentElement.lang = i18n.language;
  }, [i18n.language, isRTL]);

  return { isRTL };
}

// CSS处理
// styles.css
[dir="rtl"] {
  .sidebar {
    left: auto;
    right: 0;
  }
  
  .nav-item {
    text-align: right;
  }
  
  .icon-left {
    margin-left: 0;
    margin-right: 8px;
  }
}
```

---

## 七、决策检查清单

- [ ] 所有用户可见文本已提取
- [ ] 语言包结构清晰
- [ ] 支持动态语言切换
- [ ] 语言偏好已持久化
- [ ] 日期/数字格式化正确
- [ ] RTL布局支持完善
- [ ] 语言回退机制完善
- [ ] 翻译加载有loading状态

---

## 八、相关技能

- [React组件设计模式](./react-component-design-patterns/SKILL.md) —— 国际化组件设计
- [表单处理](./react-form-handling/SKILL.md) —— 表单验证国际化
- [状态管理实现](./state-management-implementation/SKILL.md) —— 语言状态管理

---

## 九、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含i18n配置、翻译Hook、格式化、RTL支持 |
