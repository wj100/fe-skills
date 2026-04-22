# Monorepo 治理

## 目录

- [方案选型](#方案选型)
- [pnpm Workspaces 配置](#pnpm-workspaces-配置)
- [Turborepo 集成](#turborepo-集成)
- [Nx 替代方案](#nx-替代方案)
- [共享包模式](#共享包模式)
- [依赖管理](#依赖管理)
- [构建缓存与增量构建](#构建缓存与增量构建)
- [CI/CD 适配](#cicd-适配)

## 方案选型

| 方案 | 适用场景 | 学习成本 | 生态 |
|------|---------|---------|------|
| pnpm Workspaces | 中小型项目，依赖管理为主 | 低 | 广泛 |
| Turborepo | 构建编排为主，需要缓存加速 | 中 | Vercel 生态 |
| Nx | 大型企业项目，需要完整工具链 | 高 | 独立生态 |

**决策流程：**

1. 项目少于 5 个包 → pnpm Workspaces 足够
2. 构建时间成为瓶颈 → 加入 Turborepo
3. 需要代码生成、依赖图可视化、受影响分析 → 考虑 Nx

## pnpm Workspaces 配置

### 初始化

```bash
# 安装 pnpm
npm install -g pnpm

# 初始化项目
pnpm init
```

### 工作区配置

创建 `pnpm-workspace.yaml`：

```yaml
packages:
  - 'packages/*'
  - 'apps/*'
  - 'shared/*'
```

### 典型目录结构

```
monorepo/
├── pnpm-workspace.yaml
├── package.json
├── pnpm-lock.yaml
├── apps/
│   ├── web/                # React/Vue/Angular 主应用
│   ├── admin/              # 后台管理
│   └── mobile/             # H5 应用
├── packages/
│   ├── ui/                 # 共享 UI 组件库
│   ├── utils/              # 工具函数库
│   └── types/              # 共享类型定义
└── shared/
    └── config/             # 共享配置（ESLint, TSConfig）
```

### 常用命令

```bash
# 安装所有依赖
pnpm install

# 给指定包添加依赖
pnpm --filter @mono/web add axios

# 给所有包添加开发依赖
pnpm -w add -D typescript

# 内部包互相引用
pnpm --filter @mono/web add @mono/ui --workspace

# 执行指定包的脚本
pnpm --filter @mono/web dev

# 执行所有包的构建
pnpm -r run build
```

## Turborepo 集成

### 安装

```bash
pnpm add -Dw turbo
```

### 配置 turbo.json

```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**", "build/**"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "lint": {
      "dependsOn": ["^build"]
    },
    "test": {
      "dependsOn": ["build"]
    },
    "type-check": {
      "dependsOn": ["^build"]
    }
  }
}
```

### 执行命令

```bash
# 构建所有包（自动拓扑排序 + 缓存）
pnpm turbo build

# 只构建受影响的包
pnpm turbo build --filter=...[HEAD~1]

# 并行开发多个应用
pnpm turbo dev --filter=@mono/web --filter=@mono/admin
```

## Nx 替代方案

```bash
# 初始化 Nx（可在已有 monorepo 中添加）
npx nx@latest init

# 查看依赖图
npx nx graph

# 只运行受影响的任务
npx nx affected --target=build

# 代码生成
npx nx generate @nx/react:component my-component
```

## 共享包模式

### 共享 UI 库 (packages/ui)

```json
{
  "name": "@mono/ui",
  "version": "0.1.0",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts",
    "dev": "tsup src/index.ts --format cjs,esm --dts --watch"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-dom": "^18.0.0"
  }
}
```

### 共享工具库 (packages/utils)

```json
{
  "name": "@mono/utils",
  "version": "0.1.0",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "scripts": {
    "build": "tsup src/index.ts --format cjs,esm --dts"
  }
}
```

### 共享 TypeScript 配置 (shared/config)

`tsconfig.base.json`：

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  }
}
```

各包继承：

```json
{
  "extends": "../../shared/config/tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist"
  },
  "include": ["src"]
}
```

## 依赖管理

### 依赖提升策略

在 `.npmrc` 中配置：

```ini
# 严格模式：不自动提升（推荐）
shamefully-hoist=false
strict-peer-dependencies=false

# 公共依赖提升（宽松模式）
public-hoist-pattern[]=*eslint*
public-hoist-pattern[]=*prettier*
```

### 版本同步

使用 `syncpack` 统一依赖版本：

```bash
pnpm add -Dw syncpack

# 检查版本不一致
npx syncpack list-mismatches

# 自动修复版本不一致
npx syncpack fix-mismatches
```

### Changesets 版本管理

```bash
pnpm add -Dw @changesets/cli

# 初始化
npx changeset init

# 创建变更记录
npx changeset

# 更新版本号
npx changeset version

# 发布
npx changeset publish
```

## 构建缓存与增量构建

### Turborepo 远程缓存

```bash
# 登录 Vercel（官方远程缓存）
npx turbo login

# 链接项目
npx turbo link

# 或使用自托管缓存
# turbo.json 中配置
```

### 本地缓存策略

Turborepo 默认启用本地缓存（`.turbo/` 目录）：

```bash
# 清除缓存
pnpm turbo build --force

# 查看缓存命中情况
pnpm turbo build --summarize
```

## CI/CD 适配

### GitHub Actions 示例

```yaml
name: CI
on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 2   # 用于 affected 分析

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'

      - run: pnpm install --frozen-lockfile

      # 只构建受影响的包
      - run: pnpm turbo build --filter=...[HEAD~1]
      - run: pnpm turbo test --filter=...[HEAD~1]
      - run: pnpm turbo lint --filter=...[HEAD~1]
```

### 验证清单

- [ ] `pnpm install` 无报错
- [ ] `pnpm turbo build` 所有包构建成功
- [ ] 内部包引用正确解析（`@mono/ui` 等）
- [ ] 缓存命中率符合预期
- [ ] CI 流水线正确执行受影响分析
