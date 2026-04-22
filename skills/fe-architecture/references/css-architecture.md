# CSS 架构迁移

## 目录

- [方案选型](#方案选型)
- [BEM/Plain CSS → Tailwind CSS](#bemplain-css--tailwind-css)
- [Less/Sass → CSS Modules](#lesssass--css-modules)
- [CSS-in-JS 方案](#css-in-js-方案)
- [Design Token 体系](#design-token-体系)
- [Dark Mode 实现](#dark-mode-实现)

## 方案选型

| 方案 | 优势 | 劣势 | 适用场景 |
|------|------|------|---------|
| Tailwind CSS | 开发效率高、产物小、一致性强 | 类名冗长、学习曲线 | 新项目、快速迭代 |
| CSS Modules | 天然作用域隔离、零运行时 | 样式复用不便 | 中大型项目 |
| styled-components | 动态样式方便、与组件耦合 | 运行时开销 | React 项目 |
| vanilla-extract | 零运行时、TS 类型安全 | 生态较新 | 追求性能的 TS 项目 |
| UnoCSS | 极快、高度可定制 | 社区较 Tailwind 小 | Vite 项目 |

**决策流程：**

1. 需要极致性能 + TS → vanilla-extract / Panda CSS
2. 快速开发 + 一致性 → Tailwind CSS / UnoCSS
3. 组件级隔离 → CSS Modules
4. 高度动态样式 → styled-components / Emotion

## BEM/Plain CSS → Tailwind CSS

### Phase 1: 安装配置

**React (Vite):**

```bash
pnpm add -D tailwindcss @tailwindcss/postcss postcss
```

```js
// postcss.config.js
export default {
  plugins: {
    '@tailwindcss/postcss': {},
  },
};
```

```css
/* src/index.css */
@import 'tailwindcss';
```

**Vue (Vite):**

```bash
pnpm add -D tailwindcss @tailwindcss/vite
```

```ts
// vite.config.ts
import tailwindcss from '@tailwindcss/vite';
export default defineConfig({
  plugins: [vue(), tailwindcss()],
});
```

**Angular:**

```bash
ng add @angular/tailwindcss
# 或手动安装
pnpm add -D tailwindcss @tailwindcss/postcss postcss
```

### Phase 2: 自定义主题

```css
/* src/index.css */
@import 'tailwindcss';

@theme {
  --color-primary: #3b82f6;
  --color-primary-dark: #2563eb;
  --color-secondary: #64748b;
  --font-family-sans: 'Inter', sans-serif;
  --breakpoint-xs: 475px;
}
```

### Phase 3: 渐进式迁移

**策略：新代码用 Tailwind，旧代码逐步替换。**

```html
<!-- 旧代码 (BEM) -->
<div class="card card--featured">
  <h2 class="card__title">标题</h2>
  <p class="card__content">内容</p>
</div>

<!-- 新代码 (Tailwind) -->
<div class="rounded-lg border border-blue-200 bg-blue-50 p-6 shadow-md">
  <h2 class="text-xl font-bold text-gray-900">标题</h2>
  <p class="mt-2 text-gray-600">内容</p>
</div>
```

**提取可复用组件（避免重复类名）：**

```css
/* 使用 @apply 提取复用样式（适量使用） */
@utility card {
  @apply rounded-lg border p-6 shadow-md;
}
```

### Phase 4: 清理旧 CSS

```bash
# 查找未使用的 CSS 类
npx purgecss --css src/**/*.css --content src/**/*.{html,tsx,vue} --output cleaned/

# 逐步删除旧的 BEM 样式文件
```

## Less/Sass → CSS Modules

### Vite 配置

Vite 原生支持 CSS Modules，无需额外插件：

```ts
// 文件命名约定：*.module.css / *.module.scss / *.module.less
// Vite 自动识别
```

安装预处理器（如需保留 Less/Sass 语法）：

```bash
pnpm add -D less        # Less
pnpm add -D sass        # Sass
```

### Webpack 配置

```js
module.exports = {
  module: {
    rules: [
      {
        test: /\.module\.css$/,
        use: [
          'style-loader',
          { loader: 'css-loader', options: { modules: true } },
        ],
      },
    ],
  },
};
```

### 迁移步骤

```bash
# 1. 重命名文件
mv src/components/Button.less src/components/Button.module.less

# 2. 修改引入方式
```

```tsx
// React — 旧代码
import './Button.less';
<button className="btn btn-primary">

// React — 新代码
import styles from './Button.module.less';
<button className={styles.btnPrimary}>
```

```vue
<!-- Vue — 直接使用 scoped（已自带隔离） -->
<style lang="less" scoped>
.btn { /* ... */ }
</style>

<!-- Vue — 或使用 CSS Modules -->
<style lang="less" module>
.btn { /* ... */ }
</style>
<template>
  <button :class="$style.btn">按钮</button>
</template>
```

### 类名组合

```bash
pnpm add clsx   # 轻量类名合并工具
```

```tsx
import styles from './Button.module.css';
import clsx from 'clsx';

function Button({ variant, disabled }) {
  return (
    <button className={clsx(styles.btn, styles[variant], { [styles.disabled]: disabled })}>
      按钮
    </button>
  );
}
```

## CSS-in-JS 方案

### styled-components (React)

```bash
pnpm add styled-components
pnpm add -D @types/styled-components  # TypeScript
```

```tsx
import styled from 'styled-components';

const Button = styled.button<{ $primary?: boolean }>`
  padding: 8px 16px;
  border-radius: 4px;
  background: ${props => props.$primary ? '#3b82f6' : '#fff'};
  color: ${props => props.$primary ? '#fff' : '#333'};
`;
```

### vanilla-extract (零运行时，推荐)

```bash
pnpm add @vanilla-extract/css
pnpm add -D @vanilla-extract/vite-plugin  # Vite
```

```ts
// vite.config.ts
import { vanillaExtractPlugin } from '@vanilla-extract/vite-plugin';
export default defineConfig({
  plugins: [vanillaExtractPlugin()],
});
```

```ts
// Button.css.ts
import { style, createTheme } from '@vanilla-extract/css';

export const button = style({
  padding: '8px 16px',
  borderRadius: '4px',
  ':hover': { opacity: 0.9 },
});
```

### Panda CSS (零运行时 + Tailwind 风格)

```bash
pnpm add -D @pandacss/dev
npx panda init
```

```tsx
import { css } from '../styled-system/css';

function Button() {
  return (
    <button className={css({ px: '4', py: '2', bg: 'blue.500', color: 'white' })}>
      按钮
    </button>
  );
}
```

## Design Token 体系

### CSS Custom Properties 架构

```css
/* tokens/primitive.css — 原始值 */
:root {
  --color-blue-500: #3b82f6;
  --color-blue-600: #2563eb;
  --color-gray-50: #f9fafb;
  --color-gray-900: #111827;
  --spacing-1: 0.25rem;
  --spacing-2: 0.5rem;
  --spacing-4: 1rem;
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
}

/* tokens/semantic.css — 语义化 */
:root {
  --color-primary: var(--color-blue-500);
  --color-primary-hover: var(--color-blue-600);
  --color-bg-page: var(--color-gray-50);
  --color-text-primary: var(--color-gray-900);
}

/* tokens/component.css — 组件级 */
:root {
  --button-bg: var(--color-primary);
  --button-bg-hover: var(--color-primary-hover);
  --button-padding: var(--spacing-2) var(--spacing-4);
  --button-radius: var(--radius-md);
}
```

### Token 层级

```
原始值 (Primitive) → 语义化 (Semantic) → 组件级 (Component)
     蓝色#3b82f6    →    主色 primary     →   按钮背景色
```

## Dark Mode 实现

### CSS 变量切换（推荐）

```css
:root {
  --color-bg: #ffffff;
  --color-text: #111827;
  --color-border: #e5e7eb;
}

[data-theme='dark'] {
  --color-bg: #1f2937;
  --color-text: #f9fafb;
  --color-border: #374151;
}
```

```js
// 切换主题
function toggleTheme() {
  const current = document.documentElement.getAttribute('data-theme');
  document.documentElement.setAttribute('data-theme', current === 'dark' ? 'light' : 'dark');
}
```

### Tailwind Dark Mode

```html
<!-- class 策略（推荐，可手动控制） -->
<div class="bg-white dark:bg-gray-900 text-gray-900 dark:text-gray-100">
  内容
</div>
```

```js
// 切换
document.documentElement.classList.toggle('dark');
```

### 验证清单

- [ ] 新方案已安装并正确配置
- [ ] 旧样式文件已逐步迁移（无残留未使用 CSS）
- [ ] 组件样式隔离正常（无类名冲突）
- [ ] Design Token 层级清晰（primitive → semantic → component）
- [ ] Dark Mode 切换正常
- [ ] 构建产物中无冗余 CSS
