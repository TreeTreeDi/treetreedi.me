---
title: 从 0 到 1 体验 CI/CD
date: 2025-11-10T08:53:00.000+00:00
lang: zh
duration: 8min
description: 第一次把项目接上 CI/CD 的感受、踩坑和最小落地方案
---

[[toc]]

> 这是一篇**入门体验记**:我第一次把项目接上 CI/CD 的感受、踩坑和最后落地的最小方案。
> 是在开发 npm 包的初步尝试。使用 CI 来处理 lint 操作, 用 CD 来发布 playground。

## TL;DR

- 我把“**部署与预览**”交给 **Vercel**；把“**代码质量与包发布**”交给 **GitHub Actions**。

- 入门只需两步：① Lint 工作流每次 push 自动跑；② 推 `v*` 标签就**自动发 npm**。

- 关键点：**pnpm workspace `--filter`** 精准到某个包；**npm provenance + OIDC** 更安全。
- 想先感受成效：**先 Vercel 接仓库**→ 每个 PR 都有预览；再补上 lint 工作流；最后再上 npm 发布。

---

## 我的起点（真实感受）

- 一开始我只想“**先自动起来**”，而不是上来就堆满复杂流程。

- Vercel 可以做到 PR 预览、main 自动上线；

- Actions 让我把**质量**（lint）和**分发**（npm 发布）分开，出问题也好排查。

---

## 我对 CI/CD 的直觉化理解

- **CI（持续集成）**：每次提交自动检查、构建，让问题早点暴露（我用 lint 来兜底）。
- **CD（持续交付/部署）**：代码合并后能自动发到用户能访问的地方（我用 Vercel + npm 发布）。

- **Vercel 是不是 CI/CD？** 是。它负责**构建+部署**，对前端入门特别友好；但**发 npm 包**、**到自有服务器**这类动作，还是用 Actions 更顺手。

---

## 我的最小方案

### 1) 代码质量：Lint 工作流（每次 push）

`.github/workflows/lint.yml`

```yaml
name: Lint
on: [push, pull_request]

jobs:
  lint:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: {version: 9.12.3}
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - run: pnpm install --frozen-lockfile
      - name: Lint @stream-md/react
        run: pnpm --filter @stream-md/react lint
```

**为什么这样做**

- 每次 push 自动给我“质量信号”，改动是不是把规则踩了，一眼就知道。

- `--filter` 只跑开发重点 npm 包。

---

### 2) 包发布：推标签自动发 npm

`.github/workflows/publish.yml`

```yaml
name: Publish npm
on:
  push:
    tags: ['v*'] # 例如 v1.2.3

permissions:
  id-token: write
  contents: read

jobs:
  publish:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: {version: 9.12.3}
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
          registry-url: 'https://registry.npmjs.org'
          always-auth: true
      - run: pnpm install --frozen-lockfile
      - name: Build
        run: pnpm --filter streaming-markdown-react build
      - name: Publish
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}
        run: pnpm --filter streaming-markdown-react publish --access public
```

**如何触发一次发布**

```bash
# 版本号自管：可以先改 package.json 或用工具链
git tag v1.0.0
git push origin v1.0.0
```

**为什么这样做**

- `tags: v*` = **标签即发布**，清晰、可审计；

- `id-token: write` = 开启 **provenance**（来源证明），配合 OIDC 更安全；

- 仍然用 `NPM_TOKEN` 作为落地凭证，简单直接。

---

### 3) 部署：Vercel 直接接仓库

- 连接 GitHub 仓库后：
  - 每个 PR 自动生成**预览链接**；

  - push 到 `main` 自动发到**生产**；

  - 构建命令用 `pnpm build`，产物 `.next/` 自动识别；

  - 环境变量在 Vercel 面板分环境管理。

**我的体感**：这是**最容易看到效果**的一步——给自测省了大量来回。

---

## 决策表（入门角度）

| 我的问题/目标 | 我现在的做法         | 为什么                     |
| ------------- | -------------------- | -------------------------- |
| 想“先跑起来”  | 先接 Vercel          | 一提交就能看页面，激励明显 |
| 提交要有兜底  | 加 lint 工作流       | 快，反馈明确，修起来轻     |
| 要发 npm 包   | 推 `v*` 标签触发发布 | 简单，发布历史一目了然     |
| 多包仓库      | `pnpm --filter`      | 只处理目标包，节省时间     |
| 安全合规      | OIDC + provenance    | 减少长期密钥，来源可追     |

**阈值**（对我有效的前提是）：

- 真有“发包/自托管/容器”需求时，再补 Actions 的部署 Job。

---

## 我踩过的坑 & 解决

- **本地能过，CI 报 pnpm 版本问题**
  → 在工作流固定 `pnpm 9.12.3`，并用 `--frozen-lockfile`。
- **workspace 跑全仓太慢**
  → `pnpm --filter <包名> <命令>`。

- **发包权限 401**
  → 检查 `NPM_TOKEN` 是否是“发布令牌”，并确认包名和作用域权限。

- **Vercel 构建找不到 env**
  → 在 Project Settings 的 Production/Preview 分开配置环境变量。

---

## 后续怎么进阶（我打算这么做）

- 给“生产发布”加一个 **Environment 审批**（防止误发）。

- 把**变更日志/版本号**用 `changesets` 自动化。

- 如果要上服务器/容器：在 Actions 新增一个 **deploy job**（SSH/S3/Docker/K8s 任选），仍然保持“**标签驱动**”。
