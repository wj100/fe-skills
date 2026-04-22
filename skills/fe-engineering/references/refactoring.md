# 前端项目重构工作流

## 目录

- [1. 重构总策略](#1-重构总策略)
- [2. 重构前影响分析](#2-重构前影响分析)
- [3. 安全重构通用模式](#3-安全重构通用模式)
- [4. React 重构模式](#4-react-重构模式)
- [5. Vue 重构模式](#5-vue-重构模式)
- [6. Angular 重构模式](#6-angular-重构模式)
- [7. vanilla JS / Node.js 重构模式](#7-vanilla-js--nodejs-重构模式)
- [8. 自动化重构（ast-grep / jscodeshift）](#8-自动化重构ast-grep--jscodeshift)
- [9. LSP 安全改名与引用追踪](#9-lsp-安全改名与引用追踪)
- [10. 完整执行节奏（范围→测试→重构→验证→清理）](#10-完整执行节奏范围测试重构验证清理)

## 1. 重构总策略

重构目标是“改善结构而不改变行为”。执行时：

1. 先做影响分析，不盲改。
2. 先补测试，再改代码。
3. 小步提交，分批验证。
4. 保留可回滚路径。

## 2. 重构前影响分析

### Step 1：锁定目标模块

先列出重构目标：组件、状态层、工具函数、路由模块。

```bash
ls -la src
```

### Step 2：构建依赖图与 import 链

```bash
grep -R "from './\|from '../\|from '@/" src 2>/dev/null
```

在大型项目里，优先关注：

- 被引用次数最高的文件
- 横跨多个业务域的公共模块
- 强耦合状态管理入口

### Step 3：识别高风险改动

高风险特征：

- 全局状态结构变更（Redux store/Vuex modules/NgRx store）
- 路由协议变更（URL params、query schema）
- 公共组件 props contract 变更

## 3. 安全重构通用模式

### 3.1 Extract Component

做法：

1. 把复杂 JSX/Template 片段抽离成独立组件。
2. 仅通过 props 输入，不隐式依赖父作用域。
3. 为新组件补充最小单测。

### 3.2 Extract Hook / Composable

React：抽 `useXxx`；Vue：抽 `useXxx` composable。

准则：

- 把副作用与状态逻辑从视图层分离。
- 保留输入/输出接口稳定。

### 3.3 Extract Util

将纯函数逻辑下沉到 `utils/`，并补充单测。

```ts
// before
const finalPrice = amount * (1 - discount) + shipping

// after
import { calcFinalPrice } from '@/utils/price'
const finalPrice = calcFinalPrice(amount, discount, shipping)
```

## 4. React 重构模式

### 4.1 Class Component → Hooks

迁移步骤：

1. 把 `state` 改为 `useState`。
2. 把生命周期改为 `useEffect`。
3. 把实例方法改为闭包函数。

```tsx
// before
class Counter extends React.Component {
  state = { count: 0 }
  componentDidMount() { document.title = String(this.state.count) }
  render() { return <button onClick={() => this.setState({ count: this.state.count + 1 })}>{this.state.count}</button> }
}

// after
function Counter() {
  const [count, setCount] = React.useState(0)
  React.useEffect(() => { document.title = String(count) }, [count])
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

### 4.2 HOC → Hooks

把 `withAuth(Component)` 逐步替换为 `useAuth()`，减少组件树嵌套。

### 4.3 Prop Drilling → Context / Zustand

对跨层级共享状态，优先：

1. 中小规模：React Context + reducer
2. 大规模频繁更新：Zustand/Redux Toolkit

## 5. Vue 重构模式

### 5.1 Options API → Composition API

迁移步骤：

1. `data` → `ref/reactive`
2. `computed` → `computed`
3. `methods` → 普通函数
4. 生命周期迁移到 `onMounted/onUnmounted`

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
const count = ref(0)
const double = computed(() => count.value * 2)
const inc = () => count.value++
</script>
```

### 5.2 Vuex → Pinia

做法：

1. 按模块创建 Pinia stores。
2. 把 mutation + action 合并为 action 驱动。
3. 先做适配层，逐步替换老调用。

## 6. Angular 重构模式

### 6.1 NgModule → Standalone Components

迁移步骤：

1. 在组件上启用 `standalone: true`。
2. 使用 `imports` 声明依赖组件/指令/管道。
3. 路由改为 `loadComponent`。

```ts
import { Component } from '@angular/core'

@Component({
  selector: 'app-home',
  standalone: true,
  template: `<h1>Hello</h1>`
})
export class HomeComponent {}
```

## 7. vanilla JS / Node.js 重构模式

### 7.1 vanilla JS

1. 拆分大文件为模块。
2. 用 ESM import/export 统一依赖方向。
3. 把 DOM 操作与业务计算分离。

### 7.2 Node.js

1. 拆分路由层、service 层、repository 层。
2. 把副作用（IO）边界明确化。
3. 为关键 service 函数补单测。

## 8. 自动化重构（ast-grep / jscodeshift）

### 8.1 ast-grep：模式搜索

示例：查找 React 中 `console.log`：

```bash
ast-grep --pattern 'console.log($MSG)' --lang tsx src
```

示例：批量改写（先 dry-run）：

```bash
ast-grep --pattern 'console.log($MSG)' --rewrite 'logger.info($MSG)' --lang tsx src
```

### 8.2 jscodeshift：结构化迁移

安装：

```bash
npm install -D jscodeshift
```

执行转换：

```bash
npx jscodeshift -t ./codemods/transform.js src --extensions=ts,tsx,js,jsx
```

建议：

1. 先对小目录试跑。
2. 用 Git diff 审查转换结果。
3. 每轮转换后跑测试。

## 9. LSP 安全改名与引用追踪

执行重命名时，先“查引用”，再“改名”：

1. 用 find references 确定影响面。
2. 用 prepare rename 检查当前位置可改名。
3. 用 rename 执行全局重命名。

不要用文本替换代替语义改名，避免误改字符串和注释。

## 10. 完整执行节奏（范围→测试→重构→验证→清理）

### Step 1：定义范围

- 明确本轮只改哪些目录/模块。
- 列出不在本轮范围的内容。

### Step 2：创建/补齐测试

```bash
npm run test --if-present
```

若覆盖不足，先补关键路径测试（渲染、状态更新、API 交互）。

### Step 3：实施重构

- 先做低风险提取（组件/工具函数）。
- 再做架构迁移（state/router/module）。

### Step 4：验证

```bash
npm run lint --if-present
npm run typecheck --if-present
npm run test --if-present
npm run build
```

### Step 5：清理

- 删除废弃组件、废弃 store、废弃配置。
- 删除 dead code 与未使用依赖：

```bash
npm prune
```

### Step 6：输出重构报告

报告至少包括：

1. 变更范围
2. 行为等价性验证证据
3. 性能/可维护性收益
4. 后续建议（下一批重构候选）

执行重构时，始终保持“可回滚、可验证、可审计”。
