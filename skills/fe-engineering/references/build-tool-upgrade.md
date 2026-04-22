# 构建工具链升级工作流

## 目录

- [1. 升级总流程](#1-升级总流程)
- [2. Webpack → Vite（React/Vue/vanilla）](#2-webpack--vite-reactvuevanilla)
- [3. CRA → Vite 或 Next.js](#3-cra--vite-或-nextjs)
- [4. Vue CLI → Vite](#4-vue-cli--vite)
- [5. Angular CLI 版本升级（ng update）](#5-angular-cli-版本升级ng-update)
- [6. esbuild / Rollup 集成策略](#6-esbuild--rollup-集成策略)
- [7. 通用验证清单](#7-通用验证清单)
- [8. 常见坑与解决方案](#8-常见坑与解决方案)

## 1. 升级总流程

按固定节奏执行：预检查 → 依赖调整 → 配置迁移 → 验证 → 清理。

### Step 1：预检查

```bash
node -v
npm -v || true
pnpm -v || true
yarn -v || true
npm outdated || true
```

检查现有脚本和配置：

```bash
node -p "require('./package.json').scripts"
ls -la webpack.config.* vite.config.* angular.json vue.config.js 2>/dev/null
```

### Step 2：锁定基线

执行当前项目构建与测试，记录基线结果：

```bash
npm run build
npm run test --if-present
```

### Step 3：创建迁移分支

```bash
git checkout -b chore/build-migration
```

## 2. Webpack → Vite（React/Vue/vanilla）

### 2.1 依赖调整

先移除 Webpack 相关依赖（按项目实际选择）：

```bash
npm remove webpack webpack-cli webpack-dev-server html-webpack-plugin mini-css-extract-plugin
```

安装 Vite：

```bash
npm install -D vite
```

React 项目：

```bash
npm install -D @vitejs/plugin-react
```

Vue 项目：

```bash
npm install -D @vitejs/plugin-vue
```

### 2.2 scripts 迁移

把 `package.json` scripts 改为：

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

### 2.3 配置映射

Webpack 常见项映射到 Vite：

| Webpack | Vite |
|---|---|
| `entry` | 默认 `index.html` 入口 |
| `output.path` | `build.outDir` |
| `resolve.alias` | `resolve.alias` |
| `DefinePlugin` | `define` |
| `devServer.proxy` | `server.proxy` |

React 的 `vite.config.ts`：

```ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { fileURLToPath, URL } from 'node:url'

export default defineConfig({
  plugins: [react()],
  resolve: {
    alias: {
      '@': fileURLToPath(new URL('./src', import.meta.url))
    }
  },
  server: {
    port: 3000,
    proxy: {
      '/api': 'http://localhost:8080'
    }
  },
  build: {
    outDir: 'dist'
  }
})
```

Vue 的 `vite.config.ts`：

```ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  server: {
    port: 5173
  }
})
```

vanilla 的最小配置：

```ts
import { defineConfig } from 'vite'

export default defineConfig({
  build: { outDir: 'dist' }
})
```

### 2.4 验证

```bash
npm run dev
npm run build
```

## 3. CRA → Vite 或 Next.js

### 3.1 CRA → Vite（React SPA）

移除 CRA：

```bash
npm remove react-scripts
npm install -D vite @vitejs/plugin-react
```

新增 `index.html`（Vite 需要根目录模板），把 CRA 的 `%PUBLIC_URL%` 资源引用改为绝对路径或 `/` 前缀。

环境变量改造：

- CRA：`process.env.REACT_APP_*`
- Vite：`import.meta.env.VITE_*`

批量替换示意：

```bash
grep -R "REACT_APP_" src
```

### 3.2 CRA → Next.js（需要 SSR/路由约定）

初始化 Next：

```bash
npx create-next-app@latest my-app --ts --eslint --src-dir
```

迁移策略：

1. 先迁页面与组件。
2. 再迁路由逻辑（React Router → Next File Routing）。
3. 最后迁 API 请求与 SSR/SSG 策略。

## 4. Vue CLI → Vite

### 4.1 移除 Vue CLI 依赖

```bash
npm remove @vue/cli-service
npm install -D vite @vitejs/plugin-vue
```

### 4.2 scripts 调整

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  }
}
```

### 4.3 配置迁移

`vue.config.js` 常见配置迁移：

- `devServer.proxy` → `server.proxy`
- `configureWebpack.resolve.alias` → `resolve.alias`
- `css.loaderOptions` → Vite CSS 配置或 PostCSS/Sass 配置

## 5. Angular CLI 版本升级（ng update）

### 5.1 检查当前版本

```bash
npx ng version
```

### 5.2 执行官方升级

```bash
npx ng update @angular/core @angular/cli
```

若涉及 major 升级，先参考 Angular Update Guide，再逐级升（例如 15 → 16 → 17）。

常见附带升级：

```bash
npx ng update @angular/material
```

### 5.3 验证

```bash
npx ng build
npx ng test --watch=false
```

## 6. esbuild / Rollup 集成策略

### 6.1 Rollup（适合库构建）

安装：

```bash
npm install -D rollup @rollup/plugin-node-resolve @rollup/plugin-commonjs @rollup/plugin-typescript typescript
```

`rollup.config.mjs` 示例：

```js
import resolve from '@rollup/plugin-node-resolve'
import commonjs from '@rollup/plugin-commonjs'
import typescript from '@rollup/plugin-typescript'

export default {
  input: 'src/index.ts',
  output: [
    { file: 'dist/index.cjs', format: 'cjs', sourcemap: true },
    { file: 'dist/index.mjs', format: 'esm', sourcemap: true }
  ],
  plugins: [resolve(), commonjs(), typescript({ tsconfig: './tsconfig.json' })],
  external: ['react', 'vue']
}
```

### 6.2 esbuild（适合工具脚本/小型服务）

安装：

```bash
npm install -D esbuild
```

`scripts/build.mjs` 示例：

```js
import { build } from 'esbuild'

await build({
  entryPoints: ['src/index.ts'],
  bundle: true,
  platform: 'node',
  target: 'node20',
  outfile: 'dist/index.js',
  sourcemap: true,
  minify: false
})
```

## 7. 通用验证清单

每次迁移后都执行：

```bash
npm run lint --if-present
npm run typecheck --if-present
npm run test --if-present
npm run build
```

再验证产物：

```bash
ls -la dist build 2>/dev/null
```

如果有 E2E：

```bash
npm run e2e --if-present
```

## 8. 常见坑与解决方案

1. **Node 版本不满足**
   - 现象：Vite/Angular 新版本安装失败。
   - 处理：升级 Node，更新 `.nvmrc` 与 CI 镜像。

2. **环境变量失效（CRA → Vite）**
   - 现象：`process.env.REACT_APP_*` undefined。
   - 处理：改成 `import.meta.env.VITE_*`，并重命名 `.env` 变量前缀。

3. **静态资源路径错误**
   - 现象：部署后 404。
   - 处理：检查 `base`（Vite）或 publicPath 配置。

4. **Webpack loader 能力迁移遗漏**
   - 现象：Less/Sass、SVG 处理行为不同。
   - 处理：对照 loader 功能逐项迁移到 Vite 插件或原生能力。

5. **Jest/Vitest 双栈冲突**
   - 现象：测试命令混乱。
   - 处理：指定主测试栈，另一个逐步下线，统一 `npm test` 入口。

6. **Angular 升级跨度过大**
   - 现象：大量 schematics 失败。
   - 处理：按 major 逐级升级并每级跑完整测试。

执行结束后，删除废弃配置和无用依赖，保持工具链单一来源。
