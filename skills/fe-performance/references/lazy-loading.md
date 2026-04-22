# 懒加载与代码分割实战手册

## 目录
- [1. 目标与诊断流程](#1-目标与诊断流程)
- [2. React 路由级与组件级懒加载](#2-react-路由级与组件级懒加载)
- [3. Vue 路由级与组件级懒加载](#3-vue-路由级与组件级懒加载)
- [4. Angular 路由级与组件级懒加载](#4-angular-路由级与组件级懒加载)
- [5. 原生 JS 动态 import() 模式](#5-原生-js-动态-import-模式)
- [6. Prefetch 与 Preload 策略](#6-prefetch-与-preload-策略)
- [7. Webpack/Vite 拆包配置](#7-webpackvite-拆包配置)
- [8. Magic Comments 最佳实践](#8-magic-comments-最佳实践)
- [9. 验证收益与回归防护](#9-验证收益与回归防护)

## 1. 目标与诊断流程

按以下流程执行：

1. 测基线。
   - 运行 `npm run build`。
   - 运行 `npx source-map-explorer "dist/assets/*.js"`（Vite）或 `npx source-map-explorer "build/static/js/*.js"`（CRA）。
   - 记录首屏 JS 传输体积、首屏请求数、LCP、INP。
2. 定位重模块。
   - 找出首屏不需要却被初始 chunk 打包的模块。
   - 常见问题：图表库、编辑器、国际化全量包、日期库、图标全集。
3. 先做路由级拆分，再做组件级拆分。
4. 对“高概率下一跳页面”添加 prefetch；对“当前渲染必需资源”使用 preload。
5. 对比改造前后分析报告，确认收益。

## 2. React 路由级与组件级懒加载

### 2.1 路由级 code splitting（React Router）

```tsx
import { Suspense, lazy } from 'react';
import { createBrowserRouter, RouterProvider } from 'react-router-dom';

const HomePage = lazy(() => import('./pages/HomePage'));
const ReportPage = lazy(() => import('./pages/ReportPage'));
const SettingsPage = lazy(() => import('./pages/SettingsPage'));

const router = createBrowserRouter([
  { path: '/', element: <HomePage /> },
  { path: '/report', element: <ReportPage /> },
  { path: '/settings', element: <SettingsPage /> },
]);

export default function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <RouterProvider router={router} />
    </Suspense>
  );
}
```

执行要点：

- 给 `fallback` 提供与页面骨架一致的占位 UI，避免 CLS。
- 不要把首屏关键路径组件也全部 lazy，避免引入额外 waterfall。

### 2.2 组件级懒加载（重组件按需）

```tsx
import { lazy, Suspense, useState } from 'react';

const HeavyChart = lazy(() => import('./components/HeavyChart'));

export function Dashboard() {
  const [showChart, setShowChart] = useState(false);

  return (
    <section>
      <button onClick={() => setShowChart(true)}>加载图表</button>
      {showChart ? (
        <Suspense fallback={<div>图表加载中...</div>}>
          <HeavyChart />
        </Suspense>
      ) : null}
    </section>
  );
}
```

## 3. Vue 路由级与组件级懒加载

### 3.1 Vue Router 异步路由组件

```ts
import { createRouter, createWebHistory } from 'vue-router';

const router = createRouter({
  history: createWebHistory(),
  routes: [
    { path: '/', component: () => import('@/pages/HomePage.vue') },
    { path: '/report', component: () => import('@/pages/ReportPage.vue') },
    { path: '/settings', component: () => import('@/pages/SettingsPage.vue') },
  ],
});

export default router;
```

### 3.2 `defineAsyncComponent` 组件级懒加载

```ts
import { defineAsyncComponent } from 'vue';

const AsyncEditor = defineAsyncComponent({
  loader: () => import('@/components/RichEditor.vue'),
  loadingComponent: () => import('@/components/LoadingBlock.vue'),
  delay: 120,
  timeout: 15000,
});

export default AsyncEditor;
```

### 3.3 Vue 3 Suspense 示例

```vue
<template>
  <Suspense>
    <template #default>
      <AsyncEditor />
    </template>
    <template #fallback>
      <div>编辑器加载中...</div>
    </template>
  </Suspense>
</template>
```

## 4. Angular 路由级与组件级懒加载

### 4.1 `loadChildren`（模块级）

```ts
import { Routes } from '@angular/router';

export const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.routes').then(m => m.ADMIN_ROUTES),
  },
  {
    path: 'report',
    loadChildren: () => import('./report/report.module').then(m => m.ReportModule),
  },
];
```

### 4.2 `loadComponent`（Standalone 组件）

```ts
export const routes: Routes = [
  {
    path: 'settings',
    loadComponent: () =>
      import('./settings/settings.component').then(c => c.SettingsComponent),
  },
];
```

### 4.3 模板中按需加载大组件

```html
<button (click)="showChart = true">显示图表</button>
<ng-container *ngIf="showChart">
  <app-heavy-chart />
</ng-container>
```

配合路由守卫和数据预取时，优先让数据请求与 chunk 下载并行。

## 5. 原生 JS 动态 import() 模式

### 5.1 事件触发加载

```js
document.getElementById('open-map').addEventListener('click', async () => {
  const { initMap } = await import('./map.js');
  initMap();
});
```

### 5.2 条件分支加载

```js
async function setupExperiment(flag) {
  if (flag === 'new-checkout') {
    const mod = await import('./checkout-new.js');
    mod.mount();
  } else {
    const mod = await import('./checkout-legacy.js');
    mod.mount();
  }
}
```

最佳实践：

- 保证动态路径可静态分析，避免 `import(someVar)` 导致拆包失控。
- 在用户动作之前预热：hover、idle、viewport 接近时触发预取。

## 6. Prefetch 与 Preload 策略

### 6.1 路由预取（hover prefetch）

```tsx
function PrefetchLink() {
  const onMouseEnter = () => {
    import('./pages/ReportPage');
  };

  return <a href="/report" onMouseEnter={onMouseEnter}>报表</a>;
}
```

### 6.2 HTML 资源提示

```html
<link rel="preload" href="/assets/hero-banner.avif" as="image" />
<link rel="prefetch" href="/assets/report-page.js" as="script" />
```

执行原则：

- `preload` 仅用于当前导航必需资源。
- `prefetch` 用于“高概率下一跳”，优先级低，避免抢占关键资源。

## 7. Webpack/Vite 拆包配置

### 7.1 Webpack `splitChunks`

```js
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      maxInitialRequests: 25,
      cacheGroups: {
        framework: {
          test: /[\\/]node_modules[\\/](react|react-dom|vue|@angular)[\\/]/,
          name: 'framework',
          priority: 40,
        },
        charting: {
          test: /[\\/]node_modules[\\/](echarts|chart\.js|d3)[\\/]/,
          name: 'charting',
          priority: 30,
        },
        vendors: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          priority: 10,
        },
      },
    },
  },
};
```

### 7.2 Vite `manualChunks`

```ts
// vite.config.ts
import { defineConfig } from 'vite';

export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks(id) {
          if (id.includes('node_modules')) {
            if (id.includes('react') || id.includes('vue') || id.includes('@angular')) return 'framework';
            if (id.includes('echarts') || id.includes('chart.js') || id.includes('d3')) return 'charting';
            return 'vendor';
          }
        },
      },
    },
  },
});
```

## 8. Magic Comments 最佳实践

```js
const ReportPage = () =>
  import(
    /* webpackChunkName: "report-page" */
    /* webpackPrefetch: true */
    './pages/ReportPage'
  );

const HeroModule = () =>
  import(
    /* webpackChunkName: "hero-module" */
    /* webpackPreload: true */
    './hero-module'
  );
```

规则：

- `webpackChunkName` 用于可读 chunk 命名，便于分析。
- `webpackPrefetch` 用于非当前必需资源。
- `webpackPreload` 用于当前渲染即将需要的资源，谨慎使用。

## 9. 验证收益与回归防护

### 9.1 对比分析命令

```bash
# Webpack 项目
npx webpack --mode production --profile --json > stats.json
npx webpack-bundle-analyzer stats.json

# Vite 项目
npm i -D rollup-plugin-visualizer
npm run build
```

### 9.2 验证清单

1. 首屏 JS 传输体积显著下降（目标：减少 20%~50%）。
2. 初始执行任务减少，INP/TBT 改善。
3. 路由切换首次打开对应页面可接受（避免过度拆分导致延迟感）。
4. chunk 数量与缓存命中平衡，避免“碎片化过度”。

### 9.3 常见反模式

- 过度拆包：产生大量微小 chunk，HTTP 开销反增。
- 错误 prefetch：低价值资源抢网络带宽。
- 未设置稳定 chunk 命名：回归难以追踪。
- 未做二次访问验证：只看首访，忽略缓存收益。
