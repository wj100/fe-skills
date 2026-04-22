# 前端安全加固

## 目录

- [XSS 防御](#xss-防御)
- [CSP 内容安全策略](#csp-内容安全策略)
- [依赖安全审计](#依赖安全审计)
- [CSRF 防护](#csrf-防护)
- [其他安全实践](#其他安全实践)

## XSS 防御

### 框架内置防护

**React** — 默认转义所有 JSX 插值：

```tsx
// ✅ 安全 — 自动转义
<div>{userInput}</div>

// ❌ 危险 — 绕过转义，必须审查
<div dangerouslySetInnerHTML={{ __html: userInput }} />

// ✅ 安全使用 dangerouslySetInnerHTML — 先消毒
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

**Vue** — 模板插值默认转义：

```vue
<!-- ✅ 安全 — 自动转义 -->
<div>{{ userInput }}</div>

<!-- ❌ 危险 — 原始 HTML 渲染 -->
<div v-html="userInput"></div>

<!-- ✅ 安全使用 v-html -->
<div v-html="sanitize(userInput)"></div>

<script setup>
import DOMPurify from 'dompurify';
const sanitize = (html) => DOMPurify.sanitize(html);
</script>
```

**Angular** — 内置 DomSanitizer：

```typescript
import { DomSanitizer, SafeHtml } from '@angular/platform-browser';

@Component({ template: '<div [innerHTML]="safeHtml"></div>' })
export class MyComponent {
  safeHtml: SafeHtml;
  constructor(private sanitizer: DomSanitizer) {
    // Angular 自动消毒 innerHTML 绑定
    // 如需绕过（慎用）：
    this.safeHtml = this.sanitizer.bypassSecurityTrustHtml(trustedHtml);
  }
}
```

### DOMPurify 集成

```bash
pnpm add dompurify
pnpm add -D @types/dompurify  # TypeScript
```

```ts
import DOMPurify from 'dompurify';

// 基础消毒
const clean = DOMPurify.sanitize(dirty);

// 严格模式 — 只允许特定标签
const clean = DOMPurify.sanitize(dirty, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'br'],
  ALLOWED_ATTR: ['href', 'target'],
});

// 禁止所有 HTML（纯文本）
const clean = DOMPurify.sanitize(dirty, { ALLOWED_TAGS: [] });
```

### URL 注入防护

```ts
// ❌ 危险 — javascript: 协议
<a href={userUrl}>链接</a>

// ✅ 安全 — 校验 URL 协议
function safeUrl(url: string): string {
  try {
    const parsed = new URL(url);
    if (['http:', 'https:', 'mailto:'].includes(parsed.protocol)) {
      return url;
    }
  } catch {}
  return '#';
}
<a href={safeUrl(userUrl)}>链接</a>
```

## CSP 内容安全策略

### 基础配置

Nginx 配置：

```nginx
add_header Content-Security-Policy "
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM}';
  style-src 'self' 'unsafe-inline';
  img-src 'self' data: https:;
  font-src 'self' https://fonts.gstatic.com;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';
" always;
```

Meta 标签方式（备选）：

```html
<meta http-equiv="Content-Security-Policy"
  content="default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline';">
```

### Nonce 策略（推荐）

服务端生成随机 nonce：

```js
// Node.js / Express
import crypto from 'crypto';

app.use((req, res, next) => {
  res.locals.nonce = crypto.randomBytes(16).toString('base64');
  res.setHeader('Content-Security-Policy',
    `script-src 'self' 'nonce-${res.locals.nonce}'`);
  next();
});
```

HTML 中使用：

```html
<script nonce="<%= nonce %>">
  // 内联脚本
</script>
```

### Report-Only 模式（渐进上线）

```nginx
# 先用 report-only 收集违规，不阻断
add_header Content-Security-Policy-Report-Only "
  default-src 'self';
  report-uri /csp-report;
";
```

### 框架特殊处理

**Vue/React 内联样式问题：**

Vue 和 React 的 CSS-in-JS 或动态样式可能需要 `'unsafe-inline'` for styles。推荐使用 nonce 替代：

```js
// Next.js — 内置 nonce 支持
// next.config.js 中配置 CSP headers
```

## 依赖安全审计

### 日常审计

```bash
# npm/pnpm 内置审计
pnpm audit
pnpm audit --production   # 只审计生产依赖
npm audit fix              # 自动修复（谨慎使用）

# 查看详细漏洞信息
pnpm audit --json | jq '.advisories'
```

### 自动化工具

**Renovate（推荐）：**

```json
// renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "vulnerabilityAlerts": { "enabled": true },
  "packageRules": [
    {
      "matchUpdateTypes": ["patch"],
      "automerge": true
    }
  ]
}
```

**GitHub Dependabot：**

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

### Lock 文件安全

```bash
# 验证 lock 文件完整性
npm ci                    # 严格按 lock 文件安装
pnpm install --frozen-lockfile  # pnpm 等效

# 检查 lock 文件是否被篡改
npx lockfile-lint --path pnpm-lock.yaml --type pnpm --allowed-hosts npm
```

## CSRF 防护

### SPA 中的 CSRF Token

```ts
// 从 meta 标签或 cookie 获取 token
function getCsrfToken(): string {
  return document.querySelector('meta[name="csrf-token"]')?.getAttribute('content') || '';
}

// Axios 全局配置
import axios from 'axios';
axios.defaults.headers.common['X-CSRF-TOKEN'] = getCsrfToken();

// 或使用拦截器
axios.interceptors.request.use(config => {
  config.headers['X-CSRF-TOKEN'] = getCsrfToken();
  return config;
});
```

### Cookie 安全配置

```
Set-Cookie: session=xxx; HttpOnly; Secure; SameSite=Strict; Path=/
```

| 属性 | 作用 |
|------|------|
| HttpOnly | JS 无法读取，防 XSS 窃取 |
| Secure | 仅 HTTPS 传输 |
| SameSite=Strict | 禁止跨站请求携带 |
| SameSite=Lax | 允许顶级导航携带（推荐） |

## 其他安全实践

### 敏感数据防泄漏

```bash
# .env 文件不能进入前端 bundle
# Vite: 只有 VITE_ 前缀的变量会暴露
VITE_API_URL=https://api.example.com    # ✅ 会暴露到客户端
DATABASE_URL=postgres://...              # ✅ 不会暴露

# React (CRA): 只有 REACT_APP_ 前缀
# Angular: 使用 environment.ts
```

### Subresource Integrity (SRI)

```html
<!-- CDN 资源加 integrity 校验 -->
<script
  src="https://cdn.example.com/lib.js"
  integrity="sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8w"
  crossorigin="anonymous">
</script>
```

生成 integrity hash：

```bash
cat lib.js | openssl dgst -sha384 -binary | openssl base64 -A
```

### 防点击劫持

```nginx
# Nginx
add_header X-Frame-Options "DENY" always;
# 或用 CSP
add_header Content-Security-Policy "frame-ancestors 'none';" always;
```

### 验证清单

- [ ] 所有用户输入在渲染前经过消毒（DOMPurify）
- [ ] CSP 已配置且 report-only 测试通过
- [ ] `pnpm audit` 无高危/严重漏洞
- [ ] Cookie 设置了 HttpOnly + Secure + SameSite
- [ ] 无敏感信息泄漏到前端 bundle
- [ ] CDN 资源使用 SRI 校验
- [ ] 自动化依赖更新已配置（Renovate/Dependabot）
