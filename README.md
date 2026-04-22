# fe-skills — 前端开发技能包 / Frontend Development Skill Pack

> 面向 AI Coding Agent 的中文前端工程化技能包，覆盖工程基建、性能优化、架构演进、质量安全四大领域。
>
> A Chinese-language frontend engineering skill pack for AI coding agents, covering engineering infrastructure, performance optimization, architecture evolution, and quality assurance.

## 技能总览 / Skill Overview

| 技能 / Skill | 名称 / Name | 涵盖领域 / Domains |
|---|---|---|
| [fe-engineering](./skills/fe-engineering/) | 前端工程基建 | 技术栈分析、文档获取、构建工具升级、项目重构、开发规范、Monorepo |
| [fe-performance](./skills/fe-performance/) | 前端性能优化 | 懒加载、图片优化、Service Worker、首屏优化、Core Web Vitals、Bundle 瘦身 |
| [fe-architecture](./skills/fe-architecture/) | 前端架构演进 | 框架升级、微前端、TypeScript 迁移、CSS 架构迁移 |
| [fe-quality](./skills/fe-quality/) | 前端质量与安全 | 安全加固、错误监控、CI/CD 流水线、无障碍合规 |

**支持的技术栈 / Supported Tech Stacks:** Angular · Vue · React · Vanilla JS · Node.js

## 目录结构 / Directory Structure

```
fe-skills/
├── .claude-plugin/
│   ├── plugin.json                   # 插件清单 / Plugin manifest
│   └── marketplace.json              # 市场清单 / Marketplace manifest
├── README.md
└── skills/
    ├── fe-engineering/
    │   ├── SKILL.md                  # 路由入口 / Routing entry
    │   └── references/
    │       ├── stack-analysis.md      # 项目技术栈分析
    │       ├── docs-fetching.md       # 最新文档获取
    │       ├── build-tool-upgrade.md  # 构建工具链升级
    │       ├── refactoring.md         # 前端项目重构
    │       ├── dev-standards.md       # 开发规范标准化
    │       └── monorepo.md           # Monorepo 治理
    ├── fe-performance/
    │   ├── SKILL.md
    │   └── references/
    │       ├── lazy-loading.md        # 懒加载与代码分割
    │       ├── image-optimization.md  # 图片优化（WebP化）
    │       ├── service-worker.md      # Service Worker 缓存
    │       ├── first-paint.md         # 首屏性能优化
    │       ├── core-web-vitals.md     # Core Web Vitals 治理
    │       └── bundle-analysis.md     # Bundle 分析与瘦身
    ├── fe-architecture/
    │   ├── SKILL.md
    │   └── references/
    │       ├── framework-upgrade.md   # 框架大版本升级
    │       ├── micro-frontend.md      # 微前端架构
    │       ├── typescript-migration.md # TypeScript 迁移
    │       └── css-architecture.md    # CSS 架构迁移
    └── fe-quality/
        ├── SKILL.md
        └── references/
            ├── security.md           # 前端安全加固
            ├── error-monitoring.md    # 错误监控集成
            ├── cicd-pipeline.md       # 前端 CI/CD 流水线
            └── accessibility.md       # 无障碍（a11y）合规
```

## 在线安装 / Online Installation

### 方式一：Claude Code（推荐）/ Claude Code (Recommended)

通过自建 marketplace 安装：

Install via custom marketplace:

```
/plugin marketplace add wj100/fe-skills
/plugin install fe-skills@fe-skills-marketplace
```

### 方式二：Gemini CLI

```bash
gemini extensions install https://github.com/wj100/fe-skills
```

### 方式三：GitHub Copilot CLI

```bash
copilot plugin marketplace add wj100/fe-skills
copilot plugin install fe-skills@fe-skills-marketplace
```

### 方式四：OpenCode

直接对 OpenCode 说这句话，它会自动 clone 仓库并复制到 `~/.agents/skills/`：

Just tell OpenCode this sentence — it will clone the repo and copy skills to `~/.agents/skills/` automatically:

```
全局安装这个 skill https://github.com/wj100/fe-skills
```

如果只想装到当前项目（项目级安装）：

For project-level install only:

```
项目级安装这个 skill https://github.com/wj100/fe-skills
```

## 使用方式 / Usage

### 自动触发 / Auto-Trigger

技能会根据对话内容自动触发。以下是各技能的典型触发场景：

Skills trigger automatically based on conversation context. Typical trigger scenarios:

| 你说的话 / What you say | 触发技能 / Triggered skill |
|---|---|
| "分析一下这个项目的技术栈" | fe-engineering |
| "帮我把 Webpack 迁移到 Vite" | fe-engineering |
| "统一一下项目的 ESLint 配置" | fe-engineering |
| "优化一下首屏加载速度" | fe-performance |
| "图片转成 WebP 格式" | fe-performance |
| "配置 Service Worker 缓存" | fe-performance |
| "把 Vue2 升级到 Vue3" | fe-architecture |
| "引入微前端架构" | fe-architecture |
| "项目迁移到 TypeScript" | fe-architecture |
| "加一下 CSP 安全策略" | fe-quality |
| "接入 Sentry 错误监控" | fe-quality |
| "搭建前端 CI/CD 流水线" | fe-quality |

### 手动调用 / Manual Invocation

也可以直接通过 skill 工具手动加载：

You can also manually load skills via the skill tool:

```
# Claude Code
/skill fe-engineering
/skill fe-performance

# OpenCode
@skill fe-engineering
@skill fe-performance
```

### 配合其他技能使用 / Use with Other Skills

fe-skills 专注于前端工程化工作流，与以下技能互补：

fe-skills focuses on frontend engineering workflows and complements these skills:

| 互补技能 / Complementary Skill | 职责 / Responsibility |
|---|---|
| `frontend-design` | UI 视觉设计与美学 / UI visual design & aesthetics |
| `webapp-testing` | Playwright E2E 测试 / Playwright E2E testing |
| `web-design-guidelines` | Web 设计规范检查 / Web design guideline compliance |

## 设计理念 / Design Philosophy

### 渐进式加载 / Progressive Disclosure

每个技能采用三层加载架构，最大化利用上下文窗口：

Each skill uses a three-level loading architecture to maximize context window efficiency:

1. **元数据层 / Metadata** — `name` + `description`，始终在上下文中（~100 tokens）
2. **路由层 / Router** — `SKILL.md` 主体，技能触发时加载（<200 行）
3. **参考层 / Reference** — `references/*.md`，按需加载（150-350 行/文件）

### 工作流导向 / Workflow-Oriented

所有内容都是**可执行的工作流**，不是知识百科。每个参考文件遵循：

All content is **executable workflows**, not encyclopedic knowledge. Each reference file follows:

```
问题识别 → 诊断分析 → 分步执行 → 验证确认
Problem identification → Diagnosis → Step-by-step execution → Verification
```

### 中英混合 / Bilingual Style

说明文字使用中文，技术术语保留英文原文：

Explanations in Chinese, technical terms kept in English:

```markdown
执行 `npm run build` 构建生产环境包，使用 `webpack-bundle-analyzer` 分析产物体积。
```

## 技术栈覆盖 / Tech Stack Coverage

每个参考文件在适用时都会提供框架特定的指导：

Each reference file provides framework-specific guidance where applicable:

| 技术栈 / Stack | 覆盖范围 / Coverage |
|---|---|
| **React** | Next.js, CRA, hooks 模式, React.lazy, ErrorBoundary |
| **Vue** | Vue 2/3, Nuxt, Composition API, Pinia, defineAsyncComponent |
| **Angular** | Angular 14+, standalone components, Angular Universal, NgRx |
| **Vanilla JS** | 原生 DOM API, Web Components, 无框架场景 |
| **Node.js** | 构建工具配置, SSR 服务端, 工具链脚本 |

## 贡献 / Contributing

欢迎通过 Issue 或 PR 贡献内容。请遵循以下原则：

Contributions welcome via Issue or PR. Please follow these principles:

1. **工作流导向** — 提供可执行的步骤，而非理论说明
2. **真实工具** — 只引用真实存在的包名和命令
3. **中英混合** — 中文说明 + 英文术语
4. **框架覆盖** — 尽量覆盖 Angular/Vue/React/Vanilla JS

---

**License:** MIT
