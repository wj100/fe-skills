# 最新文档获取工作流

## 目录

- [1. 目标与原则](#1-目标与原则)
- [2. Context7 获取官方文档流程](#2-context7-获取官方文档流程)
- [3. Web 搜索获取 changelog/migration 指南](#3-web-搜索获取-changelogmigration-指南)
- [4. 官方文档入口清单（Angular/Vue/React/Node.js）](#4-官方文档入口清单-angularvuereactnodejs)
- [5. 当前版本 vs 最新版本比对](#5-当前版本-vs-最新版本比对)
- [6. 识别 breaking changes 的读法](#6-识别-breaking-changes-的读法)
- [7. 形成升级结论的输出模板](#7-形成升级结论的输出模板)

## 1. 目标与原则

获取“最新且可执行”的升级依据，不做拍脑袋升级。

执行时遵守：

1. 先拿官方文档，再看社区文章。
2. 先确认版本区间，再读 migration guide。
3. 先识别 breaking changes，再排期改造。
4. 所有结论必须附来源链接或来源工具结果。

## 2. Context7 获取官方文档流程

### Step 1：解析 library ID

对每个核心库执行解析（示例库名）：

- Angular
- Vue
- React
- Node.js
- Vite
- Webpack

执行动作：

1. 调用 “Resolve Context7 Library ID”。
2. 选择最匹配且文档覆盖高的结果。
3. 记录 libraryId（如 `/vercel/next.js` 这种格式）。

### Step 2：查询目标问题

针对升级任务，优先问三类问题：

1. “从 X 升级到 Y 的官方迁移步骤”
2. “该版本的 breaking changes 清单”
3. “配置项变更与废弃项（deprecated）”

示例查询语句（可直接复用）：

- `How to migrate from webpack 4 to webpack 5 with breaking changes`
- `Vue 2 to Vue 3 migration guide key breaking changes`
- `Angular CLI ng update migration steps from v16 to v17`
- `React 18 upgrade guide and strict mode changes`

### Step 3：结果整理

将 Context7 返回信息按以下结构记录：

- 文档主题
- 关键版本区间
- 必做项（must-do）
- 可选项（optional）
- 风险项（breaking/deprecated）

## 3. Web 搜索获取 changelog/migration 指南

当 Context7 信息不完整或需要最新发布日志时，执行 Web 搜索。

推荐查询模板：

```text
<library> official changelog <major-version>
<library> migration guide <from-version> to <to-version>
<library> release notes breaking changes
```

示例：

```text
vite official changelog 5
webpack migration guide 4 to 5
angular release notes v17 breaking changes
react 19 release notes
node.js 22 changelog
```

筛选规则：

1. 优先 `official`, `docs`, `github releases`。
2. 社区文章仅用于补充案例，不作为唯一依据。
3. 若官方文档与社区文章冲突，以官方文档为准。

## 4. 官方文档入口清单（Angular/Vue/React/Node.js）

### Angular

- 主站：`https://angular.dev/`
- 更新指南：`https://angular.dev/update-guide`
- CLI 文档：`https://angular.dev/tools/cli`

### Vue

- 主站：`https://vuejs.org/`
- 迁移指南（Vue 2 -> Vue 3）：`https://v3-migration.vuejs.org/`
- Router：`https://router.vuejs.org/`
- Pinia：`https://pinia.vuejs.org/`

### React

- 主站：`https://react.dev/`
- 升级相关：`https://react.dev/blog`
- Router（常见配套）：`https://reactrouter.com/`

### Node.js

- 主站：`https://nodejs.org/en/docs`
- 发布与变更：`https://nodejs.org/en/blog`
- EOL 计划：`https://endoflife.date/nodejs`

## 5. 当前版本 vs 最新版本比对

先看本地版本：

```bash
node -v
node -p "require('./package.json').dependencies"
node -p "require('./package.json').devDependencies"
```

使用包管理器查看可升级项：

```bash
# npm
npm outdated

# pnpm
pnpm outdated

# yarn classic
yarn outdated
```

查看某个包的最新版本：

```bash
npm view vite version
npm view webpack version
npm view react version
npm view vue version
npm view @angular/core version
```

对比策略：

1. 标记 `Current`, `Wanted`, `Latest`。
2. `Major` 升级单独列出风险。
3. 优先处理安全补丁和构建工具链关键依赖。

## 6. 识别 breaking changes 的读法

阅读 changelog 时，按以下关键词扫描：

- `Breaking Changes`
- `Deprecated`
- `Removed`
- `Default behavior changed`
- `Minimum Node version`

将每条 breaking change 映射到项目影响面：

1. 配置层（webpack/vite/jest/tsconfig）
2. 代码层（API 变更、生命周期变化）
3. 工具层（Node 版本要求、CI 镜像要求）

示例记录格式：

```md
- 变更：Vite 5 要求 Node >= 18
  - 影响：现有 CI 使用 Node 16
  - 动作：更新 .nvmrc、CI image、本地开发文档
```

## 7. 形成升级结论的输出模板

```md
# 文档调研结论

## 1) 调研范围
- 核心库：
- 当前版本：
- 目标版本：

## 2) 来源证据
- 官方文档：
- 官方 changelog / release notes：
- Context7 结果摘要：

## 3) breaking changes
- [高] 
- [中] 
- [低] 

## 4) 升级策略
- 先升级：
- 后升级：
- 暂缓：

## 5) 实施前置条件
- Node 版本：
- CI/CD 变更：
- 测试补齐：

## 6) 建议执行顺序
1.
2.
3.
```

执行完成后，把该结论作为 `build-tool-upgrade.md` 或 `refactoring.md` 的输入，不要直接跳过验证阶段。
