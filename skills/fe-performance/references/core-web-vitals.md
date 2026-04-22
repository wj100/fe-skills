# Core Web Vitals 治理实战手册

## 目录
- [1. 治理总流程](#1-治理总流程)
- [2. 指标阈值与优先级](#2-指标阈值与优先级)
- [3. LCP 优化：定位与修复](#3-lcp-优化定位与修复)
- [4. INP 优化：长任务与主线程治理](#4-inp-优化长任务与主线程治理)
- [5. CLS 优化：稳定布局策略](#5-cls-优化稳定布局策略)
- [6. 测量工具与诊断技巧](#6-测量工具与诊断技巧)
- [7. web-vitals 实时监控接入](#7-web-vitals-实时监控接入)
- [8. React/Vue/Angular 优化模式](#8-reactvueangular-优化模式)
- [9. 验证、告警与回归防护](#9-验证告警与回归防护)

## 1. 治理总流程

按“症状 → 证据 → 修复 → 验证”执行：

1. 收集实验室数据：Lighthouse + DevTools。
2. 收集真实用户数据：`web-vitals` 上报。
3. 把问题映射到 LCP/INP/CLS。
4. 每轮只修一个主因，避免互相干扰。
5. 修复后在低端机 + 弱网复测。

## 2. 指标阈值与优先级

建议阈值：

- LCP：`<= 2.5s`（75 分位）
- INP：`<= 200ms`（75 分位）
- CLS：`<= 0.1`（75 分位）

优先级顺序：

1. 先修 LCP（用户最直观感知）。
2. 再修 INP（交互体验）。
3. 最后修 CLS（视觉稳定）。

## 3. LCP 优化：定位与修复

### 3.1 定位 LCP 元素

在 DevTools Performance 录制页面加载，查看 `LCP` 标记，确认是：

- hero 图片
- 大标题文本块
- 首屏容器背景图

### 3.2 常见根因

1. LCP 资源过大（图片未压缩）。
2. 资源优先级不对（未 preload，或被低价值资源抢占）。
3. 首屏被 render-blocking CSS/JS 阻塞。
4. 服务器响应慢（TTFB 高）。

### 3.3 修复动作

#### 图片型 LCP

```html
<link rel="preload" as="image" href="/images/hero.avif" />
<img src="/images/hero.avif" width="1200" height="630" fetchpriority="high" alt="主视觉" />
```

#### 文本型 LCP

- 预加载首屏字体。
- 使用 `font-display: swap`。
- 内联关键 CSS，减少首屏样式阻塞。

#### 后端/边缘加速

- 开启 CDN 边缘缓存。
- 缓存 HTML（可缓存页面）。
- 启用压缩与 HTTP/2/HTTP/3。

## 4. INP 优化：长任务与主线程治理

### 4.1 识别长任务

在 Performance 面板检查 `Long Task`（>50ms），定位：

- 大量同步计算
- 重组件首次渲染
- 第三方脚本执行

### 4.2 拆分任务

```js
async function processItems(items) {
  for (let i = 0; i < items.length; i++) {
    heavyCompute(items[i]);
    if (i % 20 === 0) {
      await scheduler.yield();
    }
  }
}
```

兼容性回退：

```js
function idle(task) {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(task, { timeout: 300 });
  } else {
    setTimeout(task, 16);
  }
}
```

### 4.3 主线程减负

1. 将重计算迁移到 Web Worker。
2. 路由级与组件级懒加载，减少初始执行量。
3. 延迟非关键第三方脚本。
4. 避免事件处理函数内同步大循环。

### 4.4 交互代码优化示例

```js
button.addEventListener('click', () => {
  requestAnimationFrame(() => {
    panel.classList.add('open');
  });
  idle(() => prefetchNextData());
});
```

## 5. CLS 优化：稳定布局策略

### 5.1 图片/视频尺寸预留

```html
<img src="/images/product.webp" width="400" height="300" alt="商品" loading="lazy" />
```

```css
.video-shell {
  aspect-ratio: 16 / 9;
  background: #f3f5f8;
}
```

### 5.2 字体导致 CLS 的治理

```css
@font-face {
  font-family: 'InterSubset';
  src: url('/fonts/inter-subset.woff2') format('woff2');
  font-display: swap;
}
```

必要时设置字体度量调整（`size-adjust`）减少替换跳动。

### 5.3 动态内容插入模式

不要在已有内容上方直接插入通知条。预留槽位：

```css
.top-banner-slot {
  min-height: 48px;
}
```

```js
const slot = document.querySelector('.top-banner-slot');
slot.textContent = '新活动上线';
```

## 6. 测量工具与诊断技巧

### 6.1 Lighthouse

```bash
npx lighthouse http://localhost:4173 --view
```

### 6.2 DevTools

重点查看：

- Performance：LCP、Long Task、主线程火焰图
- Network：关键资源优先级、队头阻塞
- Rendering：Layout Shift Regions

### 6.3 web-vitals

```bash
npm i web-vitals
```

## 7. web-vitals 实时监控接入

```ts
// src/perf.ts
import { onCLS, onINP, onLCP } from 'web-vitals';

function report(metric: { name: string; value: number; id: string }) {
  navigator.sendBeacon(
    '/analytics/vitals',
    JSON.stringify({
      name: metric.name,
      value: metric.value,
      id: metric.id,
      path: location.pathname,
      ua: navigator.userAgent,
      ts: Date.now(),
    })
  );
}

onLCP(report);
onINP(report);
onCLS(report);
```

后端入库后按页面、设备、网络类型分组看 75 分位。

## 8. React/Vue/Angular 优化模式

### 8.1 React

- 使用 `React.lazy` + `Suspense` 拆分重组件。
- 避免父组件频繁重渲染导致交互延迟。
- 对长列表使用虚拟滚动（`react-window`）。

### 8.2 Vue

- 使用 `defineAsyncComponent` 拆分低频组件。
- 对响应式数据做颗粒度控制，避免大对象深层响应追踪。
- 列表场景配合虚拟列表。

### 8.3 Angular

- 使用 `loadChildren`/`loadComponent`。
- 启用 `OnPush` 变更检测策略。
- 将重计算移到 RxJS 流或 Web Worker。

## 9. 验证、告警与回归防护

### 9.1 验证步骤

1. 对比优化前后 Lighthouse。
2. 对比真实用户 75 分位变化。
3. 在低端 Android 设备复测交互。

### 9.2 告警建议

- LCP 75 分位 > 2.5s 告警。
- INP 75 分位 > 200ms 告警。
- CLS 75 分位 > 0.1 告警。

### 9.3 回归防护

1. 在 CI 中引入 Lighthouse CI。
2. 对大 PR 强制运行 bundle 分析。
3. 将 vitals 指标纳入发布门禁。
