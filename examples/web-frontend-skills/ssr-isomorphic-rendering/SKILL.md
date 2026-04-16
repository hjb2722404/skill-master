---
name: ssr-isomorphic-rendering
description: >
  本技能用于实现SSR/同构渲染，包括Next.js/Nuxt.js、hydration、数据预取、SEO优化等。
  当需要实现服务端渲染、优化首屏加载、提升SEO时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# SSR/同构渲染

## 一、概述

### 1.1 这是什么

SSR/同构渲染技能提供完整的服务端渲染解决方案，涵盖Next.js/Nuxt.js、hydration、数据预取、SEO优化等。

### 1.2 适用场景

- ✅ SEO优化需求
- ✅ 首屏加载优化
- ✅ 社交媒体分享
- ✅ 低性能设备优化
- ✅ 内容密集型网站
- ❌ 纯后台管理系统
- ❌ 强交互型应用

### 1.3 核心原则

1. **同构代码** —— 服务端和客户端共享代码
2. **数据同步** —— 服务端数据脱水，客户端注水
3. **hydration** —— 正确恢复交互性
4. **渐进增强** —— 无JS也能基本可用

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要SEO优化
- 首屏加载时间要求高
- 需要社交媒体分享
- 需要改善低性能设备体验
- 内容型网站

### 2.2 输入

- SEO需求分析
- 数据获取方式
- 框架选型
- 部署环境

### 2.3 输出

- SSR框架配置
- 数据预取逻辑
- hydration配置
- SEO优化方案

---

## 三、Next.js实现

### 3.1 页面数据获取

```typescript
// pages/index.tsx
import { GetServerSideProps, GetStaticProps } from 'next';

interface Props {
  posts: Post[];
}

// 服务端渲染 (SSR)
export const getServerSideProps: GetServerSideProps<Props> = async () => {
  const posts = await fetchPosts();
  
  return {
    props: { posts },
  };
};

// 静态生成 (SSG)
export const getStaticProps: GetStaticProps<Props> = async () => {
  const posts = await fetchPosts();
  
  return {
    props: { posts },
    revalidate: 60, // ISR: 60秒后重新生成
  };
};

export default function HomePage({ posts }: Props) {
  return (
    <div>
      <h1>博客文章</h1>
      {posts.map((post) => (
        <article key={post.id}>
          <h2>{post.title}</h2>
          <p>{post.excerpt}</p>
        </article>
      ))}
    </div>
  );
}
```

### 3.2 App Router数据获取

```typescript
// app/page.tsx
async function getPosts() {
  const res = await fetch('https://api.example.com/posts', {
    next: { revalidate: 60 },
  });
  return res.json();
}

export default async function HomePage() {
  const posts = await getPosts();

  return (
    <div>
      <h1>博客文章</h1>
      {posts.map((post: Post) => (
        <article key={post.id}>
          <h2>{post.title}</h2>
        </article>
      ))}
    </div>
  );
}
```

---

## 四、数据预取与hydration

### 4.1 React Query SSR

```typescript
// utils/queryClient.ts
import { QueryClient } from '@tanstack/react-query';

export function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000,
      },
    },
  });
}

// app/providers.tsx
'use client';

import { QueryClientProvider } from '@tanstack/react-query';
import { useState } from 'react';
import { makeQueryClient } from '@/utils/queryClient';

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => makeQueryClient());

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  );
}

// app/page.tsx
import { dehydrate, HydrationBoundary } from '@tanstack/react-query';
import { makeQueryClient } from '@/utils/queryClient';
import Posts from './Posts';

export default async function HomePage() {
  const queryClient = makeQueryClient();
  
  await queryClient.prefetchQuery({
    queryKey: ['posts'],
    queryFn: fetchPosts,
  });

  return (
    <HydrationBoundary state={dehydrate(queryClient)}>
      <Posts />
    </HydrationBoundary>
  );
}
```

### 4.2 避免hydration不匹配

```typescript
'use client';

import { useEffect, useState } from 'react';

// 只在客户端渲染的组件
function ClientOnly({ children }: { children: React.ReactNode }) {
  const [hasMounted, setHasMounted] = useState(false);

  useEffect(() => {
    setHasMounted(true);
  }, []);

  if (!hasMounted) return null;

  return <>{children}</>;
}

// 使用
function Timestamp() {
  return (
    <ClientOnly>
      <span>{new Date().toLocaleString()}</span>
    </ClientOnly>
  );
}

// 或者使用suppressHydrationWarning
function DateDisplay({ date }: { date: Date }) {
  return (
    <time suppressHydrationWarning>
      {date.toLocaleString()}
    </time>
  );
}
```

---

## 五、SEO优化

### 5.1 元数据配置

```typescript
// app/layout.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: {
    default: '我的博客',
    template: '%s | 我的博客',
  },
  description: '一个分享技术的博客',
  keywords: ['技术', '博客', '前端'],
  authors: [{ name: '作者' }],
  openGraph: {
    title: '我的博客',
    description: '一个分享技术的博客',
    type: 'website',
    images: ['/og-image.jpg'],
  },
  twitter: {
    card: 'summary_large_image',
    title: '我的博客',
    description: '一个分享技术的博客',
  },
};

// app/blog/[slug]/page.tsx
import type { Metadata } from 'next';

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const post = await getPost(params.slug);
  
  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      images: [post.coverImage],
    },
  };
}
```

### 5.2 结构化数据

```typescript
// components/StructuredData.tsx
export function ArticleStructuredData({ post }: { post: Post }) {
  const structuredData = {
    '@context': 'https://schema.org',
    '@type': 'Article',
    headline: post.title,
    description: post.excerpt,
    author: {
      '@type': 'Person',
      name: post.author.name,
    },
    datePublished: post.createdAt,
    dateModified: post.updatedAt,
    image: post.coverImage,
  };

  return (
    <script
      type="application/ld+json"
      dangerouslySetInnerHTML={{
        __html: JSON.stringify(structuredData),
      }}
    />
  );
}
```

---

## 六、性能优化

### 6.1 流式渲染

```typescript
// app/page.tsx
import { Suspense } from 'react';
import { PostSkeleton } from '@/components/PostSkeleton';

export default function HomePage() {
  return (
    <div>
      <h1>我的博客</h1>
      
      {/* 立即渲染 */}
      <Header />
      
      {/* 流式加载 */}
      <Suspense fallback={<PostSkeleton />}>
        <Posts />
      </Suspense>
      
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
    </div>
  );
}
```

### 6.2 选择性hydration

```typescript
'use client';

import { useInView } from 'react-intersection-observer';

function LazyHydrate({ children }: { children: React.ReactNode }) {
  const { ref, inView } = useInView({
    triggerOnce: true,
    rootMargin: '200px',
  });

  return (
    <div ref={ref}>
      {inView ? children : <Placeholder />}
    </div>
  );
}
```

---

## 七、决策检查清单

- [ ] SSR框架配置正确
- [ ] 数据预取逻辑完善
- [ ] hydration错误处理
- [ ] SEO元数据配置
- [ ] 结构化数据添加
- [ ] 流式渲染优化
- [ ] 客户端状态同步
- [ ] 错误边界处理

---

## 八、相关技能

- [性能优化实战](./performance-optimization/SKILL.md) —— 性能优化
- [API数据获取与缓存](./api-data-fetching-and-caching/SKILL.md) —— 数据获取
- [懒加载与代码分割](./lazy-loading-and-code-splitting/SKILL.md) —— 代码分割

---

## 九、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含Next.js SSR、数据预取、SEO优化 |
