---
name: search-and-filter-implementation
description: >
  本技能用于实现搜索与筛选功能，包括实时搜索、多条件筛选、搜索历史、自动补全等。
  当需要实现搜索功能、设计筛选界面、实现自动补全时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# 搜索与筛选实现

## 一、概述

### 1.1 这是什么

搜索与筛选实现技能提供完整的搜索功能解决方案，涵盖实时搜索、多条件筛选、搜索历史、自动补全等功能。

### 1.2 适用场景

- ✅ 实时搜索（输入即搜索）
- ✅ 多条件组合筛选
- ✅ 搜索历史记录
- ✅ 自动补全/联想
- ✅ 高级筛选面板
- ❌ 简单静态列表过滤
- ❌ 后端已提供完整搜索接口

### 1.3 核心原则

1. **防抖优化** —— 避免频繁请求
2. **状态同步** —— URL与筛选状态同步
3. **用户体验** —— 即时反馈和加载状态
4. **可访问性** —— 键盘导航和屏幕阅读器支持

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要实现搜索输入功能
- 需要多条件筛选界面
- 需要搜索历史功能
- 需要自动补全功能
- 需要筛选状态持久化

### 2.2 输入

- 搜索字段定义
- 筛选条件配置
- 数据源类型
- 是否需要持久化

### 2.3 输出

- 搜索组件实现
- 筛选面板组件
- 自动补全组件
- 状态管理逻辑

---

## 三、实时搜索

### 3.1 防抖搜索Hook

```typescript
import { useState, useCallback, useRef, useEffect } from 'react';

interface UseSearchOptions<T> {
  searchFn: (query: string) => Promise<T[]>;
  debounceMs?: number;
  minLength?: number;
}

export function useSearch<T>(options: UseSearchOptions<T>) {
  const { searchFn, debounceMs = 300, minLength = 2 } = options;
  
  const [query, setQuery] = useState('');
  const [results, setResults] = useState<T[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  
  const debounceRef = useRef<NodeJS.Timeout>();
  const abortControllerRef = useRef<AbortController>();

  const search = useCallback(async (searchQuery: string) => {
    // 清除之前的定时器
    if (debounceRef.current) {
      clearTimeout(debounceRef.current);
    }
    
    // 取消之前的请求
    abortControllerRef.current?.abort();
    
    setQuery(searchQuery);
    
    // 清空短查询
    if (searchQuery.length < minLength) {
      setResults([]);
      setError(null);
      return;
    }

    // 防抖执行
    debounceRef.current = setTimeout(async () => {
      setLoading(true);
      setError(null);
      
      abortControllerRef.current = new AbortController();
      
      try {
        const data = await searchFn(searchQuery);
        setResults(data);
      } catch (err) {
        if ((err as Error).name !== 'AbortError') {
          setError(err as Error);
        }
      } finally {
        setLoading(false);
      }
    }, debounceMs);
  }, [searchFn, debounceMs, minLength]);

  const clear = useCallback(() => {
    setQuery('');
    setResults([]);
    setError(null);
    debounceRef.current && clearTimeout(debounceRef.current);
    abortControllerRef.current?.abort();
  }, []);

  useEffect(() => {
    return () => {
      debounceRef.current && clearTimeout(debounceRef.current);
      abortControllerRef.current?.abort();
    };
  }, []);

  return { query, results, loading, error, search, clear };
}
```

### 3.2 搜索组件

```typescript
interface SearchInputProps {
  value: string;
  onChange: (value: string) => void;
  placeholder?: string;
  loading?: boolean;
  autoFocus?: boolean;
}

export function SearchInput({
  value,
  onChange,
  placeholder = '搜索...',
  loading,
  autoFocus,
}: SearchInputProps) {
  const inputRef = useRef<HTMLInputElement>(null);

  return (
    <div className="search-input-wrapper">
      <input
        ref={inputRef}
        type="text"
        value={value}
        onChange={(e) => onChange(e.target.value)}
        placeholder={placeholder}
        autoFocus={autoFocus}
        className="search-input"
      />
      {loading ? (
        <span className="search-loading">加载中...</span>
      ) : value ? (
        <button
          className="search-clear"
          onClick={() => {
            onChange('');
            inputRef.current?.focus();
          }}
        >
          ×
        </button>
      ) : (
        <span className="search-icon">🔍</span>
      )}
    </div>
  );
}
```

---

## 四、多条件筛选

### 4.1 筛选配置

```typescript
interface FilterConfig {
  key: string;
  label: string;
  type: 'select' | 'multiselect' | 'date' | 'dateRange' | 'number' | 'text';
  options?: { value: string; label: string }[];
  placeholder?: string;
}

interface FilterValue {
  [key: string]: any;
}

const filterConfigs: FilterConfig[] = [
  {
    key: 'status',
    label: '状态',
    type: 'select',
    options: [
      { value: 'active', label: '活跃' },
      { value: 'inactive', label: '禁用' },
    ],
  },
  {
    key: 'tags',
    label: '标签',
    type: 'multiselect',
    options: [
      { value: 'vip', label: 'VIP' },
      { value: 'new', label: '新用户' },
    ],
  },
  {
    key: 'createdAt',
    label: '创建时间',
    type: 'dateRange',
  },
];
```

### 4.2 筛选面板组件

```typescript
interface FilterPanelProps {
  configs: FilterConfig[];
  values: FilterValue;
  onChange: (values: FilterValue) => void;
}

export function FilterPanel({ configs, values, onChange }: FilterPanelProps) {
  const handleChange = (key: string, value: any) => {
    onChange({ ...values, [key]: value });
  };

  const handleClear = () => {
    onChange({});
  };

  const hasFilters = Object.keys(values).length > 0;

  return (
    <div className="filter-panel">
      <div className="filter-header">
        <h3>筛选条件</h3>
        {hasFilters && (
          <button onClick={handleClear} className="clear-btn">
            清除全部
          </button>
        )}
      </div>
      
      <div className="filter-body">
        {configs.map((config) => (
          <div key={config.key} className="filter-item">
            <label>{config.label}</label>
            <FilterControl
              config={config}
              value={values[config.key]}
              onChange={(value) => handleChange(config.key, value)}
            />
          </div>
        ))}
      </div>
    </div>
  );
}

function FilterControl({
  config,
  value,
  onChange,
}: {
  config: FilterConfig;
  value: any;
  onChange: (value: any) => void;
}) {
  switch (config.type) {
    case 'select':
      return (
        <select value={value || ''} onChange={(e) => onChange(e.target.value || undefined)}>
          <option value="">全部</option>
          {config.options?.map((opt) => (
            <option key={opt.value} value={opt.value}>
              {opt.label}
            </option>
          ))}
        </select>
      );
      
    case 'multiselect':
      return (
        <div className="multiselect">
          {config.options?.map((opt) => {
            const selected = (value || []).includes(opt.value);
            return (
              <label key={opt.value} className={selected ? 'selected' : ''}>
                <input
                  type="checkbox"
                  checked={selected}
                  onChange={(e) => {
                    const current = value || [];
                    if (e.target.checked) {
                      onChange([...current, opt.value]);
                    } else {
                      onChange(current.filter((v: string) => v !== opt.value));
                    }
                  }}
                />
                {opt.label}
              </label>
            );
          })}
        </div>
      );
      
    case 'dateRange':
      return (
        <div className="date-range">
          <input
            type="date"
            value={value?.start || ''}
            onChange={(e) => onChange({ ...value, start: e.target.value })}
          />
          <span>至</span>
          <input
            type="date"
            value={value?.end || ''}
            onChange={(e) => onChange({ ...value, end: e.target.value })}
          />
        </div>
      );
      
    default:
      return (
        <input
          type="text"
          value={value || ''}
          placeholder={config.placeholder}
          onChange={(e) => onChange(e.target.value || undefined)}
        />
      );
  }
}
```

---

## 五、搜索历史

```typescript
const HISTORY_KEY = 'search_history';
const MAX_HISTORY = 10;

export function useSearchHistory() {
  const [history, setHistory] = useState<string[]>([]);

  // 加载历史
  useEffect(() => {
    const saved = localStorage.getItem(HISTORY_KEY);
    if (saved) {
      setHistory(JSON.parse(saved));
    }
  }, []);

  // 添加历史
  const addHistory = useCallback((query: string) => {
    if (!query.trim()) return;
    
    setHistory((prev) => {
      const newHistory = [
        query,
        ...prev.filter((h) => h !== query),
      ].slice(0, MAX_HISTORY);
      
      localStorage.setItem(HISTORY_KEY, JSON.stringify(newHistory));
      return newHistory;
    });
  }, []);

  // 删除单条
  const removeHistory = useCallback((query: string) => {
    setHistory((prev) => {
      const newHistory = prev.filter((h) => h !== query);
      localStorage.setItem(HISTORY_KEY, JSON.stringify(newHistory));
      return newHistory;
    });
  }, []);

  // 清空历史
  const clearHistory = useCallback(() => {
    setHistory([]);
    localStorage.removeItem(HISTORY_KEY);
  }, []);

  return { history, addHistory, removeHistory, clearHistory };
}
```

---

## 六、自动补全

```typescript
interface UseAutocompleteOptions<T> {
  fetchSuggestions: (query: string) => Promise<T[]>;
  debounceMs?: number;
  minLength?: number;
}

export function useAutocomplete<T>(options: UseAutocompleteOptions<T>) {
  const { fetchSuggestions, debounceMs = 200, minLength = 1 } = options;
  
  const [query, setQuery] = useState('');
  const [suggestions, setSuggestions] = useState<T[]>([]);
  const [isOpen, setIsOpen] = useState(false);
  const [highlightedIndex, setHighlightedIndex] = useState(-1);
  const [loading, setLoading] = useState(false);

  const debounceRef = useRef<NodeJS.Timeout>();

  const updateSuggestions = useCallback(async (value: string) => {
    if (value.length < minLength) {
      setSuggestions([]);
      return;
    }

    setLoading(true);
    try {
      const data = await fetchSuggestions(value);
      setSuggestions(data);
      setIsOpen(true);
    } finally {
      setLoading(false);
    }
  }, [fetchSuggestions, minLength]);

  const handleInputChange = useCallback((value: string) => {
    setQuery(value);
    setHighlightedIndex(-1);
    
    clearTimeout(debounceRef.current);
    debounceRef.current = setTimeout(() => {
      updateSuggestions(value);
    }, debounceMs);
  }, [updateSuggestions, debounceMs]);

  const handleKeyDown = useCallback((e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault();
        setHighlightedIndex((prev) =>
          prev < suggestions.length - 1 ? prev + 1 : prev
        );
        break;
      case 'ArrowUp':
        e.preventDefault();
        setHighlightedIndex((prev) => (prev > 0 ? prev - 1 : -1));
        break;
      case 'Enter':
        if (highlightedIndex >= 0) {
          selectSuggestion(suggestions[highlightedIndex]);
        }
        break;
      case 'Escape':
        setIsOpen(false);
        break;
    }
  }, [suggestions, highlightedIndex]);

  const selectSuggestion = useCallback((suggestion: T) => {
    setQuery(getSuggestionLabel(suggestion));
    setIsOpen(false);
    setHighlightedIndex(-1);
  }, []);

  const getSuggestionLabel = (suggestion: T): string => {
    // 根据实际数据结构返回显示文本
    return String(suggestion);
  };

  return {
    query,
    suggestions,
    isOpen,
    highlightedIndex,
    loading,
    handleInputChange,
    handleKeyDown,
    selectSuggestion,
    setIsOpen,
  };
}
```

---

## 七、决策检查清单

- [ ] 搜索实现了防抖
- [ ] 筛选条件与URL同步
- [ ] 实现了搜索历史
- [ ] 自动补全支持键盘导航
- [ ] 加载状态显示清晰
- [ ] 空结果处理完善
- [ ] 筛选条件可清空
- [ ] 响应式设计

---

## 八、相关技能

- [API数据获取与缓存](./api-data-fetching-and-caching/SKILL.md) —— 搜索接口调用
- [React组件设计模式](./react-component-design-patterns/SKILL.md) —— 搜索组件设计
- [路由与导航管理](./routing-and-navigation/SKILL.md) —— URL状态同步

---

## 九、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含实时搜索、多条件筛选、搜索历史、自动补全 |
