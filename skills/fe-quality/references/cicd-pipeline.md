# 前端 CI/CD 流水线

## 目录

- [GitHub Actions 方案](#github-actions-方案)
- [GitLab CI 方案](#gitlab-ci-方案)
- [Lighthouse CI](#lighthouse-ci)
- [视觉回归测试](#视觉回归测试)
- [部署策略](#部署策略)

## GitHub Actions 方案

### 基础流水线

```yaml
# .github/workflows/ci.yml
name: Frontend CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node-version: [18, 20]

    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with:
          version: 9

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node-version }}
          cache: 'pnpm'

      - name: Install dependencies
        run: pnpm install --frozen-lockfile

      - name: Lint
        run: pnpm lint

      - name: Type check
        run: pnpm type-check

      - name: Unit test
        run: pnpm test -- --coverage

      - name: Build
        run: pnpm build

      - name: Upload coverage
        if: matrix.node-version == 20
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
```

### 缓存优化

```yaml
      # pnpm store 缓存（action-setup 自动处理）
      # 构建缓存（Vite/Webpack）
      - name: Cache build
        uses: actions/cache@v4
        with:
          path: |
            .next/cache
            node_modules/.vite
            .angular/cache
          key: build-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}
          restore-keys: build-${{ runner.os }}-
```

### PR 预览部署

```yaml
  preview:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    needs: ci
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm build

      # Vercel 部署
      - uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}

      # 或 Cloudflare Pages
      # - uses: cloudflare/pages-action@v1
      #   with:
      #     apiToken: ${{ secrets.CF_API_TOKEN }}
      #     accountId: ${{ secrets.CF_ACCOUNT_ID }}
      #     projectName: my-project
      #     directory: dist
```

## GitLab CI 方案

```yaml
# .gitlab-ci.yml
image: node:20-alpine

stages:
  - install
  - check
  - build
  - deploy

variables:
  PNPM_VERSION: "9"

cache:
  key:
    files: [pnpm-lock.yaml]
  paths:
    - node_modules/
    - .pnpm-store/

before_script:
  - corepack enable
  - corepack prepare pnpm@${PNPM_VERSION} --activate
  - pnpm config set store-dir .pnpm-store

install:
  stage: install
  script:
    - pnpm install --frozen-lockfile

lint:
  stage: check
  script:
    - pnpm lint
  allow_failure: false

type-check:
  stage: check
  script:
    - pnpm type-check

test:
  stage: check
  script:
    - pnpm test -- --coverage
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage/cobertura-coverage.xml

build:
  stage: build
  script:
    - pnpm build
  artifacts:
    paths: [dist/]
    expire_in: 1 week

deploy-preview:
  stage: deploy
  script:
    - echo "Deploy preview for MR $CI_MERGE_REQUEST_IID"
  environment:
    name: review/$CI_COMMIT_REF_SLUG
    url: https://$CI_COMMIT_REF_SLUG.preview.example.com
  rules:
    - if: $CI_MERGE_REQUEST_IID

deploy-prod:
  stage: deploy
  script:
    - echo "Deploy to production"
  environment:
    name: production
    url: https://example.com
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
```

## Lighthouse CI

### 安装配置

```bash
pnpm add -D @lhci/cli
```

### 配置文件

```js
// lighthouserc.js
module.exports = {
  ci: {
    collect: {
      url: ['http://localhost:3000', 'http://localhost:3000/about'],
      startServerCommand: 'pnpm preview',
      startServerReadyPattern: 'Local',
      numberOfRuns: 3,
    },
    assert: {
      assertions: {
        'categories:performance': ['error', { minScore: 0.8 }],
        'categories:accessibility': ['warn', { minScore: 0.9 }],
        'categories:best-practices': ['warn', { minScore: 0.9 }],
        'categories:seo': ['warn', { minScore: 0.9 }],
        'first-contentful-paint': ['warn', { maxNumericValue: 2000 }],
        'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
        'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
        'total-blocking-time': ['warn', { maxNumericValue: 300 }],
      },
    },
    upload: {
      target: 'temporary-public-storage',  // 免费临时存储
    },
  },
};
```

### CI 集成

```yaml
# GitHub Actions
  lighthouse:
    needs: ci
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20, cache: 'pnpm' }
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
      - name: Lighthouse CI
        run: npx @lhci/cli autorun
        env:
          LHCI_GITHUB_APP_TOKEN: ${{ secrets.LHCI_GITHUB_APP_TOKEN }}
```

## 视觉回归测试

### Playwright 截图对比

```ts
// tests/visual.spec.ts
import { test, expect } from '@playwright/test';

test('首页视觉一致性', async ({ page }) => {
  await page.goto('/');
  await page.waitForLoadState('networkidle');
  await expect(page).toHaveScreenshot('home.png', {
    maxDiffPixelRatio: 0.01,   // 允许 1% 像素差异
    fullPage: true,
  });
});

test('暗色模式', async ({ page }) => {
  await page.goto('/');
  await page.emulateMedia({ colorScheme: 'dark' });
  await expect(page).toHaveScreenshot('home-dark.png');
});
```

```bash
# 首次运行生成基准截图
npx playwright test --update-snapshots

# 后续运行对比
npx playwright test
```

### Chromatic (Storybook 项目)

```bash
pnpm add -D chromatic

# CI 中运行
npx chromatic --project-token=$CHROMATIC_TOKEN
```

## 部署策略

### 静态资源 CDN 部署

```yaml
  deploy:
    steps:
      - run: pnpm build

      # 上传到阿里云 OSS
      - name: Deploy to OSS
        uses: manyuanrong/setup-ossutil@v3.0
        with:
          endpoint: oss-cn-hangzhou.aliyuncs.com
          access-key-id: ${{ secrets.OSS_KEY_ID }}
          access-key-secret: ${{ secrets.OSS_KEY_SECRET }}
      - run: ossutil cp -r dist/ oss://my-bucket/app/ --update

      # 或上传到腾讯云 COS
      # - uses: TencentCloud/cos-action@v1
```

### Docker + Nginx 部署

```dockerfile
# Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package.json pnpm-lock.yaml ./
RUN corepack enable && pnpm install --frozen-lockfile
COPY . .
RUN pnpm build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
```

```nginx
# nginx.conf
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA 路由兜底
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 静态资源长缓存
    location /assets/ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # 安全头
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;

    # gzip
    gzip on;
    gzip_types text/css application/javascript application/json image/svg+xml;
    gzip_min_length 1024;
}
```

### 环境变量管理

```bash
# .env.development
VITE_API_URL=http://localhost:3000/api

# .env.staging
VITE_API_URL=https://staging-api.example.com

# .env.production
VITE_API_URL=https://api.example.com
```

```yaml
# CI 中按环境构建
- run: pnpm build --mode staging
  env:
    VITE_API_URL: ${{ vars.STAGING_API_URL }}
```

### 验证清单

- [ ] CI 流水线包含 lint → type-check → test → build 全链路
- [ ] 依赖安装使用 `--frozen-lockfile` 保证一致性
- [ ] 缓存已配置（node_modules、构建缓存）
- [ ] PR 有预览部署
- [ ] Lighthouse CI 性能预算已设置
- [ ] 部署后有健康检查
