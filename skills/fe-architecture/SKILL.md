---
name: fe-architecture
description: |
  前端架构演进技能包，涵盖框架大版本升级、微前端架构、TypeScript 迁移、CSS 架构迁移。
  当需要升级框架版本（Vue2→3、React Class→Hooks、Angular 升级）、引入微前端、迁移到 TypeScript、或重构 CSS 架构时触发此技能。
  支持 Angular、Vue、React、原生 JS、Node.js 技术栈。
---

# 前端架构演进执行路由

按“现状 → 目标态 → 参考文档”选择路径，先做风险评估，再执行分阶段迁移。

## 1. 架构决策树

### 1.1 框架层

1. 判断现状：
   - Vue 2.x（Options API、Vuex、Vue Router 3）
   - React Class 组件占比高（生命周期复杂、HOC 多）
   - Angular 跨多个 major 版本落后
2. 判断目标：
   - 升级到 Vue 3 + Composition API + Vue Router 4 + Pinia
   - 迁移到 React Hooks + modern Context
   - 升级到目标 Angular major，并推进 Standalone
3. 路由到文档：
   - 打开 `references/framework-upgrade.md`

### 1.2 应用边界层

1. 判断现状：
   - 单体前端发布慢、团队并行冲突高、基座耦合重
2. 判断目标：
   - 微前端拆分独立发布
   - 明确主子应用边界与通信协议
3. 路由到文档：
   - 打开 `references/micro-frontend.md`

### 1.3 类型系统层

1. 判断现状：
   - JavaScript 项目类型不稳定、线上类型错误频繁
2. 判断目标：
   - 渐进迁移到 TypeScript，最终启用严格模式
3. 路由到文档：
   - 打开 `references/typescript-migration.md`

### 1.4 样式架构层

1. 判断现状：
   - 全局样式污染、选择器冲突、主题扩展困难
2. 判断目标：
   - Tailwind、CSS Modules、CSS-in-JS、Design Token 体系化落地
3. 路由到文档：
   - 打开 `references/css-architecture.md`

## 2. 迁移前风险评估清单（必须逐项勾选）

1. 冻结窗口：
   - 设定 code freeze 时间段，禁止无关功能并行进入主干。
2. 基线可观测性：
   - 建立迁移前基线：build 时长、bundle 体积、TTI、关键错误率。
3. 测试覆盖：
   - 核对单元、集成、E2E 覆盖关键业务路径。
4. 依赖兼容性：
   - 输出第三方依赖兼容矩阵：版本、替代方案、阻断级别。
5. 回滚机制：
   - 准备回滚分支、版本 tag、配置开关（feature flag / runtime switch）。
6. 数据与协议：
   - 审核 API 契约与事件协议，禁止迁移中“隐式改字段”。
7. 发布策略：
   - 明确灰度比例、监控指标阈值、熔断条件与负责人。
8. 组织协同：
   - 指定迁移 owner、reviewer、应急联系人和决策链路。

## 3. 通用迁移原则

1. 优先执行“渐进式迁移”，避免“大爆炸重写”。
2. 先迁移低风险、低耦合模块，再迁移高风险核心模块。
3. 每一步都保持“可编译、可测试、可发布”。
4. 强制小步提交：单次变更只做一类演进动作。
5. 先建立兼容层，再替换调用方，最后删除旧实现。
6. 为关键差异建立 lint rule / codemod，减少人工误差。
7. 用数据驱动决策：性能、错误率、开发效率三指标同时达标。
8. 在迁移阶段保留双轨运行能力（compat mode / dual runtime）。

## 4. 文档路由规则

1. 执行框架升级：先读 `references/framework-upgrade.md`。
2. 执行微前端拆分：先读 `references/micro-frontend.md`。
3. 执行 TypeScript 迁移：先读 `references/typescript-migration.md`。
4. 执行 CSS 架构迁移：先读 `references/css-architecture.md`。

## 5. 执行顺序建议

1. 先做“框架与构建层”升级（减少后续迁移阻力）。
2. 再做“类型系统”迁移（提升改造安全性）。
3. 然后做“样式体系”重构（减少视觉回归风险）。
4. 最后做“微前端”拆分（在稳定技术基线之上拆边界）。

## 6. 完成判定

1. 关键业务链路测试全部通过。
2. 线上核心指标不低于迁移前基线。
3. 无 P0/P1 级兼容性缺陷。
4. 旧技术栈入口、依赖与文档已清理。
5. 回滚预案通过演练并可在目标时间内完成。
