---
name: fe-engineering
description: |
  前端工程基建技能包，涵盖项目技术栈分析、最新文档获取、构建工具链升级、前端项目重构、开发规范标准化、Monorepo 治理。
  当需要分析前端项目架构、升级构建工具、统一开发规范、管理 Monorepo、或进行系统性重构时触发此技能。
  支持 Angular、Vue、React、原生 JS、Node.js 技术栈。
---

# 前端工程基建技能路由

本技能是“路由层”，只负责快速定位到正确的工作流文档。执行任务时先做场景判断，再按指引加载对应 `references/*.md`。

## 覆盖范围

- 技术栈识别与技术画像输出
- 最新官方文档/变更日志获取
- 构建工具链迁移与升级
- 系统性重构与自动化改造
- 开发规范、代码风格、提交流程统一
- Monorepo 治理（pnpm workspace / Turborepo / Nx）

## 场景路由表

| 场景 | 读取文件 | 执行动作 |
|---|---|---|
| 先摸清项目：框架、构建工具、测试、状态管理 | `references/stack-analysis.md` | 按步骤生成“技术画像” |
| 需要拿最新文档、迁移指南、breaking changes | `references/docs-fetching.md` | 用 Context7 + Web 搜索做证据化结论 |
| 要做 Webpack/CRA/Vue CLI/Angular CLI 升级迁移 | `references/build-tool-upgrade.md` | 执行“预检查 → 迁移 → 验证 → 清理” |
| 要进行代码/架构重构（含自动化批改） | `references/refactoring.md` | 先影响分析，再分批重构与验证 |
| 要统一 ESLint/Prettier/Husky/Commitlint/TSConfig | `references/dev-standards.md` | 一次性落地工程规范 |
| 要拆多包、做共享包、启用缓存与增量构建 | `references/monorepo.md` | 建立 Monorepo 治理基线 |

## 决策树（简版）

1. 如果你**还不知道项目现状**，先阅读 `references/stack-analysis.md`。
2. 如果你要**判断是否升级以及升级风险**，接着阅读 `references/docs-fetching.md`。
3. 如果目标是**构建链路迁移/升级**，阅读 `references/build-tool-upgrade.md`。
4. 如果目标是**业务代码重构**，阅读 `references/refactoring.md`。
5. 如果目标是**统一工程规范**，阅读 `references/dev-standards.md`。
6. 如果目标是**多包治理/提效**，阅读 `references/monorepo.md`。

## 快速开始（最常见：技术栈分析）

在项目根目录执行：

```bash
# 1) 基础环境
node -v
npm -v || true
pnpm -v || true
yarn -v || true

# 2) 关键文件探测
ls -la
ls -la package.json tsconfig.json vite.config.ts webpack.config.js angular.json nuxt.config.ts next.config.js 2>/dev/null

# 3) 依赖摘要（需要 jq；无 jq 时改用 node -p）
jq '.dependencies, .devDependencies' package.json 2>/dev/null
```

然后严格按 `references/stack-analysis.md` 里的“检测矩阵”和“技术画像模板”输出结论。

## 执行约束

- 当需要进行构建工具升级时，阅读 `references/build-tool-upgrade.md`。
- 当需要拉取最新官方文档与迁移说明时，阅读 `references/docs-fetching.md`。
- 当需要进行系统重构与自动化改造时，阅读 `references/refactoring.md`。
- 当需要统一开发规范与提交流程时，阅读 `references/dev-standards.md`。
- 当需要实施 Monorepo 治理时，阅读 `references/monorepo.md`。

每次只加载与当前目标直接相关的 reference，避免一次性加载全部文档。
