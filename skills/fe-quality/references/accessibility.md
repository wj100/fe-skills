# 无障碍（a11y）合规

## 目录

- [WCAG 核心原则](#wcag-核心原则)
- [自动化检测工具](#自动化检测工具)
- [常见问题与修复](#常见问题与修复)
- [组件级 a11y 模式](#组件级-a11y-模式)
- [测试工作流](#测试工作流)

## WCAG 核心原则

| 原则 | 含义 | 关键要求 |
|------|------|---------|
| 可感知 (Perceivable) | 信息可被所有用户感知 | 图片 alt、视频字幕、颜色对比度 |
| 可操作 (Operable) | 界面可通过多种方式操作 | 键盘导航、充足时间、无闪烁 |
| 可理解 (Understandable) | 信息和操作可被理解 | 清晰标签、一致导航、错误提示 |
| 健壮 (Robust) | 兼容各种辅助技术 | 语义化 HTML、ARIA 属性正确 |

**合规等级：**

| 等级 | 要求 | 适用场景 |
|------|------|---------|
| A | 基础无障碍 | 最低标准，所有网站应达到 |
| AA | 增强无障碍 | **推荐目标**，大多数法规要求 |
| AAA | 最高无障碍 | 特殊场景（政府、医疗） |

## 自动化检测工具

### ESLint 插件

**React:**

```bash
pnpm add -D eslint-plugin-jsx-a11y
```

```js
// eslint.config.js (flat config)
import jsxA11y from 'eslint-plugin-jsx-a11y';

export default [
  jsxA11y.flatConfigs.recommended,
  // ...其他配置
];
```

检测范围：图片 alt、表单 label、可点击元素角色、tabIndex 使用等。

**Vue:**

```bash
pnpm add -D eslint-plugin-vuejs-accessibility
```

```js
// eslint.config.js
import vueA11y from 'eslint-plugin-vuejs-accessibility';

export default [
  ...vueA11y.configs['flat/recommended'],
];
```

**Angular:**

Angular ESLint 默认包含 a11y 规则（`@angular-eslint/template/accessibility-*`）。

```bash
ng add @angular-eslint/schematics
```

### axe-core 集成

```bash
# CLI 快速检测
pnpm add -D @axe-core/cli
npx @axe-core/cli http://localhost:3000

# 开发时浏览器检测（React）
pnpm add -D @axe-core/react
```

```tsx
// React 开发环境检测（仅 dev）
if (import.meta.env.DEV) {
  import('@axe-core/react').then(({ default: axe }) => {
    axe(React, ReactDOM, 1000);  // 控制台输出违规
  });
}
```

**Vue Devtools:**

```bash
pnpm add -D vue-axe  # Vue 3
```

```ts
if (import.meta.env.DEV) {
  import('vue-axe').then(({ default: VueAxe }) => {
    app.use(VueAxe);
  });
}
```

### Lighthouse a11y 审计

```bash
# CLI 方式
npx lighthouse http://localhost:3000 --only-categories=accessibility --output=json

# Chrome DevTools → Lighthouse → Accessibility
```

## 常见问题与修复

### 图片 alt 文本

```html
<!-- ✅ 内容图片 — 提供描述性 alt -->
<img src="chart.png" alt="2024年第一季度销售额增长了35%">

<!-- ✅ 装饰图片 — 空 alt -->
<img src="divider.png" alt="">

<!-- ✅ 按钮中的图标 — aria-label 在按钮上 -->
<button aria-label="关闭对话框">
  <img src="close.svg" alt="">
</button>

<!-- ❌ 缺少 alt -->
<img src="photo.jpg">
```

### 表单标签关联

```html
<!-- ✅ 显式关联 -->
<label for="email">邮箱地址</label>
<input id="email" type="email" aria-describedby="email-hint">
<span id="email-hint">请输入工作邮箱</span>

<!-- ✅ 隐式关联 -->
<label>
  邮箱地址
  <input type="email">
</label>

<!-- ✅ 无可见 label 时 -->
<input type="search" aria-label="搜索文章">
```

### 颜色对比度

```
最低要求（AA）：
- 正文文本：4.5:1
- 大文本（18px+ 或 14px+ 粗体）：3:1
- UI 组件和图形：3:1
```

```css
/* ❌ 对比度不足 */
.text { color: #999; background: #fff; }  /* 2.85:1 */

/* ✅ 对比度达标 */
.text { color: #595959; background: #fff; }  /* 7:1 */
```

检测工具：Chrome DevTools → Elements → 选中元素 → 查看 color 旁的对比度比值。

### 键盘导航

```html
<!-- ✅ 跳过导航链接 -->
<a href="#main-content" class="skip-link">跳到主要内容</a>
<nav><!-- 导航 --></nav>
<main id="main-content"><!-- 主内容 --></main>

<style>
.skip-link {
  position: absolute;
  top: -40px;
  left: 0;
  z-index: 100;
}
.skip-link:focus {
  top: 0;  /* Tab 聚焦时显示 */
}
</style>
```

```tsx
// ✅ 自定义可点击元素 — 必须支持键盘
<div
  role="button"
  tabIndex={0}
  onClick={handleClick}
  onKeyDown={(e) => {
    if (e.key === 'Enter' || e.key === ' ') {
      e.preventDefault();
      handleClick();
    }
  }}
>
  可点击元素
</div>

// ✅ 更好的做法 — 直接用 button
<button onClick={handleClick}>可点击元素</button>
```

### 语义化 HTML

```html
<!-- ✅ 语义化结构 -->
<header>
  <nav aria-label="主导航"><!-- ... --></nav>
</header>
<main>
  <article>
    <h1>文章标题</h1>
    <section aria-labelledby="section-1">
      <h2 id="section-1">章节标题</h2>
    </section>
  </article>
  <aside aria-label="侧边栏"><!-- ... --></aside>
</main>
<footer><!-- ... --></footer>

<!-- ❌ div 汤 -->
<div class="header">
  <div class="nav"><!-- ... --></div>
</div>
<div class="main">
  <div class="article"><!-- ... --></div>
</div>
```

## 组件级 a11y 模式

### Modal / Dialog

```tsx
function Modal({ isOpen, onClose, title, children }) {
  const modalRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    if (isOpen) {
      modalRef.current?.focus();
      document.body.style.overflow = 'hidden';
    }
    return () => { document.body.style.overflow = ''; };
  }, [isOpen]);

  // Escape 关闭
  useEffect(() => {
    const handler = (e: KeyboardEvent) => {
      if (e.key === 'Escape') onClose();
    };
    if (isOpen) document.addEventListener('keydown', handler);
    return () => document.removeEventListener('keydown', handler);
  }, [isOpen, onClose]);

  if (!isOpen) return null;

  return (
    <div role="dialog" aria-modal="true" aria-labelledby="modal-title" ref={modalRef} tabIndex={-1}>
      <h2 id="modal-title">{title}</h2>
      {children}
      <button onClick={onClose}>关闭</button>
    </div>
  );
}
```

### Dropdown / Menu

```html
<button
  aria-haspopup="true"
  aria-expanded="false"
  aria-controls="dropdown-menu"
  id="dropdown-btn">
  菜单
</button>
<ul id="dropdown-menu" role="menu" aria-labelledby="dropdown-btn" hidden>
  <li role="menuitem" tabindex="-1">选项一</li>
  <li role="menuitem" tabindex="-1">选项二</li>
  <li role="menuitem" tabindex="-1">选项三</li>
</ul>
```

键盘交互：↑↓ 切换选项，Enter 选择，Escape 关闭。

### Toast / Notification

```html
<!-- 非紧急通知 -->
<div role="status" aria-live="polite">
  保存成功
</div>

<!-- 紧急告警 -->
<div role="alert" aria-live="assertive">
  网络连接已断开
</div>
```

### Tabs

```html
<div role="tablist" aria-label="设置选项卡">
  <button role="tab" aria-selected="true" aria-controls="panel-1" id="tab-1">基本</button>
  <button role="tab" aria-selected="false" aria-controls="panel-2" id="tab-2" tabindex="-1">高级</button>
</div>
<div role="tabpanel" id="panel-1" aria-labelledby="tab-1">基本设置内容</div>
<div role="tabpanel" id="panel-2" aria-labelledby="tab-2" hidden>高级设置内容</div>
```

键盘交互：←→ 切换 Tab，Tab 键进入 panel 内容。

## 测试工作流

### 自动化测试脚本 (Playwright + axe)

```bash
pnpm add -D @axe-core/playwright
```

```ts
// tests/a11y.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('首页无障碍扫描', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();

  expect(results.violations).toEqual([]);
});

test('表单页无障碍扫描', async ({ page }) => {
  await page.goto('/form');
  const results = await new AxeBuilder({ page })
    .include('#main-form')           // 只扫描特定区域
    .disableRules(['color-contrast']) // 临时跳过特定规则
    .analyze();

  expect(results.violations).toEqual([]);
});
```

### 键盘测试清单

手动测试以下项目：

- [ ] Tab 键可以遍历所有可交互元素
- [ ] Tab 顺序符合视觉阅读顺序
- [ ] 当前焦点元素有清晰的视觉指示（focus ring）
- [ ] 模态框打开时焦点被锁定在内部
- [ ] 模态框关闭后焦点回到触发元素
- [ ] 自定义组件支持 Enter / Space 激活
- [ ] 下拉菜单支持 ↑↓ 导航和 Escape 关闭

### 屏幕阅读器测试

| 平台 | 屏幕阅读器 | 快捷键 |
|------|-----------|--------|
| macOS | VoiceOver | Cmd + F5 |
| Windows | NVDA (免费) | Ctrl + Alt + N |
| Windows | Narrator | Win + Ctrl + Enter |

检查要点：
- 页面标题被正确朗读
- 图片 alt 文本有意义
- 表单 label 与输入框关联
- 动态内容变化被播报（aria-live）
- 页面标题层级清晰（h1 → h2 → h3）

### 验证清单

- [ ] ESLint a11y 插件已配置且无报错
- [ ] axe-core 扫描无 serious/critical 违规
- [ ] 所有图片有适当的 alt 文本
- [ ] 所有表单有关联的 label
- [ ] 颜色对比度达到 AA 标准（4.5:1）
- [ ] 键盘可完整导航所有功能
- [ ] ARIA 属性使用正确
- [ ] 自动化 a11y 测试已加入 CI
