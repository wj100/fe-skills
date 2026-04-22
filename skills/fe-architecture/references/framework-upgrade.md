# 框架大版本升级迁移手册

## 目录

- [0. 统一执行模型](#0-统一执行模型)
- [1. Vue 2 → Vue 3](#1-vue-2--vue-3)
  - [1.1 迁移前审计](#11-迁移前审计)
  - [1.2 分阶段迁移步骤](#12-分阶段迁移步骤)
  - [1.3 关键 Breaking Changes 清单](#13-关键-breaking-changes-清单)
  - [1.4 Vue Router 3 → 4](#14-vue-router-3--4)
  - [1.5 Vuex → Pinia](#15-vuex--pinia)
  - [1.6 第三方依赖兼容检查](#16-第三方依赖兼容检查)
  - [1.7 回滚策略](#17-回滚策略)
- [2. React Class → Hooks](#2-react-class--hooks)
  - [2.1 迁移前审计](#21-迁移前审计)
  - [2.2 模式映射](#22-模式映射)
  - [2.3 HOC → Custom Hooks](#23-hoc--custom-hooks)
  - [2.4 Context API 现代化](#24-context-api-现代化)
  - [2.5 常见陷阱与规避](#25-常见陷阱与规避)
  - [2.6 回滚策略](#26-回滚策略)
- [3. Angular Major 升级](#3-angular-major-升级)
  - [3.1 迁移前审计](#31-迁移前审计)
  - [3.2 ng update 执行流](#32-ng-update-执行流)
  - [3.3 Module → Standalone](#33-module--standalone)
  - [3.4 RxJS 升级](#34-rxjs-升级)
  - [3.5 Angular Material 同步](#35-angular-material-同步)
  - [3.6 回滚策略](#36-回滚策略)

## 0. 统一执行模型

1. 建立升级分支：

```bash
git checkout -b chore/framework-migration
```

2. 固化基线并打 tag：

```bash
npm ci
npm run lint
npm run test
npm run build
git tag before-framework-migration
```

3. 每个子阶段都执行验证：
   - `npm run lint`
   - `npm run test`
   - `npm run build`
   - 核心页面冒烟测试

4. 每个子阶段都独立提交，禁止跨阶段混改。

---

## 1. Vue 2 → Vue 3

### 1.1 迁移前审计

1. 收集现状：

```bash
npm ls vue vue-router vuex
```

2. 运行 `vue-migration-helper` 审计（若团队使用该工具）：

```bash
npx vue-migration-helper src
```

3. 运行 codemod 预检查：

```bash
npx vue-codemod --help
```

4. 建立问题清单：
   - 过滤器 `filters`
   - `.native` 事件修饰符
   - `v-model` 语义变更
   - `$listeners` / `$scopedSlots`
   - `this.$set` / `this.$delete`

### 1.2 分阶段迁移步骤

#### 阶段 A：引入兼容构建（@vue/compat）

1. 安装依赖：

```bash
npm install vue@^3 @vue/compat --save
npm install @vue/compiler-sfc@^3 --save-dev
```

2. 以 Vite 为例配置 alias：

```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  resolve: {
    alias: {
      vue: '@vue/compat'
    }
  },
  plugins: [vue()]
})
```

3. 打开 compat 提示：

```ts
// main.ts
import { createApp, configureCompat } from 'vue'
import App from './App.vue'

configureCompat({ MODE: 2 })
createApp(App).mount('#app')
```

4. 执行验证并记录警告数量，按模块逐步消减。

#### 阶段 B：入口与全局 API 替换

1. `new Vue()` 改为 `createApp()`。
2. 全局 API 改为 app 级 API：

```ts
const app = createApp(App)
app.component('BaseButton', BaseButton)
app.use(router)
app.use(store)
app.mount('#app')
```

3. 验证 SSR/CSR 入口是否一致。

#### 阶段 C：组件迁移到 Composition API

1. 先迁移高复用组件，再迁移页面容器。
2. 典型改造：

```ts
import { ref, computed, onMounted } from 'vue'

export default {
  setup() {
    const count = ref(0)
    const double = computed(() => count.value * 2)
    onMounted(() => {
      console.log('mounted')
    })
    return { count, double }
  }
}
```

3. 每迁移一个目录即跑测试并发版到预发。

### 1.3 关键 Breaking Changes 清单

1. 删除 `filters`：改为 `computed` 或 methods。
2. `v-model` 默认 prop/event 变为 `modelValue` / `update:modelValue`。
3. 删除 `$listeners`：使用 `$attrs`（并配合 `emits`）。
4. `functional` SFC 语法废弃：改普通组件或函数式 render。
5. `this.$set` / `this.$delete` 删除：直接赋值或结构重建。
6. `slot` / `scopedSlots` 统一为 `v-slot`。
7. 事件总线（Vue 实例）模式不再推荐：替换为 Pinia 或 mitt。

### 1.4 Vue Router 3 → 4

1. 升级依赖：

```bash
npm install vue-router@^4
```

2. 调整创建方式：

```ts
import { createRouter, createWebHistory } from 'vue-router'

const router = createRouter({
  history: createWebHistory(),
  routes
})
```

3. `beforeRouteEnter` 中 `next` 用法做兼容检查。
4. 动态路由命名与重定向规则逐条回归。

### 1.5 Vuex → Pinia

1. 安装 Pinia：

```bash
npm install pinia
```

2. 挂载：

```ts
import { createPinia } from 'pinia'
app.use(createPinia())
```

3. 从 `module` 迁移到 `defineStore`：

```ts
import { defineStore } from 'pinia'

export const useUserStore = defineStore('user', {
  state: () => ({ token: '' }),
  actions: {
    setToken(token: string) {
      this.token = token
    }
  }
})
```

4. 保持旧 Vuex 与新 Pinia 双轨一段时间，通过 adapter 兼容。

### 1.6 第三方依赖兼容检查

1. 生成依赖列表：

```bash
npm outdated
npm ls --depth=0
```

2. 核查重点：
   - UI 库是否提供 Vue 3 版本。
   - `vue-class-component`、`vue-property-decorator` 是否仍被使用。
   - i18n、表单、图表库是否支持 Composition API。

3. 对不兼容库建立替换优先级：阻断级 > 高风险级 > 可延后级。

### 1.7 回滚策略

1. 保留 `before-framework-migration` tag。
2. 每阶段独立 PR，出问题时仅回滚该 PR。
3. 发布时加 runtime flag：
   - `VUE3_COMPAT_ENABLED=true/false`
4. 若 P1 以上故障出现，执行：

```bash
git checkout release
git revert <migration-commit>
npm ci && npm run build
```

---

## 2. React Class → Hooks

### 2.1 迁移前审计

1. 统计 Class 组件：

```bash
grep -R "extends React.Component\|extends Component" src -n
```

2. 标记高风险组件：
   - 含 `componentDidCatch`
   - 含复杂 `setState` 回调链
   - 含多个 HOC 嵌套

3. 开启 lint 约束：

```bash
npm install eslint-plugin-react-hooks -D
```

```json
{
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

### 2.2 模式映射

1. `this.state` / `setState` → `useState`。
2. `componentDidMount` / `componentDidUpdate` / `componentWillUnmount` → `useEffect`。
3. `shouldComponentUpdate` → `React.memo` + `useMemo`。
4. 实例方法绑定 → 闭包函数 + `useCallback`。

示例：

```tsx
function Counter() {
  const [count, setCount] = useState(0)

  useEffect(() => {
    document.title = `count:${count}`
    return () => {
      // cleanup
    }
  }, [count])

  return <button onClick={() => setCount(c => c + 1)}>{count}</button>
}
```

### 2.3 HOC → Custom Hooks

1. 把共享逻辑从 HOC 抽到 hook：

```tsx
function useAuth() {
  const [user, setUser] = useState<User | null>(null)
  useEffect(() => {
    fetch('/api/me').then(r => r.json()).then(setUser)
  }, [])
  return { user }
}
```

2. 页面组件直接调用 hook，移除 HOC 包装层。
3. 校验渲染树深度和 DevTools 可读性是否改善。

### 2.4 Context API 现代化

1. 拆分大 Context，避免全量重渲染。
2. 使用 selector 模式或多个 provider。
3. 把派生值放入 `useMemo`。

### 2.5 常见陷阱与规避

1. `useEffect` 依赖缺失导致脏读。
2. stale closure 导致读取旧状态。
3. 把异步函数直接写进 effect 回调。
4. 在循环/条件中调用 hooks。

规避模板：

```tsx
useEffect(() => {
  let active = true
  ;(async () => {
    const data = await query()
    if (active) setData(data)
  })()
  return () => {
    active = false
  }
}, [query])
```

### 2.6 回滚策略

1. 按组件批次迁移，每批独立提交。
2. 保留 class 版本文件一周期（`Component.legacy.tsx`）。
3. 通过开关切换：

```tsx
export default process.env.REACT_HOOKS_ENABLED ? NewComp : LegacyComp
```

4. 若线上回归失败，立即切回 legacy 并回滚对应提交。

---

## 3. Angular Major 升级

### 3.1 迁移前审计

1. 检查当前版本：

```bash
ng version
```

2. 查看官方迁移路径：
   - 打开 `https://update.angular.io/` 选择 from/to 版本。

3. 校验 Node.js 与 TypeScript 版本兼容矩阵。

### 3.2 ng update 执行流

1. 先更新 CLI 和 Core：

```bash
ng update @angular/core @angular/cli
```

2. 再更新生态包：

```bash
ng update @angular/cdk @angular/material
```

3. 多个 major 跨越时按“逐 major”执行，不要跨级直跳。
4. 每次 `ng update` 后立即执行：

```bash
npm run test
npm run build
ng lint
```

### 3.3 Module → Standalone

1. 使用官方 schematic：

```bash
ng generate @angular/core:standalone
```

2. 迁移路由入口：

```ts
bootstrapApplication(AppComponent, {
  providers: [provideRouter(routes)]
})
```

3. 把 NgModule providers 下沉到 `bootstrapApplication` 或 route providers。
4. 对懒加载路由做逐个回归。

### 3.4 RxJS 升级

1. 升级并执行迁移：

```bash
npm install rxjs@^7
```

2. 替换废弃 API，统一 pipeable operators。
3. 检查 `toPromise`，替换为 `firstValueFrom/lastValueFrom`。

### 3.5 Angular Material 同步

1. 同步 major：

```bash
ng update @angular/material
```

2. 执行主题与 typography 回归。
3. 检查 overlay、dialog、form-field 在新版本样式差异。

### 3.6 回滚策略

1. 每个 major 升级独立分支：`upgrade/ng16-to-ng17`。
2. 每次成功升级后打 tag：`after-ng17-upgrade`。
3. 发布时保留上一版容器镜像，故障时一键回切。
4. 若需代码回退：

```bash
git checkout main
git revert <angular-upgrade-commit>
```

## 收尾检查

1. 清理过期 polyfills 与弃用配置。
2. 更新 CI Node 版本与缓存 key。
3. 更新工程文档：启动命令、调试方式、兼容说明。
4. 输出迁移报告：变更清单、风险闭环、未完成项。
