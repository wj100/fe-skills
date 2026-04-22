# Bundle 分析与瘦身

## 目录

- [分析工具配置](#分析工具配置)
- [常见臃肿模式与修复](#常见臃肿模式与修复)
- [Tree Shaking 验证](#tree-shaking-验证)
- [sideEffects 配置](#sideeffects-配置)
- [Chunk 策略优化](#chunk-策略优化)
- [依赖替换策略](#依赖替换策略)
- [持续监控](#持续监控)

## 分析工具配置

### webpack-bundle-analyzer

```bash
pnpm add -D webpack-bundle-analyzer
```

Webpack 配置：

```js
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',        // 生成 HTML 报告
      reportFilename: 'bundle-report.html',
      openAnalyzer: false,
    }),
  ],
};
```

### rollup-plugin-visualizer (Vite)

```bash
pnpm add -D rollup-plugin-visualizer
```

```ts
// vite.config.ts
import { visualizer } from 'rollup-plugin-visualizer';

export default defineConfig({
  plugins: [
    visualizer({
      filename: 'bundle-report.html',
      open: true,
      gzipSize: true,
      brotliSize: true,
    }),
  ],
});
```

### source-map-explorer

```bash
pnpm add -D source-map-explorer

# 先构建生产包（需要 source map）
npm run build

# 分析
npx source-map-explorer dist/assets/*.js
```

### Angular 项目

```bash
# Angular CLI 内置分析
ng build --stats-json
npx webpack-bundle-analyzer dist/your-app/stats.json
```

## 常见臃肿模式与修复

### Moment.js → Day.js

Moment.js 完整包约 300KB（含 locale），Day.js 仅 2KB。

```bash
pnpm remove moment
pnpm add dayjs
```

替换示例：

```js
// 旧代码
import moment from 'moment';
moment().format('YYYY-MM-DD');
moment().add(1, 'day');
moment('2024-01-01').fromNow();

// 新代码
import dayjs from 'dayjs';
import relativeTime from 'dayjs/plugin/relativeTime';
dayjs.extend(relativeTime);

dayjs().format('YYYY-MM-DD');
dayjs().add(1, 'day');
dayjs('2024-01-01').fromNow();
```

### Lodash → lodash-es 或按需引入

```bash
# 方案 1：换用 ES Module 版本
pnpm remove lodash
pnpm add lodash-es
pnpm add -D @types/lodash-es  # TypeScript 项目

# 方案 2：按需引入
import debounce from 'lodash/debounce';  # 只引入单个函数
```

Webpack 配置按需引入（如果无法改代码）：

```bash
pnpm add -D babel-plugin-lodash
```

```json
// .babelrc
{ "plugins": ["lodash"] }
```

### 大型图标库按需引入

```js
// ❌ 全量引入 — 引入整个图标库
import { Icon } from 'antd';
import * as Icons from '@ant-design/icons';

// ✅ 按需引入 — 只引入需要的图标
import { SearchOutlined, UserOutlined } from '@ant-design/icons';
```

Element Plus (Vue)：

```bash
pnpm add -D unplugin-icons unplugin-auto-import
```

### 国际化库瘦身

```js
// Webpack: 只引入需要的 locale
const webpack = require('webpack');
new webpack.ContextReplacementPlugin(/moment[/\\]locale$/, /zh-cn|en/);

// Day.js: 按需加载 locale
import 'dayjs/locale/zh-cn';
dayjs.locale('zh-cn');
```

## Tree Shaking 验证

### 检查是否生效

```bash
# 1. 构建生产包
npm run build

# 2. 在产物中搜索特定未使用的导出
grep -r "unusedFunction" dist/

# 3. 使用 bundle analyzer 查看模块是否被完整引入
```

### 常见 Tree Shaking 失败原因

| 问题 | 原因 | 修复 |
|------|------|------|
| CommonJS 模块 | `require()` 无法静态分析 | 切换到 ESM 版本（如 `lodash-es`） |
| 副作用代码 | 模块顶层有副作用 | 配置 `sideEffects` |
| 桶文件 re-export | `index.ts` 全量 re-export | 直接从源文件导入 |
| Babel 转译 | 转译为 CommonJS | 配置 `modules: false` |

### Babel 配置保留 ESM

```json
{
  "presets": [
    ["@babel/preset-env", { "modules": false }]
  ]
}
```

## sideEffects 配置

在 `package.json` 中声明：

```json
{
  "sideEffects": false
}
```

有副作用的文件需要白名单：

```json
{
  "sideEffects": [
    "*.css",
    "*.scss",
    "*.less",
    "./src/polyfills.ts",
    "./src/global-setup.ts"
  ]
}
```

## Chunk 策略优化

### Vite manualChunks

```ts
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-react': ['react', 'react-dom'],
          'vendor-router': ['react-router-dom'],
          'vendor-ui': ['antd', '@ant-design/icons'],
          'vendor-utils': ['axios', 'dayjs', 'lodash-es'],
        },
      },
    },
  },
});
```

### Webpack splitChunks

```js
// webpack.config.js
module.exports = {
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
          priority: 10,
        },
        common: {
          minChunks: 2,
          priority: 5,
          reuseExistingChunk: true,
        },
      },
    },
  },
};
```

### Angular budgets

```json
// angular.json
{
  "budgets": [
    {
      "type": "initial",
      "maximumWarning": "500kb",
      "maximumError": "1mb"
    },
    {
      "type": "anyComponentStyle",
      "maximumWarning": "4kb",
      "maximumError": "8kb"
    }
  ]
}
```

## 依赖替换策略

| 重量级依赖 | 轻量替代 | 体积对比 |
|-----------|---------|---------|
| moment | dayjs | 300KB → 2KB |
| lodash | lodash-es / radash | 70KB → 按需 |
| numeral | Intl.NumberFormat | 32KB → 0KB（原生） |
| uuid | crypto.randomUUID() | 9KB → 0KB（原生） |
| axios | fetch (原生) / ky / ofetch | 13KB → 0-3KB |
| classnames | clsx | 1.8KB → 0.5KB |
| date-fns（全量） | date-fns（按需）| 75KB → 按需 |

### 使用 bundlephobia 评估

```bash
# 在线查看包体积
# https://bundlephobia.com/package/axios

# 或使用 CLI
npx bundle-phobia-cli axios dayjs lodash-es
```

## 持续监控

### package.json 脚本

```json
{
  "scripts": {
    "analyze": "ANALYZE=true npm run build",
    "size": "size-limit",
    "size:why": "size-limit --why"
  }
}
```

### size-limit 配置

```bash
pnpm add -D size-limit @size-limit/preset-app
```

```json
// package.json
{
  "size-limit": [
    { "path": "dist/**/*.js", "limit": "300 KB", "gzip": true },
    { "path": "dist/vendor*.js", "limit": "150 KB", "gzip": true }
  ]
}
```

### CI 集成

```yaml
# GitHub Actions
- name: Check bundle size
  run: npx size-limit
```

### 验证清单

- [ ] Bundle analyzer 报告已生成并审查
- [ ] 无全量引入的大型库（Moment, Lodash, 图标库）
- [ ] sideEffects 已正确配置
- [ ] Tree shaking 生效（产物中无未使用代码）
- [ ] Chunk 拆分合理（vendor 独立、公共模块复用）
- [ ] 总包体积在预算范围内
