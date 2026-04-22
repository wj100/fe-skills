# Service Worker 缓存实战手册

## 目录
- [1. 目标与落地流程](#1-目标与落地流程)
- [2. 缓存策略基线：CacheFirst/NetworkFirst/SWR](#2-缓存策略基线cachefirstnetworkfirstswr)
- [3. Vite 集成：vite-plugin-pwa](#3-vite-集成vite-plugin-pwa)
- [4. Webpack 集成：workbox-webpack-plugin](#4-webpack-集成workbox-webpack-plugin)
- [5. Angular 集成：@angular/service-worker](#5-angular-集成angularservice-worker)
- [6. 路由与 API 缓存规则设计](#6-路由与-api-缓存规则设计)
- [7. 版本更新与缓存失效机制](#7-版本更新与缓存失效机制)
- [8. 离线回退页与降级体验](#8-离线回退页与降级体验)
- [9. 调试、排错与注销](#9-调试排错与注销)
- [10. 验证收益与风险控制](#10-验证收益与风险控制)

## 1. 目标与落地流程

按以下顺序推进：

1. 识别资源类型：HTML、静态资源、API、媒体资源。
2. 为每类资源匹配缓存策略。
3. 在框架中接入 Workbox 或官方 SW 方案。
4. 设计版本化更新与回滚策略。
5. 用 DevTools Application 面板验证缓存命中。

## 2. 缓存策略基线：CacheFirst/NetworkFirst/SWR

### 2.1 CacheFirst

适用：哈希命名静态资源（`*.js`、`*.css`、字体、logo）。

优点：命中后极快。风险：更新不及时。

### 2.2 NetworkFirst

适用：HTML 文档、强时效 API。

优点：尽量新鲜。风险：弱网时慢；需设置超时与回退。

### 2.3 StaleWhileRevalidate

适用：列表 API、头像、非关键数据。

优点：响应快且后台更新。风险：可能短暂看到旧数据。

## 3. Vite 集成：vite-plugin-pwa

### 3.1 安装

```bash
npm i -D vite-plugin-pwa workbox-window
```

### 3.2 配置示例

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import { VitePWA } from 'vite-plugin-pwa';

export default defineConfig({
  plugins: [
    VitePWA({
      registerType: 'prompt',
      includeAssets: ['offline.html', 'favicon.svg'],
      manifest: {
        name: 'FE Performance App',
        short_name: 'FEPerf',
        display: 'standalone',
        background_color: '#ffffff',
        theme_color: '#1677ff',
      },
      workbox: {
        globPatterns: ['**/*.{js,css,html,svg,png,webp,avif,woff2}'],
        navigateFallback: '/offline.html',
        runtimeCaching: [
          {
            urlPattern: ({ request }) => request.destination === 'script' || request.destination === 'style',
            handler: 'CacheFirst',
            options: {
              cacheName: 'static-assets-v1',
              expiration: { maxEntries: 120, maxAgeSeconds: 60 * 60 * 24 * 30 },
            },
          },
          {
            urlPattern: ({ url }) => url.pathname.startsWith('/api/'),
            handler: 'NetworkFirst',
            options: {
              cacheName: 'api-cache-v1',
              networkTimeoutSeconds: 3,
              expiration: { maxEntries: 100, maxAgeSeconds: 60 * 10 },
            },
          },
          {
            urlPattern: ({ request }) => request.destination === 'image',
            handler: 'StaleWhileRevalidate',
            options: {
              cacheName: 'image-cache-v1',
              expiration: { maxEntries: 300, maxAgeSeconds: 60 * 60 * 24 * 14 },
            },
          },
        ],
      },
    }),
  ],
});
```

### 3.3 前端注册更新提示

```ts
import { registerSW } from 'virtual:pwa-register';

const updateSW = registerSW({
  onNeedRefresh() {
    const ok = window.confirm('发现新版本，是否立即刷新？');
    if (ok) updateSW(true);
  },
  onOfflineReady() {
    console.log('离线缓存已就绪');
  },
});
```

## 4. Webpack 集成：workbox-webpack-plugin

### 4.1 安装

```bash
npm i -D workbox-webpack-plugin
```

### 4.2 `GenerateSW` 配置

```js
// webpack.config.js
const WorkboxPlugin = require('workbox-webpack-plugin');

module.exports = {
  plugins: [
    new WorkboxPlugin.GenerateSW({
      clientsClaim: true,
      skipWaiting: false,
      cleanupOutdatedCaches: true,
      navigateFallback: '/offline.html',
      runtimeCaching: [
        {
          urlPattern: /\.(?:js|css)$/,
          handler: 'CacheFirst',
          options: {
            cacheName: 'static-assets-v1',
            expiration: { maxEntries: 100, maxAgeSeconds: 2592000 },
          },
        },
        {
          urlPattern: /\/api\//,
          handler: 'NetworkFirst',
          options: {
            cacheName: 'api-cache-v1',
            networkTimeoutSeconds: 3,
            expiration: { maxEntries: 80, maxAgeSeconds: 600 },
          },
        },
      ],
    }),
  ],
};
```

### 4.3 `InjectManifest` 自定义 `sw.js`

```js
// sw.js
import { precacheAndRoute } from 'workbox-precaching';
import { registerRoute, NavigationRoute } from 'workbox-routing';
import { CacheFirst, NetworkFirst, StaleWhileRevalidate } from 'workbox-strategies';

precacheAndRoute(self.__WB_MANIFEST);

registerRoute(
  ({ request }) => request.destination === 'image',
  new StaleWhileRevalidate({ cacheName: 'image-cache-v1' })
);

registerRoute(
  ({ url }) => url.pathname.startsWith('/api/'),
  new NetworkFirst({ cacheName: 'api-cache-v1', networkTimeoutSeconds: 3 })
);

const offlineHandler = async () => caches.match('/offline.html');
registerRoute(new NavigationRoute(offlineHandler));
```

## 5. Angular 集成：@angular/service-worker

### 5.1 启用

```bash
ng add @angular/pwa
```

### 5.2 `ngsw-config.json` 示例

```json
{
  "$schema": "./node_modules/@angular/service-worker/config/schema.json",
  "index": "/index.html",
  "assetGroups": [
    {
      "name": "app",
      "installMode": "prefetch",
      "resources": {
        "files": ["/favicon.ico", "/index.html", "/*.css", "/*.js"]
      }
    },
    {
      "name": "assets",
      "installMode": "lazy",
      "updateMode": "prefetch",
      "resources": {
        "files": ["/assets/**", "/*.(svg|png|jpg|webp|avif|woff2)"]
      }
    }
  ],
  "dataGroups": [
    {
      "name": "api-fresh",
      "urls": ["/api/orders/**"],
      "cacheConfig": {
        "strategy": "freshness",
        "maxSize": 60,
        "maxAge": "10m",
        "timeout": "3s"
      }
    },
    {
      "name": "api-fast",
      "urls": ["/api/profile/**"],
      "cacheConfig": {
        "strategy": "performance",
        "maxSize": 40,
        "maxAge": "1h"
      }
    }
  ]
}
```

## 6. 路由与 API 缓存规则设计

按业务价值拆分：

1. 静态资源：`CacheFirst` + 长缓存。
2. HTML 导航：`NetworkFirst` + fallback。
3. 实时 API（价格、库存、状态）：`NetworkFirst`。
4. 弱实时 API（推荐、公告）：`StaleWhileRevalidate`。
5. 个人敏感数据：谨慎缓存，必要时不缓存。

建议在响应头协同设置：

- `Cache-Control: public, max-age=31536000, immutable`（哈希静态资源）
- API 按业务设置短缓存或 no-store。

## 7. 版本更新与缓存失效机制

执行以下方案：

1. 给缓存名带版本：`static-assets-v2`。
2. 发布新版本时：
   - 预缓存新资源。
   - 通知用户刷新。
3. 激活新 SW 后清理旧缓存。

```js
self.addEventListener('activate', event => {
  const keep = ['static-assets-v2', 'api-cache-v2', 'image-cache-v2'];
  event.waitUntil(
    caches.keys().then(keys => Promise.all(keys.filter(k => !keep.includes(k)).map(k => caches.delete(k))))
  );
});
```

## 8. 离线回退页与降级体验

创建 `public/offline.html`：

```html
<!doctype html>
<html lang="zh-CN">
<head><meta charset="UTF-8" /><meta name="viewport" content="width=device-width,initial-scale=1" /><title>离线中</title></head>
<body>
  <h1>网络连接不可用</h1>
  <p>请检查网络后重试。</p>
  <button onclick="location.reload()">重新加载</button>
</body>
</html>
```

对关键业务页提供最小离线能力：最近浏览记录、本地草稿、失败重试队列。

## 9. 调试、排错与注销

### 9.1 调试

1. DevTools → Application → Service Workers。
2. 勾选 `Update on reload` 强制更新。
3. 查看 Cache Storage 内容是否符合预期。

### 9.2 注销（紧急回滚）

```js
async function unregisterAllSW() {
  const regs = await navigator.serviceWorker.getRegistrations();
  await Promise.all(regs.map(r => r.unregister()));
  const keys = await caches.keys();
  await Promise.all(keys.map(k => caches.delete(k)));
  location.reload();
}
```

### 9.3 常见问题

- 更新不生效：检查 `skipWaiting` 与用户刷新流程。
- API 旧数据：策略选错，改为 `NetworkFirst` 并缩短 `maxAge`。
- 离线页不触发：确认 `navigateFallback` 与路由匹配。

## 10. 验证收益与风险控制

### 10.1 验证指标

- 二次访问 TTFB 和 FCP 显著下降。
- 弱网下可访问离线回退页。
- API 缓存命中率可观且数据时效可接受。

### 10.2 验证命令

```bash
npx lighthouse http://localhost:4173 --view
```

### 10.3 风险控制

1. 不缓存敏感响应或带用户隐私的接口。
2. 对支付、库存等强一致接口禁用离线缓存。
3. 在灰度环境先验证更新流程，再全量发布。
