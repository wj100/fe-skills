---
name: fe-performance
description: |
  前端性能优化技能包，涵盖懒加载与代码分割、图片优化（WebP化）、Service Worker 缓存、首屏性能优化、Core Web Vitals 治理、Bundle 分析与瘦身。
  当需要优化前端加载速度、减小包体积、改善用户体验指标、配置缓存策略、或进行性能诊断时触发此技能。
  支持 Angular、Vue、React、原生 JS、Node.js 技术栈。
---

# 前端性能优化路由手册

按“症状 → 诊断 → 定位 → 修复 → 验证”的顺序执行。不要一次性并行改太多项，避免回归来源不明。

## 1. 快速诊断总流程（5~20 分钟）

1. 基线采集
   - 运行 `npx lighthouse http://localhost:4173 --view`
   - 在 Chrome DevTools `Performance` 录制一次冷启动与一次热启动。
   - 在 Network 面板确认是否启用缓存、HTTP/2/HTTP/3、压缩（br/gzip）。
2. 指标分桶
   - 首屏慢（FCP/LCP 差）
   - 可交互慢（TBT/INP 差）
   - 布局抖动（CLS 差）
   - 包体积大（JS/CSS 传输过大）
   - 离线/弱网重复加载（缓存策略差）
3. 按症状路由到对应参考文档（见下一节）。
4. 每完成一个修复，重复 Lighthouse 与真实设备复测。

## 2. 决策树（症状到文档）

### A. 首屏白屏久、LCP 超标

- 先读 `references/first-paint.md`
- 同时读 `references/image-optimization.md`
- 当 LCP 元素是大图时，优先图片链路；当 LCP 元素是文本/容器时，优先 Critical CSS、字体与资源提示。

### B. 首次加载 JS 体积大、路由切换卡

- 先读 `references/lazy-loading.md`
- 再读 `references/bundle-analysis.md`
- 当路由级 chunk 不合理时先做 code splitting；当第三方库过大时先做依赖替换与 tree shaking。

### C. INP 高、点击响应慢、长任务多

- 先读 `references/core-web-vitals.md`
- 再读 `references/lazy-loading.md`（减少首批执行 JS）
- 必要时读 `references/bundle-analysis.md`（定位重库导致主线程阻塞）。

### D. CLS 高、页面抖动

- 先读 `references/core-web-vitals.md`
- 再读 `references/image-optimization.md`（尺寸占位、响应式图片）
- 若由字体切换触发，回到 `references/first-paint.md` 的字体治理章节。

### E. 弱网/离线体验差、二次访问无提升

- 先读 `references/service-worker.md`
- 按资源类型拆分策略：静态资源 `CacheFirst`，HTML/API 视业务用 `NetworkFirst` 或 `StaleWhileRevalidate`。

### F. Lighthouse 分数波动大、改动后收益不稳

- 先读 `references/core-web-vitals.md`（监控与上报）
- 再读 `references/bundle-analysis.md`（构建产物可视化 + 回归对比）

## 3. 常见指标到行动映射

- 当 **LCP 超标**时，阅读 `references/first-paint.md` 和 `references/image-optimization.md`。
- 当 **INP 超标**时，阅读 `references/core-web-vitals.md` 和 `references/lazy-loading.md`。
- 当 **CLS 超标**时，阅读 `references/core-web-vitals.md` 和 `references/image-optimization.md`。
- 当 **JS 传输体积 > 300KB（gzip）**时，阅读 `references/bundle-analysis.md` 和 `references/lazy-loading.md`。
- 当 **二次访问仍慢**时，阅读 `references/service-worker.md`。
- 当 **字体闪烁、首屏样式延迟**时，阅读 `references/first-paint.md`。

## 4. 统一执行模板（每一项优化都复用）

1. 定义单一目标
   - 示例：`LCP 3.8s → 2.5s 内`。
2. 锁定瓶颈
   - 用 Lighthouse 诊断项 + DevTools 时间线定位具体资源/任务。
3. 执行最小改动
   - 一次只改一个方向：图片、拆包、缓存、关键渲染路径、第三方脚本。
4. 验证收益
   - 对比改动前后：指标、请求数、传输体积、主线程阻塞时间。
5. 防回归
   - 把关键阈值接入 CI（例如 Lighthouse CI、bundlesize）。

## 5. 文档读取顺序建议

- 冷启动慢：`first-paint.md` → `image-optimization.md` → `lazy-loading.md`
- 交互慢：`core-web-vitals.md` → `bundle-analysis.md`
- 离线与缓存：`service-worker.md`
- 大项目长期治理：`bundle-analysis.md` → `core-web-vitals.md`（监控闭环）

## 6. 交付检查清单

- 至少完成一次“实验组 vs 对照组”指标对比。
- 提供改动清单（配置、代码、构建参数）。
- 输出可复现命令（构建、分析、压测、审计）。
- 明确风险点（缓存更新、SSR 兼容、老设备回退策略）。

## 7. references 索引

- 懒加载与代码分割：`references/lazy-loading.md`
- 图片优化：`references/image-optimization.md`
- Service Worker 缓存：`references/service-worker.md`
- 首屏性能优化：`references/first-paint.md`
- Core Web Vitals 治理：`references/core-web-vitals.md`
- Bundle 分析与瘦身：`references/bundle-analysis.md`
