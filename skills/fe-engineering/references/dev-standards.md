# 开发规范标准化工作流

## 目录

- [1. 目标与落地顺序](#1-目标与落地顺序)
- [2. ESLint 标准化](#2-eslint-标准化)
- [3. Prettier 与 ESLint 集成](#3-prettier-与-eslint-集成)
- [4. Husky + lint-staged Git Hook](#4-husky--lint-staged-git-hook)
- [5. Commitlint（Conventional Commits）](#5-commitlintconventional-commits)
- [6. EditorConfig 统一编辑器行为](#6-editorconfig-统一编辑器行为)
- [7. TypeScript tsconfig 推荐基线](#7-typescript-tsconfig-推荐基线)
- [8. package.json 脚本模板](#8-packagejson-脚本模板)
- [9. Node 版本锁定（.nvmrc / .node-version）](#9-node-版本锁定nvmrc--node-version)

## 1. 目标与落地顺序

一次性把“代码风格 + 提交流程 + 类型约束 + 运行时版本”统一。

执行顺序：

1. ESLint
2. Prettier
3. Husky + lint-staged
4. Commitlint
5. EditorConfig
6. tsconfig
7. scripts 与 Node 版本锁定

## 2. ESLint 标准化

### 2.1 安装基础依赖

```bash
npm install -D eslint typescript @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

### 2.2 React 项目

```bash
npm install -D eslint-plugin-react eslint-plugin-react-hooks eslint-plugin-jsx-a11y
```

`.eslintrc.cjs`：

```js
module.exports = {
  root: true,
  env: { browser: true, es2022: true, node: true },
  parser: '@typescript-eslint/parser',
  plugins: ['@typescript-eslint', 'react', 'react-hooks', 'jsx-a11y'],
  extends: [
    'eslint:recommended',
    'plugin:@typescript-eslint/recommended',
    'plugin:react/recommended',
    'plugin:react-hooks/recommended',
    'plugin:jsx-a11y/recommended'
  ],
  settings: { react: { version: 'detect' } },
  ignorePatterns: ['dist', 'build', 'node_modules']
}
```

### 2.3 Vue 项目

```bash
npm install -D eslint-plugin-vue vue-eslint-parser
```

`.eslintrc.cjs`：

```js
module.exports = {
  root: true,
  env: { browser: true, es2022: true, node: true },
  parser: 'vue-eslint-parser',
  parserOptions: {
    parser: '@typescript-eslint/parser',
    sourceType: 'module'
  },
  plugins: ['vue', '@typescript-eslint'],
  extends: ['eslint:recommended', 'plugin:vue/vue3-recommended', 'plugin:@typescript-eslint/recommended'],
  ignorePatterns: ['dist', 'node_modules']
}
```

### 2.4 Angular 项目

```bash
npm install -D @angular-eslint/eslint-plugin @angular-eslint/eslint-plugin-template @angular-eslint/template-parser
```

Angular 推荐直接使用 `angular-eslint` 官方方案并由 CLI 集成。

### 2.5 vanilla JS / Node.js

保持 `eslint:recommended` + `@typescript-eslint/recommended`（若用 TS）。

## 3. Prettier 与 ESLint 集成

安装：

```bash
npm install -D prettier eslint-config-prettier eslint-plugin-prettier
```

`.prettierrc.json`：

```json
{
  "semi": false,
  "singleQuote": true,
  "printWidth": 100,
  "trailingComma": "none"
}
```

在 ESLint extends 末尾加入：

```js
'plugin:prettier/recommended'
```

`.prettierignore`：

```text
dist
build
coverage
node_modules
```

## 4. Husky + lint-staged Git Hook

安装：

```bash
npm install -D husky lint-staged
npx husky init
```

设置 pre-commit：

```bash
npx husky add .husky/pre-commit "npx lint-staged"
```

`package.json` 添加：

```json
{
  "lint-staged": {
    "*.{js,jsx,ts,tsx,vue}": [
      "eslint --fix",
      "prettier --write"
    ],
    "*.{json,md,css,scss,less,yml,yaml}": [
      "prettier --write"
    ]
  }
}
```

## 5. Commitlint（Conventional Commits）

安装：

```bash
npm install -D @commitlint/cli @commitlint/config-conventional
```

`commitlint.config.cjs`：

```js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [
      2,
      'always',
      ['feat', 'fix', 'docs', 'style', 'refactor', 'perf', 'test', 'build', 'ci', 'chore', 'revert']
    ],
    'subject-case': [0]
  }
}
```

配置 commit-msg hook：

```bash
npx husky add .husky/commit-msg "npx --no -- commitlint --edit \$1"
```

中文友好建议：

- `subject` 允许中文，如：`feat: 新增订单筛选组件`

## 6. EditorConfig 统一编辑器行为

`.editorconfig`：

```ini
root = true

[*]
charset = utf-8
end_of_line = lf
insert_final_newline = true
indent_style = space
indent_size = 2
trim_trailing_whitespace = true

[*.md]
trim_trailing_whitespace = false
```

## 7. TypeScript tsconfig 推荐基线

### 通用基线（React/Vue/Node）

`tsconfig.json`：

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "strict": true,
    "noImplicitOverride": true,
    "noUncheckedIndexedAccess": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "resolveJsonModule": true,
    "forceConsistentCasingInFileNames": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src"]
}
```

### Angular

优先使用 Angular CLI 生成配置，在其基础上打开 `strict` 系列选项。

### vanilla JS

若暂不启用 TS，先用 JSDoc + `checkJs` 渐进迁移：

```json
{
  "compilerOptions": {
    "checkJs": true,
    "allowJs": true
  }
}
```

## 8. package.json 脚本模板

统一脚本命名，保证跨项目可复用：

```json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext .js,.jsx,.ts,.tsx,.vue",
    "lint:fix": "eslint . --ext .js,.jsx,.ts,.tsx,.vue --fix",
    "format": "prettier . --write",
    "format:check": "prettier . --check",
    "typecheck": "tsc --noEmit",
    "test": "vitest run"
  }
}
```

Angular 可替换为：

- `dev`: `ng serve`
- `build`: `ng build`
- `test`: `ng test --watch=false`

Node.js 项目可替换为：

- `dev`: `node --watch src/index.js` 或 `tsx watch src/index.ts`

## 9. Node 版本锁定（.nvmrc / .node-version）

创建 `.nvmrc`：

```text
20
```

创建 `.node-version`：

```text
20
```

并在 `package.json` 增加：

```json
{
  "engines": {
    "node": ">=20 <23"
  },
  "packageManager": "pnpm@9.12.0"
}
```

最后执行验收：

```bash
npm run lint --if-present
npm run format:check --if-present
npm run typecheck --if-present
npm run test --if-present
npm run build --if-present
```

把失败项修复后再推广到团队模板仓库，确保标准化可持续。
