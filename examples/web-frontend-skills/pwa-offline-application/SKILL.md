---
name: pwa-offline-application
description: >
  本技能用于实现PWA离线应用，包括Service Worker、缓存策略、后台同步、推送通知等。
  当需要实现离线访问、设计缓存策略、实现推送通知时激活此技能。
metadata:
  parent_methodology: web-frontend-methodology
  version: "1.0.0"
---

# PWA离线应用

## 一、概述

### 1.1 这是什么

PWA离线应用技能提供完整的渐进式Web应用解决方案，涵盖Service Worker、缓存策略、后台同步、推送通知等。

### 1.2 适用场景

- ✅ 离线访问需求
- ✅ 弱网环境优化
- ✅ 推送通知
- ✅ 后台同步
- ✅ 添加到主屏幕
- ❌ 纯静态展示网站
- ❌ 强依赖实时数据的金融应用

### 1.3 核心原则

1. **离线优先** —— 核心功能离线可用
2. **渐进增强** —— 从基础功能逐步增强
3. **缓存策略** —— 合理的缓存更新机制
4. **用户体验** —— 清晰的在线/离线状态

---

## 二、何时使用

### 2.1 激活条件

**当且仅当以下情况时激活本技能：**

- 需要支持离线访问
- 需要优化弱网体验
- 需要推送通知
- 需要后台同步
- 需要添加到主屏幕

### 2.2 输入

- 离线功能需求
- 缓存资源清单
- 推送通知需求
- 后台同步需求

### 2.3 输出

- Service Worker配置
- 缓存策略实现
- 推送通知配置
- 离线页面

---

## 三、Service Worker

### 3.1 注册Service Worker

```typescript
// main.ts
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker
      .register('/sw.js')
      .then((registration) => {
        console.log('SW registered:', registration);
      })
      .catch((error) => {
        console.log('SW registration failed:', error);
      });
  });
}
```

### 3.2 Service Worker实现

```javascript
// sw.js
const CACHE_NAME = 'app-v1';
const STATIC_ASSETS = [
  '/',
  '/index.html',
  '/static/js/main.js',
  '/static/css/main.css',
  '/offline.html',
];

// 安装：缓存静态资源
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then((cache) => {
      return cache.addAll(STATIC_ASSETS);
    })
  );
  self.skipWaiting();
});

// 激活：清理旧缓存
self.addEventListener('activate', (event) => {
  event.waitUntil(
    caches.keys().then((cacheNames) => {
      return Promise.all(
        cacheNames
          .filter((name) => name !== CACHE_NAME)
          .map((name) => caches.delete(name))
      );
    })
  );
  self.clients.claim();
});

// 拦截请求
self.addEventListener('fetch', (event) => {
  const { request } = event;

  // API请求：网络优先
  if (request.url.includes('/api/')) {
    event.respondWith(networkFirst(request));
    return;
  }

  // 静态资源：缓存优先
  if (request.destination === 'image' || request.destination === 'script') {
    event.respondWith(cacheFirst(request));
    return;
  }

  // 默认：缓存优先
  event.respondWith(cacheFirst(request));
});
```

---

## 四、缓存策略

```javascript
// 缓存优先
async function cacheFirst(request) {
  const cached = await caches.match(request);
  if (cached) return cached;

  try {
    const response = await fetch(request);
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, response.clone());
    return response;
  } catch (error) {
    return caches.match('/offline.html');
  }
}

// 网络优先
async function networkFirst(request) {
  try {
    const networkResponse = await fetch(request);
    const cache = await caches.open(CACHE_NAME);
    cache.put(request, networkResponse.clone());
    return networkResponse;
  } catch (error) {
    const cached = await caches.match(request);
    if (cached) return cached;
    throw error;
  }
}

// 仅网络
async function networkOnly(request) {
  return fetch(request);
}

// 仅缓存
async function cacheOnly(request) {
  const cached = await caches.match(request);
  if (cached) return cached;
  throw new Error('Not found in cache');
}

// 过时重新验证 (Stale While Revalidate)
async function staleWhileRevalidate(request) {
  const cache = await caches.open(CACHE_NAME);
  const cached = await cache.match(request);

  const fetchPromise = fetch(request).then((response) => {
    cache.put(request, response.clone());
    return response;
  });

  return cached || fetchPromise;
}
```

---

## 五、后台同步

```javascript
// 注册后台同步
async function registerBackgroundSync(tag) {
  const registration = await navigator.serviceWorker.ready;
  
  if ('sync' in registration) {
    await registration.sync.register(tag);
  }
}

// 使用
async function submitForm(data) {
  try {
    await fetch('/api/submit', {
      method: 'POST',
      body: JSON.stringify(data),
    });
  } catch (error) {
    // 离线时保存到IndexedDB并注册后台同步
    await saveToIndexedDB(data);
    await registerBackgroundSync('submit-form');
    showNotification('将在网络恢复后自动提交');
  }
}

// Service Worker中处理
self.addEventListener('sync', (event) => {
  if (event.tag === 'submit-form') {
    event.waitUntil(syncFormSubmissions());
  }
});

async function syncFormSubmissions() {
  const submissions = await getFromIndexedDB();
  
  for (const data of submissions) {
    try {
      await fetch('/api/submit', {
        method: 'POST',
        body: JSON.stringify(data),
      });
      await removeFromIndexedDB(data.id);
    } catch (error) {
      console.error('Sync failed:', error);
    }
  }
}
```

---

## 六、推送通知

```javascript
// 请求通知权限
async function requestNotificationPermission() {
  const permission = await Notification.requestPermission();
  return permission === 'granted';
}

// 订阅推送
async function subscribeToPush() {
  const registration = await navigator.serviceWorker.ready;
  
  const subscription = await registration.pushManager.subscribe({
    userVisibleOnly: true,
    applicationServerKey: urlBase64ToUint8Array(VAPID_PUBLIC_KEY),
  });

  // 发送订阅信息到服务器
  await fetch('/api/subscribe', {
    method: 'POST',
    body: JSON.stringify(subscription),
  });
}

// Service Worker中接收推送
self.addEventListener('push', (event) => {
  const data = event.data.json();
  
  event.waitUntil(
    self.registration.showNotification(data.title, {
      body: data.body,
      icon: '/icon-192x192.png',
      badge: '/badge-72x72.png',
      data: data.url,
      actions: [
        { action: 'open', title: '打开' },
        { action: 'close', title: '关闭' },
      ],
    })
  );
});

// 点击通知
self.addEventListener('notificationclick', (event) => {
  event.notification.close();

  if (event.action === 'open' || !event.action) {
    event.waitUntil(
      clients.openWindow(event.notification.data)
    );
  }
});
```

---

## 七、Manifest配置

```json
// manifest.json
{
  "name": "My PWA App",
  "short_name": "MyApp",
  "description": "A progressive web app",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#000000",
  "orientation": "portrait",
  "scope": "/",
  "icons": [
    {
      "src": "/icon-72x72.png",
      "sizes": "72x72",
      "type": "image/png"
    },
    {
      "src": "/icon-192x192.png",
      "sizes": "192x192",
      "type": "image/png"
    },
    {
      "src": "/icon-512x512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

---

## 八、决策检查清单

- [ ] Service Worker已注册
- [ ] 缓存策略配置合理
- [ ] 离线页面已实现
- [ ] 后台同步功能完善
- [ ] 推送通知已配置
- [ ] Manifest文件正确
- [ ] 图标尺寸完整
- [ ] 在线/离线状态提示

---

## 九、相关技能

- [性能优化实战](./performance-optimization/SKILL.md) —— 性能优化
- [API数据获取与缓存](./api-data-fetching-and-caching/SKILL.md) —— 数据缓存

---

## 十、版本历史

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| 1.0.0 | 2026-04-09 | 初始版本，包含Service Worker、缓存策略、后台同步、推送通知 |
