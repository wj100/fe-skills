# 错误监控集成

## 目录

- [Sentry 集成](#sentry-集成)
- [自研监控 SDK](#自研监控-sdk)
- [Error Boundary 模式](#error-boundary-模式)
- [日志规范](#日志规范)

## Sentry 集成

### 安装

```bash
# React
pnpm add @sentry/react

# Vue
pnpm add @sentry/vue

# Angular
pnpm add @sentry/angular

# 原生 JS
pnpm add @sentry/browser
```

### 初始化

**React:**

```tsx
// src/main.tsx
import * as Sentry from '@sentry/react';

Sentry.init({
  dsn: 'https://xxx@sentry.io/xxx',
  environment: import.meta.env.MODE,        // development / production
  release: `my-app@${__APP_VERSION__}`,     // 关联 source map
  integrations: [
    Sentry.browserTracingIntegration(),     // 性能监控
    Sentry.replayIntegration(),             // 会话回放
  ],
  tracesSampleRate: 0.1,                    // 采样 10% 请求
  replaysSessionSampleRate: 0.01,           // 1% 会话回放
  replaysOnErrorSampleRate: 1.0,            // 错误时 100% 回放
});
```

**Vue:**

```ts
// src/main.ts
import * as Sentry from '@sentry/vue';

Sentry.init({
  app,
  dsn: 'https://xxx@sentry.io/xxx',
  environment: import.meta.env.MODE,
  integrations: [
    Sentry.browserTracingIntegration({ router }),  // 传入 Vue Router
  ],
  tracesSampleRate: 0.1,
  trackComponents: true,                           // 追踪组件渲染
});
```

**Angular:**

```typescript
// app.config.ts
import * as Sentry from '@sentry/angular';

Sentry.init({
  dsn: 'https://xxx@sentry.io/xxx',
  integrations: [Sentry.browserTracingIntegration()],
  tracesSampleRate: 0.1,
});

// providers 中注册
export const appConfig: ApplicationConfig = {
  providers: [
    { provide: ErrorHandler, useValue: Sentry.createErrorHandler() },
    { provide: Sentry.TraceService, deps: [Router] },
    { provide: APP_INITIALIZER, useFactory: () => () => {}, deps: [Sentry.TraceService], multi: true },
  ],
};
```

### Source Map 上传

```bash
pnpm add -D @sentry/vite-plugin  # Vite
pnpm add -D @sentry/webpack-plugin  # Webpack
```

**Vite 配置：**

```ts
// vite.config.ts
import { sentryVitePlugin } from '@sentry/vite-plugin';

export default defineConfig({
  build: { sourcemap: true },   // 必须开启 source map
  plugins: [
    sentryVitePlugin({
      org: 'your-org',
      project: 'your-project',
      authToken: process.env.SENTRY_AUTH_TOKEN,
    }),
  ],
});
```

**CI 中上传（推荐）：**

```yaml
# GitHub Actions
- name: Upload source maps
  run: npx @sentry/cli sourcemaps upload --org your-org --project your-project dist/
  env:
    SENTRY_AUTH_TOKEN: ${{ secrets.SENTRY_AUTH_TOKEN }}
```

### 自定义错误上报

```ts
// 手动捕获错误
Sentry.captureException(error);

// 附加上下文
Sentry.captureException(error, {
  tags: { module: 'checkout', severity: 'critical' },
  extra: { orderId: '12345', userId: 'abc' },
});

// 设置用户信息
Sentry.setUser({ id: 'user-123', email: 'user@example.com' });

// 面包屑（调试线索）
Sentry.addBreadcrumb({
  category: 'user-action',
  message: '点击了提交按钮',
  level: 'info',
});
```

## 自研监控 SDK

### 全局错误捕获

```ts
// monitor.ts — 轻量监控 SDK
interface ErrorReport {
  type: 'js' | 'resource' | 'promise' | 'request';
  message: string;
  stack?: string;
  url: string;
  timestamp: number;
  userAgent: string;
  extra?: Record<string, unknown>;
}

const reportQueue: ErrorReport[] = [];

// JS 运行时错误
window.onerror = (message, source, lineno, colno, error) => {
  reportQueue.push({
    type: 'js',
    message: String(message),
    stack: error?.stack,
    url: source || location.href,
    timestamp: Date.now(),
    userAgent: navigator.userAgent,
    extra: { lineno, colno },
  });
  flushReports();
};

// Promise 未捕获拒绝
window.addEventListener('unhandledrejection', (event) => {
  reportQueue.push({
    type: 'promise',
    message: event.reason?.message || String(event.reason),
    stack: event.reason?.stack,
    url: location.href,
    timestamp: Date.now(),
    userAgent: navigator.userAgent,
  });
  flushReports();
});

// 资源加载失败
window.addEventListener('error', (event) => {
  const target = event.target as HTMLElement;
  if (target?.tagName && ['SCRIPT', 'LINK', 'IMG'].includes(target.tagName)) {
    reportQueue.push({
      type: 'resource',
      message: `${target.tagName} 加载失败: ${(target as HTMLScriptElement).src || (target as HTMLLinkElement).href}`,
      url: location.href,
      timestamp: Date.now(),
      userAgent: navigator.userAgent,
    });
    flushReports();
  }
}, true);  // 捕获阶段
```

### 请求错误拦截

```ts
// Fetch 拦截
const originalFetch = window.fetch;
window.fetch = async (...args) => {
  const startTime = Date.now();
  try {
    const response = await originalFetch(...args);
    if (!response.ok) {
      reportQueue.push({
        type: 'request',
        message: `HTTP ${response.status}: ${args[0]}`,
        url: String(args[0]),
        timestamp: startTime,
        userAgent: navigator.userAgent,
        extra: { duration: Date.now() - startTime, status: response.status },
      });
      flushReports();
    }
    return response;
  } catch (error) {
    reportQueue.push({
      type: 'request',
      message: `Fetch 失败: ${args[0]}`,
      url: String(args[0]),
      timestamp: startTime,
      userAgent: navigator.userAgent,
      extra: { error: (error as Error).message },
    });
    flushReports();
    throw error;
  }
};
```

### 上报策略

```ts
// 使用 Beacon API 可靠上报（页面卸载时也能发送）
function flushReports() {
  if (reportQueue.length === 0) return;

  const reports = reportQueue.splice(0, reportQueue.length);
  const blob = new Blob([JSON.stringify(reports)], { type: 'application/json' });

  // Beacon API — 不会被页面卸载中断
  if (navigator.sendBeacon) {
    navigator.sendBeacon('/api/monitor/report', blob);
  } else {
    fetch('/api/monitor/report', { method: 'POST', body: blob, keepalive: true });
  }
}

// 定时批量上报
setInterval(flushReports, 10000);

// 页面卸载前上报
window.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden') flushReports();
});
```

### 采样策略

```ts
// 高流量站点需要采样
const SAMPLE_RATE = 0.1; // 10%

function shouldReport(): boolean {
  return Math.random() < SAMPLE_RATE;
}

// 在上报前检查
if (shouldReport()) {
  reportQueue.push(/* ... */);
}
```

## Error Boundary 模式

### React

```tsx
import { Component, ErrorInfo, ReactNode } from 'react';

interface Props { children: ReactNode; fallback?: ReactNode; }
interface State { hasError: boolean; error?: Error; }

class ErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    // 上报到监控系统
    Sentry.captureException(error, { extra: { componentStack: errorInfo.componentStack } });
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback || (
        <div style={{ padding: '20px', textAlign: 'center' }}>
          <h2>页面出错了</h2>
          <button onClick={() => this.setState({ hasError: false })}>重试</button>
        </div>
      );
    }
    return this.props.children;
  }
}

// 使用
<ErrorBoundary fallback={<ErrorPage />}>
  <App />
</ErrorBoundary>
```

### Vue

```ts
// 全局错误处理
app.config.errorHandler = (err, instance, info) => {
  Sentry.captureException(err, { extra: { info, component: instance?.$options.name } });
};

// 组件级 — errorCaptured 钩子
defineComponent({
  errorCaptured(err, instance, info) {
    this.hasError = true;
    Sentry.captureException(err);
    return false; // 阻止继续传播
  },
});
```

### Angular

```typescript
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  handleError(error: unknown): void {
    Sentry.captureException(error);
    console.error('未捕获错误:', error);
  }
}

// 注册
{ provide: ErrorHandler, useClass: GlobalErrorHandler }
```

## 日志规范

### 结构化日志格式

```ts
enum LogLevel { DEBUG = 0, INFO = 1, WARN = 2, ERROR = 3 }

function log(level: LogLevel, message: string, data?: Record<string, unknown>) {
  const entry = {
    level: LogLevel[level],
    message,
    timestamp: new Date().toISOString(),
    url: location.href,
    ...data,
  };

  if (level >= LogLevel.WARN) {
    flushReports(); // 警告及以上立即上报
  }

  if (import.meta.env.DEV) {
    console[level >= LogLevel.ERROR ? 'error' : level >= LogLevel.WARN ? 'warn' : 'log'](entry);
  }
}
```

### 敏感数据过滤

```ts
const SENSITIVE_KEYS = ['password', 'token', 'secret', 'authorization', 'cookie', 'credit'];

function sanitizeData(data: Record<string, unknown>): Record<string, unknown> {
  const result = { ...data };
  for (const key of Object.keys(result)) {
    if (SENSITIVE_KEYS.some(s => key.toLowerCase().includes(s))) {
      result[key] = '[REDACTED]';
    }
  }
  return result;
}
```

### 验证清单

- [ ] Sentry（或自研 SDK）已初始化并能捕获错误
- [ ] Source map 已上传，堆栈可读
- [ ] Error Boundary 已配置，页面崩溃有友好提示
- [ ] 请求错误被拦截上报
- [ ] 敏感数据已从日志中过滤
- [ ] 采样率已根据流量合理配置
