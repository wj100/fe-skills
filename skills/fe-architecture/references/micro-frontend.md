# 微前端架构迁移手册

## 目录

- [0. 目标与边界](#0-目标与边界)
- [1. 方案选型决策矩阵](#1-方案选型决策矩阵)
- [2. qiankun 方案（重点）](#2-qiankun-方案重点)
  - [2.1 迁移前审计](#21-迁移前审计)
  - [2.2 主应用配置](#22-主应用配置)
  - [2.3 子应用接入（Vue/React/Angular）](#23-子应用接入vuereactangular)
  - [2.4 沙箱隔离机制](#24-沙箱隔离机制)
  - [2.5 应用间通信](#25-应用间通信)
  - [2.6 样式隔离方案](#26-样式隔离方案)
  - [2.7 路由管理](#27-路由管理)
  - [2.8 部署策略](#28-部署策略)
  - [2.9 回滚策略](#29-回滚策略)
- [3. Module Federation 方案](#3-module-federation-方案)
  - [3.1 Webpack 5 配置](#31-webpack-5-配置)
  - [3.2 Vite Federation 配置](#32-vite-federation-配置)
  - [3.3 共享依赖管理](#33-共享依赖管理)
  - [3.4 动态远程加载](#34-动态远程加载)
  - [3.5 回滚策略](#35-回滚策略)
- [4. 常见挑战与治理方案](#4-常见挑战与治理方案)

## 0. 目标与边界

1. 定义迁移目标：独立发布、团队解耦、故障隔离。
2. 划分拆分维度：按业务域，不按技术层拆分。
3. 约束共享面：只共享协议，不共享实现。
4. 设定验收指标：
   - 单子应用独立构建与独立发布成功。
   - 主应用故障不拖垮其他子应用。
   - 首屏性能退化不超过既定阈值。

## 1. 方案选型决策矩阵

| 方案 | 学习成本 | 工程侵入 | 隔离能力 | 生态成熟度 | 典型适用场景 |
|---|---:|---:|---:|---:|---|
| qiankun | 低-中 | 中 | 中-高（JS 沙箱 + 样式隔离） | 高（国内案例多） | 中国市场中后台、渐进改造 |
| Module Federation | 中 | 中-高（构建深度绑定） | 中（偏模块共享） | 高 | 同构技术栈、组件级共享 |
| single-spa | 中-高 | 中 | 中 | 中 | 多框架混合、平台治理成熟团队 |
| wujie | 中 | 低-中 | 高（iframe + 保活机制） | 中 | 强隔离诉求、历史系统包袱重 |
| micro-app | 低-中 | 低 | 中-高 | 中 | 快速落地、低改造成本 |

执行建议：

1. 优先选 qiankun 作为主路径。
2. 若强依赖构建期共享与远程模块能力，补充 Module Federation。
3. 对高隔离业务域（金融、高安全）评估 wujie。

## 2. qiankun 方案（重点）

### 2.1 迁移前审计

1. 盘点应用清单与技术栈：Vue/React/Angular 版本、路由模式、公共依赖。
2. 输出主子应用映射：
   - 主应用：统一导航、权限、布局、埋点。
   - 子应用：各业务域页面与 API 调用。
3. 定义统一约定：
   - 路由基座：`/app-a/*`、`/app-b/*`
   - 通信协议：`{ type, payload, source, traceId }`
   - 生命周期钩子规范。

### 2.2 主应用配置

1. 安装：

```bash
npm install qiankun
```

2. 注册子应用：

```ts
// main.ts
import { registerMicroApps, start } from 'qiankun'

registerMicroApps([
  {
    name: 'app-vue',
    entry: '//localhost:7101',
    container: '#subapp-container',
    activeRule: '/app-vue'
  },
  {
    name: 'app-react',
    entry: '//localhost:7102',
    container: '#subapp-container',
    activeRule: '/app-react'
  }
])

start({
  prefetch: 'all',
  sandbox: { strictStyleIsolation: false }
})
```

3. 配置主应用容器路由，保留统一 404/鉴权拦截。

### 2.3 子应用接入（Vue/React/Angular）

#### Vue 子应用

```ts
// src/main.ts
let app: any

export async function bootstrap() {}
export async function mount(props: any) {
  const { createApp } = await import('vue')
  const App = (await import('./App.vue')).default
  app = createApp(App)
  app.mount(props.container ? props.container.querySelector('#app') : '#app')
}
export async function unmount() {
  app?.unmount()
  app = null
}
```

#### React 子应用

```tsx
import ReactDOM from 'react-dom/client'
import App from './App'

let root: ReactDOM.Root | null = null

export async function mount(props: any) {
  const container = props.container?.querySelector('#root') || document.getElementById('root')!
  root = ReactDOM.createRoot(container)
  root.render(<App />)
}

export async function unmount() {
  root?.unmount()
  root = null
}
```

#### Angular 子应用

1. 通过自定义入口桥接 Angular bootstrap 与 qiankun 生命周期。
2. 在 `mount` 中动态创建 platform，在 `unmount` 中销毁 moduleRef。
3. 避免在全局 `window` 泄露引用。

### 2.4 沙箱隔离机制

1. 启用默认 JS 沙箱，避免子应用污染 `window`。
2. 对强副作用库（地图、编辑器）包裹隔离适配层。
3. 使用 `excludeAssetFilter` 排除必须共享的脚本（谨慎）：

```ts
start({
  sandbox: true,
  excludeAssetFilter: (url) => url.includes('legacy-global-sdk')
})
```

### 2.5 应用间通信

1. 使用 `initGlobalState`：

```ts
import { initGlobalState } from 'qiankun'

const actions = initGlobalState({ token: '', theme: 'light' })
actions.onGlobalStateChange((state, prev) => {
  console.log(state, prev)
})
actions.setGlobalState({ theme: 'dark' })
```

2. 定义状态白名单，禁止大对象跨应用传输。
3. 对跨应用事件记录 traceId，便于排障。

### 2.6 样式隔离方案

1. 优先在子应用使用 CSS Modules / BEM 命名空间。
2. 若冲突严重，启用 strict style isolation（Shadow DOM，需评估 UI 库兼容性）。
3. 对全局样式（reset、font）只允许主应用注入。

### 2.7 路由管理

1. 采用主应用主路由 + 子应用子路由模式。
2. 子应用路由使用 `base`：

```ts
createWebHistory('/app-vue')
```

3. 刷新直达时确保网关回源到主应用 `index.html`。

### 2.8 部署策略

1. 产物分离：主应用与子应用独立仓库/独立流水线。
2. 入口地址通过配置中心下发，避免硬编码。
3. 灰度策略：先灰度主应用路由分流，再灰度子应用版本。

示例配置：

```json
{
  "app-vue": "https://cdn.example.com/app-vue/1.3.0/",
  "app-react": "https://cdn.example.com/app-react/2.1.4/"
}
```

### 2.9 回滚策略

1. 主应用保留上一版入口清单。
2. 子应用发布遵循版本目录不可覆盖（immutable）。
3. 故障时优先回滚配置映射，不先回滚代码：
   - 把 `app-vue` 映射从 `1.3.0` 切回 `1.2.7`。
4. 主应用提供熔断开关：

```ts
if (window.__DISABLE_APP_VUE__) {
  // 跳过注册
}
```

## 3. Module Federation 方案

### 3.1 Webpack 5 配置

1. Host 配置：

```js
// webpack.config.js
const ModuleFederationPlugin = require('webpack/lib/container/ModuleFederationPlugin')

plugins: [
  new ModuleFederationPlugin({
    name: 'host',
    remotes: {
      app1: 'app1@https://cdn.example.com/app1/remoteEntry.js'
    },
    shared: {
      react: { singleton: true, requiredVersion: '^18.2.0' },
      'react-dom': { singleton: true, requiredVersion: '^18.2.0' }
    }
  })
]
```

2. Remote 配置：

```js
new ModuleFederationPlugin({
  name: 'app1',
  filename: 'remoteEntry.js',
  exposes: {
    './UserPanel': './src/UserPanel'
  },
  shared: {
    react: { singleton: true },
    'react-dom': { singleton: true }
  }
})
```

### 3.2 Vite Federation 配置

1. 安装插件：

```bash
npm install @originjs/vite-plugin-federation -D
```

2. 配置：

```ts
import federation from '@originjs/vite-plugin-federation'

federation({
  name: 'host',
  remotes: {
    app1: 'https://cdn.example.com/app1/assets/remoteEntry.js'
  },
  shared: ['vue']
})
```

### 3.3 共享依赖管理

1. 把核心框架声明为 `singleton`。
2. 指定 `requiredVersion`，避免隐式升级。
3. 对非核心库允许重复加载，换取解耦。

### 3.4 动态远程加载

1. 通过远程配置中心拉取 remote URL。
2. 注入 script 前做 SRI 校验与超时控制。
3. 加载失败时降级到本地兜底模块。

### 3.5 回滚策略

1. 保留上一版 `remoteEntry.js`。
2. 通过配置中心切回上一版 remote URL。
3. 加载失败率超阈值自动熔断。

## 4. 常见挑战与治理方案

1. CSS 冲突：
   - 统一前缀 + CSS Modules。
   - 避免跨子应用注入全局 reset。

2. 全局状态共享：
   - 只共享身份、主题、租户等最小状态。
   - 复杂业务状态留在子应用本地。

3. 公共依赖版本冲突：
   - 建立依赖白名单（必须单例）与灰名单（可多版本）。
   - 在 CI 阶段扫描 lockfile 并阻断高风险冲突。

4. 性能开销与优化：
   - 预加载高频子应用静态资源。
   - 低频子应用按需加载。
   - 对主应用首屏禁用全量 prefetch。

5. 观测与排障：
   - 主子应用统一 traceId。
   - 埋点字段增加 `appName`、`appVersion`。
   - 错误平台按应用维度聚合并设置告警阈值。

6. 安全治理：
   - 限制 remote 源站白名单。
   - 启用 CSP 与 SRI。
   - 对跨应用通信 payload 执行 schema 校验。
