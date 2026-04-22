# TypeScript 迁移手册

## 目录

- [0. 迁移总策略](#0-迁移总策略)
- [1. Phase 1：基础设施搭建](#1-phase-1基础设施搭建)
  - [1.1 初始化 TypeScript](#11-初始化-typescript)
  - [1.2 tsconfig 基线配置](#12-tsconfig-基线配置)
  - [1.3 各框架推荐配置](#13-各框架推荐配置)
  - [1.4 构建工具支持](#14-构建工具支持)
  - [1.5 回滚策略](#15-回滚策略)
- [2. Phase 2：渐进式迁移](#2-phase-2渐进式迁移)
  - [2.1 文件迁移优先级](#21-文件迁移优先级)
  - [2.2 js → ts 转换流程](#22-js--ts-转换流程)
  - [2.3 .d.ts 声明补齐](#23-dts-声明补齐)
  - [2.4 any 消除路线](#24-any-消除路线)
  - [2.5 回滚策略](#25-回滚策略)
- [3. Phase 3：严格模式推进](#3-phase-3严格模式推进)
  - [3.1 strict 子项逐步开启](#31-strict-子项逐步开启)
  - [3.2 strictNullChecks 迁移模式](#32-strictnullchecks-迁移模式)
  - [3.3 strictFunctionTypes 影响](#33-strictfunctiontypes-影响)
  - [3.4 回滚策略](#34-回滚策略)
- [4. 框架专项实践](#4-框架专项实践)
  - [4.1 React](#41-react)
  - [4.2 Vue](#42-vue)
  - [4.3 Angular](#43-angular)
- [5. 常见类型模式](#5-常见类型模式)
- [6. 收尾与验收](#6-收尾与验收)

## 0. 迁移总策略

1. 保持业务连续可交付：每次迁移不阻断发布。
2. 执行“目录级迁移”，不要“仓库级一次性迁移”。
3. 优先给高复用模块建立类型，再迁移页面。
4. 使用 `allowJs: true` 作为过渡，最后关闭。
5. 类型错误必须可追踪：在 CI 中输出错误趋势。

## 1. Phase 1：基础设施搭建

### 1.1 初始化 TypeScript

```bash
npm install typescript -D
npx tsc --init
```

安装常用工具：

```bash
npm install @types/node -D
```

### 1.2 tsconfig 基线配置

```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "Bundler",
    "jsx": "react-jsx",
    "allowJs": true,
    "checkJs": false,
    "strict": false,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src", "types"]
}
```

执行基线检查：

```bash
npx tsc --noEmit
```

### 1.3 各框架推荐配置

#### React（Vite / CRA）

```json
{
  "compilerOptions": {
    "jsx": "react-jsx",
    "lib": ["DOM", "ES2022"]
  }
}
```

#### Vue 3

```json
{
  "compilerOptions": {
    "types": ["vite/client"],
    "lib": ["ES2022", "DOM"]
  }
}
```

搭配 `vue-tsc`：

```bash
npm install vue-tsc -D
npx vue-tsc --noEmit
```

#### Angular

1. 通过 Angular CLI 默认 TS 支持。
2. 在 `tsconfig.app.json`、`tsconfig.spec.json` 分层配置。
3. 升级 Angular 时同步验证 TypeScript 版本兼容。

### 1.4 构建工具支持

#### Vite

1. Vite 原生支持 TS，不需额外 loader。
2. 在 CI 增加单独类型检查：

```bash
npx tsc --noEmit
```

#### Webpack

```bash
npm install ts-loader -D
```

```js
module.exports = {
  module: {
    rules: [{ test: /\.tsx?$/, use: 'ts-loader', exclude: /node_modules/ }]
  },
  resolve: { extensions: ['.ts', '.tsx', '.js'] }
}
```

#### Angular CLI

1. 使用官方构建链，不额外引入 ts-loader。
2. 在 `angular.json` 控制 strict 模式与 template 检查。

### 1.5 回滚策略

1. `tsconfig` 改动单独提交。
2. 若构建失败，先回滚 TS 配置，再定位冲突依赖。
3. 保留 `npm run build:js` 备用脚本（迁移初期）。

## 2. Phase 2：渐进式迁移

### 2.1 文件迁移优先级

按顺序执行：

1. `utils`（纯函数，最易收敛）
2. `types`（领域模型）
3. `components`（可复用组件）
4. `pages`（页面容器）

### 2.2 js → ts 转换流程

1. 批量重命名：

```bash
git ls-files "src/**/*.js" | head -n 20
```

> 执行时按目录小批次 rename，避免一次性大变更。

2. 先改后缀为 `.ts/.tsx`，补最小可编译类型。
3. 修复隐式 `any`、未声明属性、返回值类型。
4. 每完成一批执行：

```bash
npx tsc --noEmit && npm run test
```

### 2.3 .d.ts 声明补齐

1. 对无类型依赖安装 `@types`：

```bash
npm install @types/lodash -D
```

2. 若社区无类型，创建 `types/vendor.d.ts`：

```ts
declare module 'legacy-sdk' {
  export function init(config: Record<string, unknown>): void
  export function track(event: string, payload?: Record<string, unknown>): void
}
```

3. 避免直接 `declare module '*'`，会吞掉真实错误。

### 2.4 any 消除路线

1. 定义 `AnyDebt` 台账，按目录统计数量。
2. 优先替换外部边界 any：API 响应、事件 payload、表单值。
3. 使用 `unknown` 替代 `any` 并配合类型收窄。

示例：

```ts
function parseUser(input: unknown): User {
  if (typeof input === 'object' && input !== null && 'id' in input) {
    return input as User
  }
  throw new Error('invalid user')
}
```

4. 在 ESLint 禁止新增显式 any：

```json
{
  "rules": {
    "@typescript-eslint/no-explicit-any": "error"
  }
}
```

### 2.5 回滚策略

1. 一次只迁移一个子目录。
2. 若迁移后故障，回滚该目录提交，不回滚已稳定目录。
3. 在关键路径保留 `.js` fallback 文件一周（短期）。

## 3. Phase 3：严格模式推进

### 3.1 strict 子项逐步开启

建议顺序：

1. `noImplicitAny`
2. `noImplicitThis`
3. `strictFunctionTypes`
4. `strictNullChecks`
5. `strictBindCallApply`
6. 最终 `strict: true`

阶段配置示例：

```json
{
  "compilerOptions": {
    "strict": false,
    "noImplicitAny": true,
    "strictNullChecks": false
  }
}
```

### 3.2 strictNullChecks 迁移模式

1. 显式可空：`string | null`。
2. 使用可选链与空值合并：

```ts
const city = user?.profile?.city ?? 'unknown'
```

3. 用 type guard 收窄：

```ts
function isDefined<T>(v: T | null | undefined): v is T {
  return v != null
}
```

4. 禁止滥用非空断言 `!`。

### 3.3 strictFunctionTypes 影响

1. 回调参数逆变检查更严格。
2. 事件处理器不要偷懒用宽类型。
3. 泛型函数签名需统一输入输出约束。

### 3.4 回滚策略

1. 每个 strict 子项独立提交与发布。
2. 出现阻断时，临时关闭当前子项，不关闭已稳定子项。
3. 在 CI 中记录错误数量趋势，避免回退成“静默失败”。

## 4. 框架专项实践

### 4.1 React

1. FC 类型：

```tsx
type Props = { title: string }
function Panel({ title }: Props) {
  return <h1>{title}</h1>
}
```

2. hooks 类型：

```tsx
const [list, setList] = useState<User[]>([])
const ref = useRef<HTMLDivElement | null>(null)
```

3. 事件类型：

```tsx
const onChange = (e: React.ChangeEvent<HTMLInputElement>) => {
  setKeyword(e.target.value)
}
```

### 4.2 Vue

1. `defineComponent` 与 `script setup`：

```vue
<script setup lang="ts">
import { ref } from 'vue'
const count = ref<number>(0)
</script>
```

2. `reactive` 与 `ref`：

```ts
const form = reactive<{ name: string; age: number | null }>({ name: '', age: null })
```

3. Pinia typing：

```ts
export const useAuthStore = defineStore('auth', {
  state: () => ({ token: '' as string })
})
```

### 4.3 Angular

1. 启用 strict template checking。
2. 使用 Typed Forms：

```ts
form = new FormGroup({
  name: new FormControl<string>('', { nonNullable: true })
})
```

3. 服务层接口统一定义在 `models/`，禁止在组件内内联匿名类型。

## 5. 常见类型模式

1. API 响应类型：

```ts
type ApiResp<T> = { code: number; message: string; data: T }
```

2. 分页类型：

```ts
type PageResult<T> = { list: T[]; total: number; page: number; pageSize: number }
```

3. 事件处理：

```ts
type EventMap = {
  'user:login': { userId: string }
  'theme:change': { mode: 'light' | 'dark' }
}
```

4. 第三方类型包策略：
   - 优先安装官方或 DefinitelyTyped。
   - 锁定 `@types/*` 版本，避免隐式破坏。

## 6. 收尾与验收

1. 关闭 `allowJs`：

```json
{ "compilerOptions": { "allowJs": false } }
```

2. 开启 `strict: true` 并清零类型错误。
3. 在 CI 固化：

```bash
npx tsc --noEmit
npm run lint
npm run test
```

4. 删除过渡性声明与 fallback 文件。
5. 产出迁移报告：目录进度、类型债务清零情况、遗留风险。
