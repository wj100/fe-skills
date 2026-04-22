# 图片优化实战手册

## 目录
- [1. 目标与诊断流程](#1-目标与诊断流程)
- [2. 格式转换：PNG/JPG 到 WebP/AVIF](#2-格式转换pngjpg-到-webpavif)
- [3. 构建期优化：Vite 与 Webpack](#3-构建期优化vite-与-webpack)
- [4. CDN 动态处理：OSS/七牛/COS](#4-cdn-动态处理oss七牛cos)
- [5. 运行时 `<picture>` 回退方案](#5-运行时-picture-回退方案)
- [6. 响应式图片：srcset 与 sizes](#6-响应式图片srcset-与-sizes)
- [7. 图片懒加载：loading 与 Intersection Observer](#7-图片懒加载loading-与-intersection-observer)
- [8. SVG 优化：SVGO](#8-svg-优化svgo)
- [9. 占位图策略：LQIP/BlurHash/主色块](#9-占位图策略lqipblurhash主色块)
- [10. 验证收益与质量兜底](#10-验证收益与质量兜底)

## 1. 目标与诊断流程

执行以下流程：

1. 在 Lighthouse 中检查 `Properly size images`、`Serve images in next-gen formats`。
2. 在 Network 面板按 `Img` 过滤，记录：
   - 首屏图片总传输体积
   - LCP 图片体积与下载时间
3. 定位问题类型：
   - 格式不佳（PNG/JPG 未转 WebP/AVIF）
   - 分辨率过大（未做响应式）
   - 加载时机错误（首屏外图片提前加载）
4. 先改 LCP 图片，再改长列表图片，再改非关键图。

## 2. 格式转换：PNG/JPG 到 WebP/AVIF

### 2.1 命令行批处理（Node 工具链）

```bash
npm i -D sharp
```

```js
// scripts/convert-images.mjs
import sharp from 'sharp';
import { readdir, mkdir } from 'node:fs/promises';
import path from 'node:path';

const inputDir = 'assets/raw';
const outputDir = 'assets/optimized';
await mkdir(outputDir, { recursive: true });

const files = await readdir(inputDir);
for (const file of files) {
  if (!/\.(png|jpg|jpeg)$/i.test(file)) continue;
  const input = path.join(inputDir, file);
  const base = file.replace(/\.(png|jpg|jpeg)$/i, '');

  await sharp(input).webp({ quality: 75 }).toFile(path.join(outputDir, `${base}.webp`));
  await sharp(input).avif({ quality: 45 }).toFile(path.join(outputDir, `${base}.avif`));
}
```

```bash
node scripts/convert-images.mjs
```

### 2.2 质量建议

- WebP：`quality 70~80`
- AVIF：`quality 35~50`
- 对 UI 截图、文字图慎用过低质量，先做肉眼对比。

## 3. 构建期优化：Vite 与 Webpack

### 3.1 Vite：`vite-plugin-imagemin`

```bash
npm i -D vite-plugin-imagemin imagemin-webp imagemin-avif
```

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import viteImagemin from 'vite-plugin-imagemin';

export default defineConfig({
  plugins: [
    viteImagemin({
      mozjpeg: { quality: 75 },
      pngquant: { quality: [0.7, 0.85], speed: 4 },
      webp: { quality: 75 },
      avif: { quality: 45 },
    }),
  ],
});
```

### 3.2 Webpack：`image-webpack-loader`

```bash
npm i -D image-webpack-loader file-loader
```

```js
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.(png|jpe?g|gif|webp|avif)$/i,
        use: [
          { loader: 'file-loader', options: { name: 'images/[name].[contenthash].[ext]' } },
          {
            loader: 'image-webpack-loader',
            options: {
              mozjpeg: { progressive: true, quality: 75 },
              optipng: { enabled: false },
              pngquant: { quality: [0.7, 0.85], speed: 3 },
              webp: { quality: 75 },
            },
          },
        ],
      },
    ],
  },
};
```

## 4. CDN 动态处理：OSS/七牛/COS

优先使用 CDN 实时裁剪与格式协商，减少前端构建复杂度。

### 4.1 阿里云 OSS

```txt
https://example.oss-cn-hangzhou.aliyuncs.com/banner.jpg?x-oss-process=image/resize,w_1200/format,webp/quality,Q_75
```

### 4.2 七牛云

```txt
https://cdn.example.com/banner.jpg?imageView2/2/w/1200/format/webp/q/75
```

### 4.3 腾讯云 COS（CI）

```txt
https://example.cos.ap-shanghai.myqcloud.com/banner.jpg?imageMogr2/thumbnail/1200x/format/webp/quality/75
```

执行要点：

- 统一封装图片 URL 生成函数，按 DPR、容器宽度动态拼参数。
- 对国内用户优先使用国内节点 CDN，降低跨境 RTT。

## 5. 运行时 `<picture>` 回退方案

```html
<picture>
  <source type="image/avif" srcset="/images/hero.avif" />
  <source type="image/webp" srcset="/images/hero.webp" />
  <img
    src="/images/hero.jpg"
    alt="活动主视觉"
    width="1200"
    height="630"
    decoding="async"
    fetchpriority="high"
  />
</picture>
```

规则：

- LCP 图片保留 `fetchpriority="high"`。
- 所有图片声明 `width/height`，减少 CLS。
- 非首屏图不要设置高优先级。

## 6. 响应式图片：srcset 与 sizes

```html
<img
  src="/images/card-640.webp"
  srcset="
    /images/card-320.webp 320w,
    /images/card-640.webp 640w,
    /images/card-960.webp 960w,
    /images/card-1280.webp 1280w
  "
  sizes="(max-width: 600px) 90vw, (max-width: 1200px) 45vw, 400px"
  alt="商品图"
  width="400"
  height="300"
/>
```

执行步骤：

1. 按断点生成多尺寸图片。
2. 依据布局写 `sizes`，不要固定 `100vw`。
3. 在 DevTools 中验证真实下载尺寸是否匹配容器。

## 7. 图片懒加载：loading 与 Intersection Observer

### 7.1 原生懒加载

```html
<img src="/images/list-1.webp" loading="lazy" decoding="async" alt="列表图" width="300" height="200" />
```

### 7.2 Intersection Observer（自定义）

```js
const io = new IntersectionObserver(
  entries => {
    for (const entry of entries) {
      if (!entry.isIntersecting) continue;
      const img = entry.target;
      img.src = img.dataset.src;
      img.srcset = img.dataset.srcset || '';
      io.unobserve(img);
    }
  },
  { rootMargin: '200px 0px' }
);

document.querySelectorAll('img[data-src]').forEach(img => io.observe(img));
```

规则：

- 首屏关键图不要懒加载。
- 长列表图统一设置占位高度，避免滚动抖动。

## 8. SVG 优化：SVGO

```bash
npm i -D svgo
```

```js
// svgo.config.js
module.exports = {
  multipass: true,
  plugins: [
    'preset-default',
    { name: 'removeViewBox', active: false },
    { name: 'cleanupIds', active: true },
    { name: 'sortAttrs', active: true },
  ],
};
```

```bash
npx svgo -f src/icons -o src/icons-optimized
```

对图标系统：

- 优先使用 SVG Sprite 或按需组件导入。
- 避免把整套图标库一次性注入首屏。

## 9. 占位图策略：LQIP/BlurHash/主色块

### 9.1 LQIP（低质量预览）

```js
import sharp from 'sharp';

const placeholder = await sharp('hero.jpg').resize(24).blur(8).toBuffer();
const base64 = `data:image/jpeg;base64,${placeholder.toString('base64')}`;
```

### 9.2 BlurHash

```bash
npm i blurhash
```

在服务端生成 BlurHash，前端先渲染 Canvas 占位，再替换真实图。

### 9.3 主色块占位

提取图片主色并渲染背景块，适合列表与卡片流。

```css
.image-shell {
  background: #e7edf5;
}
```

## 10. 验证收益与质量兜底

### 10.1 验证命令

```bash
npx lighthouse http://localhost:4173 --only-categories=performance --view
```

### 10.2 验证清单

1. LCP 图片下载时间下降。
2. 首屏图片总体积下降。
3. 视觉质量可接受（人工抽样 + 设计确认）。
4. CLS 无恶化（保留尺寸占位）。

### 10.3 常见故障处理

- WebP/AVIF 404：检查产物路径与 CDN 回源策略。
- `srcset` 不生效：检查 `sizes` 是否错误导致浏览器选错资源。
- 首屏变慢：误把 LCP 图设置为懒加载，立即回退。
- 缓存错配：CDN 参数变化后未刷新缓存，执行 URL 版本化。
