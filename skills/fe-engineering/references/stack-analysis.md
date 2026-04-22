# 项目技术栈分析工作流

## 目录

- [1. 目标与输出](#1-目标与输出)
- [2. 分析前准备](#2-分析前准备)
- [3. 检测矩阵（框架/构建/样式/状态/测试）](#3-检测矩阵框架构建样式状态测试)
- [4. 分步执行（可直接复制命令）](#4-分步执行可直接复制命令)
- [5. 各技术维度识别规则](#5-各技术维度识别规则)
- [6. 技术画像模板（标准输出）](#6-技术画像模板标准输出)
- [7. 多技术栈并存与冲突判定](#7-多技术栈并存与冲突判定)

## 1. 目标与输出

执行技术栈分析时，做三件事：

1. 识别运行时与包管理（Node.js、npm/yarn/pnpm）。
2. 识别框架、构建工具、CSS 方案、状态管理、测试体系。
3. 输出结构化“技术画像”，用于升级、重构、规范化的前置输入。

最终输出必须包含：

- 主框架（Angular/Vue/React/vanilla）
- 构建工具（Webpack/Vite/Rollup/esbuild/Turbopack）
- 语言层（TypeScript/JavaScript）
- CSS 方案
- 状态管理
- 测试方案
- Node 与包管理版本
- 风险项（如多套构建并存、旧版依赖）

## 2. 分析前准备

在项目根目录执行：

```bash
pwd
ls -la
node -v
npm -v || true
pnpm -v || true
yarn -v || true
```

若是 Monorepo，先识别 workspace：

```bash
ls -la pnpm-workspace.yaml package.json turbo.json nx.json 2>/dev/null
```

检查关键配置文件是否存在：

```bash
ls -la \
  package.json package-lock.json pnpm-lock.yaml yarn.lock \
  tsconfig.json jsconfig.json \
  vite.config.ts vite.config.js \
  webpack.config.js webpack.config.ts \
  rollup.config.js rollup.config.mjs \
  esbuild.config.js \
  angular.json \
  next.config.js next.config.mjs \
  nuxt.config.ts \
  tailwind.config.js tailwind.config.ts \
  postcss.config.js \
  jest.config.js vitest.config.ts cypress.config.ts playwright.config.ts 2>/dev/null
```

## 3. 检测矩阵（框架/构建/样式/状态/测试）

| 维度 | 主要信号 | 次要信号 | 判定策略 |
|---|---|---|---|
| 框架 | `dependencies` 中核心包 | 配置文件/目录结构 | 先依赖，再配置交叉验证 |
| 构建工具 | `devDependencies` + 配置文件 | scripts 命令 | 以“可执行脚本 + 配置文件”双命中为准 |
| CSS | CSS 工具包 | 对应配置文件 | 依赖与配置任一命中即记录 |
| 状态管理 | Redux/Pinia/Vuex/NgRx 等 | 源码 import | 先 package，再源码引用确认使用率 |
| 测试 | Jest/Vitest/Cypress/Playwright | 配置文件、scripts | 关注单测/组件测/E2E 分层 |

## 4. 分步执行（可直接复制命令）

### Step 1：读取 package.json 基础信息

```bash
jq '{name, private, type, engines, packageManager, scripts, dependencies, devDependencies}' package.json
```

无 `jq` 时使用：

```bash
node -e "const p=require('./package.json');console.log(JSON.stringify({name:p.name,engines:p.engines,packageManager:p.packageManager,scripts:p.scripts,dependencies:p.dependencies,devDependencies:p.devDependencies},null,2))"
```

### Step 2：识别主框架

使用内容检索模式（可用 grep/rg）：

```bash
# React
grep -E '"react"|"next"' package.json

# Vue
grep -E '"vue"|"nuxt"|"@vue/cli-service"' package.json

# Angular
grep -E '"@angular/core"|"@angular/cli"' package.json
```

判定规则：

- 命中 `@angular/core` → Angular。
- 命中 `vue` 且无 Angular 核心依赖 → Vue。
- 命中 `react` 且无 Angular/Vue 核心依赖 → React。
- 三者都未命中，且存在 `index.html + src/main.js` → vanilla JS。

### Step 3：识别构建工具

```bash
grep -E '"vite"|"webpack"|"rollup"|"esbuild"|"@rspack/core"|"turbopack"' package.json
ls -la vite.config.* webpack.config.* rollup.config.* esbuild.config.* 2>/dev/null
```

脚本信号：

```bash
node -e "const s=require('./package.json').scripts||{};console.log(s)"
```

判定示例：

- `scripts.dev` 含 `vite` + 存在 `vite.config.ts` → 主构建为 Vite。
- `scripts.build` 含 `webpack --mode production` + `webpack.config.js` → 主构建为 Webpack。
- `next build` → 构建由 Next.js（底层可能 Turbopack/Webpack）接管。

### Step 4：识别 Node 与包管理

```bash
cat .nvmrc 2>/dev/null || true
cat .node-version 2>/dev/null || true
node -p "require('./package.json').engines"
node -p "require('./package.json').packageManager"
```

锁文件判定：

- `pnpm-lock.yaml` → pnpm
- `yarn.lock` → yarn
- `package-lock.json` → npm

### Step 5：识别 CSS 方案

```bash
grep -E '"tailwindcss"|"sass"|"less"|"styled-components"|"@emotion/react"|"postcss"' package.json
ls -la tailwind.config.* postcss.config.* 2>/dev/null
```

源码层检测模式：

```bash
grep -R "module\.css\|module\.scss\|styled-components\|@emotion/react" src 2>/dev/null
```

### Step 6：识别状态管理

```bash
grep -E '"redux"|"@reduxjs/toolkit"|"zustand"|"jotai"|"mobx"|"pinia"|"vuex"|"@ngrx/store"' package.json
grep -R "from 'zustand'\|from '@reduxjs/toolkit'\|from 'pinia'\|from 'vuex'\|from '@ngrx/store'" src 2>/dev/null
```

### Step 7：识别测试体系

```bash
grep -E '"jest"|"vitest"|"cypress"|"playwright"|"@testing-library/react"|"@testing-library/vue"' package.json
ls -la jest.config.* vitest.config.* cypress.config.* playwright.config.* 2>/dev/null
node -p "require('./package.json').scripts"
```

## 5. 各技术维度识别规则

### 5.1 Angular

识别信号：

- `@angular/core`、`@angular/cli`、`angular.json`
- 常见脚本：`ng serve`、`ng build`、`ng test`

进一步检查：

```bash
grep -E '"@angular/core"|"@angular/cli"|"@ngrx/store"' package.json
ls -la angular.json tsconfig.app.json tsconfig.spec.json 2>/dev/null
```

### 5.2 Vue

识别信号：

- `vue`、`@vitejs/plugin-vue`、`@vue/cli-service`
- `vite.config.ts` 或 `vue.config.js`

```bash
grep -E '"vue"|"@vitejs/plugin-vue"|"@vue/cli-service"|"pinia"|"vuex"' package.json
ls -la vite.config.* vue.config.js nuxt.config.ts 2>/dev/null
```

### 5.3 React

识别信号：

- `react`、`react-dom`、`next`
- `@vitejs/plugin-react`、`react-scripts`

```bash
grep -E '"react"|"react-dom"|"next"|"react-scripts"|"@vitejs/plugin-react"|"@reduxjs/toolkit"|"zustand"|"jotai"' package.json
ls -la next.config.* vite.config.* 2>/dev/null
```

### 5.4 vanilla JS + Node.js

识别信号：

- 无 Angular/Vue/React 核心依赖
- 存在 `esbuild/rollup/webpack/vite` 任一工具
- 或 Node.js 服务端打包脚本（SSR/API BFF）

```bash
grep -E '"express"|"koa"|"fastify"|"esbuild"|"rollup"|"webpack"|"vite"' package.json
```

## 6. 技术画像模板（标准输出）

按以下模板输出，不要只给自然语言段落：

```md
# 技术画像

## 1) 基础信息
- 项目名：
- Node.js：
- 包管理器：npm / yarn / pnpm
- 包管理声明：`packageManager` / lockfile

## 2) 框架与运行时
- 主框架：Angular / Vue / React / vanilla JS
- 关联框架：Next.js / Nuxt / Nest/BFF（如有）

## 3) 构建工具链
- 当前主构建：Webpack / Vite / Rollup / esbuild / Turbopack
- 关键配置文件：
- scripts 入口（dev/build/test）：

## 4) 工程能力
- 语言：TypeScript / JavaScript
- CSS 方案：Tailwind / Sass / Less / CSS Modules / styled-components
- 状态管理：Redux / Zustand / Jotai / Vuex / Pinia / NgRx
- 测试：Jest / Vitest / Cypress / Playwright

## 5) 风险与建议
- 风险1：
- 风险2：
- 下一步动作：
```

## 7. 多技术栈并存与冲突判定

常见冲突模式与处理：

1. `vite` 与 `webpack` 同时存在：
   - 检查 `scripts.build` 实际调用谁。
   - 若两套都在用，标记“迁移中状态”，避免误删旧配置。
2. `vuex` 与 `pinia` 同时存在：
   - 搜索 `createStore` 与 `createPinia` 的真实引用频次。
3. `jest` 与 `vitest` 同时存在：
   - 根据 CI 命令与脚本入口确定主测试栈。
4. Monorepo 多子项目框架不同：
   - 分别输出子包画像，再给出统一基建建议（Node 版本、lint、test、build）。

执行总结时，先陈述“证据”，再给“结论”，最后给“下一步动作”。
