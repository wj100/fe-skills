# 首屏性能优化实战手册

## 目录
- [1. 目标与诊断流程](#1-目标与诊断流程)
- [2. 关键渲染路径治理](#2-关键渲染路径治理)
- [3. Critical CSS 提取](#3-critical-css-提取)
- [4. PurgeCSS 清理无用样式](#4-purgecss-清理无用样式)
- [5. Resource Hints：preload/prefetch/preconnect](#5-resource-hintspreloadprefetchpreconnect)
- [6. 字体优化策略](#6-字体优化策略)
- [7. SSR/SSG：React/Vue/Angular](#7-ssrssgreactvueangular)
- [8. Above-the-fold 优化](#8-above-the-fold-优化)
- [9. Inline Critical JS 与第三方脚本策略](#9-inline-critical-js-与第三方脚本策略)
- [10. 验证与回归检查](#10-验证与回归检查)

## 1. 目标与诊断流程

执行顺序：

1. 跑 Lighthouse，重点查看 FCP、LCP、TBT。
2. 在 Performance 面板确认：
   - 首屏渲染阻塞资源（CSS/JS/字体）
   - LCP 元素类型（图片、文本、容器）
3. 在 Network 确认：
   - 是否存在高优先级但非关键资源
   - 是否缺失关键资源 preload
4. 先优化首屏关键路径，再优化非关键资源。

## 2. 关键渲染路径治理

执行动作：

1. 缩短 HTML 到首屏可见内容的依赖链。
2. 将非关键 JS 延后执行（`defer`/动态加载）。
3. 合理内联极小关键脚本，减少主包阻塞。
4. 合并重复请求，避免多次重定向。

HTML 模板示例：

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <link rel="preconnect" href="https://cdn.example.com" crossorigin />
  <link rel="preload" href="/assets/critical.css" as="style" />
  <link rel="stylesheet" href="/assets/critical.css" />
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.ts" defer></script>
</body>
</html>
```

## 3. Critical CSS 提取

### 3.1 使用 Critters（Webpack）

```bash
npm i -D critters-webpack-plugin
```

```js
// webpack.config.js
const Critters = require('critters-webpack-plugin');

module.exports = {
  plugins: [
    new Critters({
      preload: 'swap',
      pruneSource: true,
      reduceInlineStyles: true,
    }),
  ],
};
```

### 3.2 Vite 中通过 SSR 后处理注入关键样式

Vite 没有官方内置 Critters 工作流时，采用构建后处理：

1. 先生成静态 HTML。
2. 用 `critters` Node API 处理 HTML。

```bash
npm i -D critters
```

```js
// scripts/inline-critical-css.mjs
import { readFile, writeFile } from 'node:fs/promises';
import Critters from 'critters';

const html = await readFile('dist/index.html', 'utf8');
const critters = new Critters({ path: 'dist', preload: 'swap' });
const out = await critters.process(html);
await writeFile('dist/index.html', out, 'utf8');
```

## 4. PurgeCSS 清理无用样式

### 4.1 PostCSS 集成

```bash
npm i -D @fullhuman/postcss-purgecss
```

```js
// postcss.config.cjs
const purgecss = require('@fullhuman/postcss-purgecss')({
  content: ['./index.html', './src/**/*.{js,ts,jsx,tsx,vue,html}'],
  safelist: ['is-open', /^modal-/, /^ant-/, /^el-/],
});

module.exports = {
  plugins: [
    ...(process.env.NODE_ENV === 'production' ? [purgecss] : []),
  ],
};
```

操作要点：

- 对运行时动态 class 建立 `safelist`。
- 发布前对关键页面做 UI 回归检查。

## 5. Resource Hints：preload/prefetch/preconnect

```html
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
<link rel="dns-prefetch" href="//api.example.com" />
<link rel="preload" as="font" href="/fonts/Inter-Subset.woff2" type="font/woff2" crossorigin />
<link rel="prefetch" href="/assets/report-page.js" as="script" />
```

规则：

- `preload` 给当前导航必需资源。
- `prefetch` 给后续可能访问资源。
- `preconnect` 给高频第三方域名。
- 只保留高价值 hint，避免连接泛滥。

## 6. 字体优化策略

### 6.1 `font-display` 与预加载

```css
@font-face {
  font-family: 'InterSubset';
  src: url('/fonts/Inter-Subset.woff2') format('woff2');
  font-display: swap;
  font-weight: 400 700;
}
```

```html
<link rel="preload" href="/fonts/Inter-Subset.woff2" as="font" type="font/woff2" crossorigin />
```

### 6.2 字体子集化（subset）

```bash
pyftsubset "./fonts/SourceHanSansCN-Regular.otf" \
  --text-file=./scripts/chars.txt \
  --output-file="./public/fonts/SourceHanSansCN-Subset.woff2" \
  --flavor=woff2
```

字体治理要点：

- 只给首屏内容保留必要字符集。
- 长文或后台页按需加载扩展字体。

## 7. SSR/SSG：React/Vue/Angular

### 7.1 React（Next.js）

执行选择：

- 静态内容页：`SSG`
- 高频更新但可缓存页：`ISR`
- 个性化强依赖请求头：`SSR`

```tsx
// app/page.tsx
export const revalidate = 60; // ISR

export default async function Page() {
  const data = await fetch('https://api.example.com/home', { next: { revalidate: 60 } }).then(r => r.json());
  return <main>{data.title}</main>;
}
```

### 7.2 Vue（Nuxt）

```ts
// nuxt.config.ts
export default defineNuxtConfig({
  ssr: true,
  routeRules: {
    '/': { static: true },
    '/news/**': { isr: 120 },
  },
});
```

### 7.3 Angular Universal

```bash
ng add @nguniversal/express-engine
npm run build:ssr
npm run serve:ssr
```

首屏目标：优先返回可渲染 HTML，减少客户端首帧等待。

## 8. Above-the-fold 优化

执行策略：

1. 精简首屏 DOM 深度与节点数。
2. 首屏仅保留必要组件，次要模块滚动后加载。
3. 为首屏图片和容器预留尺寸，避免 CLS。
4. Skeleton 使用轻量 CSS，不引入重动画库。

示例：

```css
.hero {
  min-height: 56vh;
}

.skeleton {
  background: linear-gradient(90deg, #f2f4f7 25%, #e8ecf3 37%, #f2f4f7 63%);
  background-size: 400% 100%;
  animation: pulse 1.2s ease infinite;
}
```

## 9. Inline Critical JS 与第三方脚本策略

### 9.1 关键内联脚本

只内联极小脚本，例如主题切换，避免闪烁：

```html
<script>
  (() => {
    const t = localStorage.getItem('theme');
    if (t) document.documentElement.dataset.theme = t;
  })();
</script>
```

### 9.2 第三方脚本加载策略

```html
<script src="https://www.googletagmanager.com/gtag/js?id=G-XXXX" async></script>
<script defer src="https://cdn.example.com/ab-test.js"></script>
```

动态注入（用户同意后加载）：

```js
function loadThirdParty() {
  const s = document.createElement('script');
  s.src = 'https://cdn.example.com/chat-widget.js';
  s.async = true;
  document.body.appendChild(s);
}
```

执行原则：

- 分离业务关键与营销脚本。
- 对第三方脚本设定加载预算与超时机制。

## 10. 验证与回归检查

### 10.1 验证命令

```bash
npx lighthouse http://localhost:4173 --view
```

### 10.2 对比清单

1. FCP/LCP 是否下降。
2. render-blocking resources 是否减少。
3. 主线程阻塞时间是否降低。
4. 首屏样式与字体无闪烁异常。

### 10.3 回归防护

- 在 CI 中接入 Lighthouse CI。
- 设定阈值：LCP、TBT、CLS 不得劣化超过基线。
- 发布后监控真实用户数据（RUM）确认收益稳定。
