# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个基于 Vue 3 + Vite + SSG 的个人网站项目，使用 TypeScript 编写。核心技术栈包括：

- **框架**: Vue 3 + Vue Router (file-based routing)
- **构建**: Vite + vite-ssg (静态站点生成)
- **样式**: UnoCSS (原子化 CSS)
- **内容**: Markdown 文件作为页面，使用 gray-matter 解析 frontmatter
- **包管理**: pnpm workspace + catalog 依赖管理

## 常用命令

### 开发与构建

```bash
pnpm dev              # 启动开发服务器 (http://localhost:3333)
pnpm build            # 完整构建流程 (包含静态资源、SSG、RSS、字体拷贝)
pnpm preview          # 预览构建产物
```

### 代码质量

```bash
pnpm lint             # 运行 ESLint 检查和修复
```

### 内容管理

```bash
pnpm static           # 拉取静态资源和赞助商信息
pnpm photos           # 管理照片
pnpm compress         # 压缩暂存区的图片
pnpm redirects        # 生成重定向规则
```

## 架构要点

### 路由系统

- 使用 `unplugin-vue-router` 实现文件系统路由
- `pages/` 目录下的 `.vue` 和 `.md` 文件自动生成路由
- Markdown 文件的 frontmatter 通过 `gray-matter` 解析后注入到路由 meta 中

### Markdown 处理

- **插件链**: MarkdownItShiki (代码高亮) → anchor (标题锚点) → LinkAttributes (外链处理) → TOC (目录生成) → MarkdownItMagicLink (项目链接美化) → GitHubAlerts
- **Wrapper 组件**: demo 目录使用 `WrapperDemo`，其他使用 `WrapperPost`
- **OG 图片**: 构建时自动为文章生成 Open Graph 图片（使用 Sharp + SVG 模板）

### 组件自动导入

- Vue 组件从 `src/components/` 自动导入
- Icons 通过 `unplugin-icons` 自动导入（无前缀，如 `<i-ri-menu-2-fill />`）
- Vue API 和 VueUse 通过 `unplugin-auto-import` 自动导入

### 样式系统

- UnoCSS shortcuts: `bg-base`, `color-base`, `border-base` 适配深色模式
- 动态 shortcuts: `btn-{color}` 生成按钮样式
- 自定义规则: `slide-enter-{n}` 用于动画阶段控制

### 路径别名

- `~/` → `src/`（在 vite.config.ts 和 tsconfig.json 中配置）

### SSG 配置

- 使用 `vite-ssg` 进行静态站点生成
- 格式化模式: `minify`
- 路由滚动行为通过 `vue-router-better-scroller` 控制

### 依赖管理

- 使用 pnpm workspace catalog 统一管理版本
- 依赖分类: `build`, `dev`, `frontend`, `types`
- Sharp 版本锁定在 `0.32.6` (注意兼容性)

## 代码约定

### 组件命名

- 遵循 Vue 单文件组件规范（PascalCase）
- 查看 `src/components/` 现有组件了解命名风格

### Markdown 文件

- 使用 frontmatter 定义元数据（title, date, lang 等）
- 支持 Twoslash 类型标注（需显式触发）
- 支持 GitHub Alerts 语法

### 图片处理

- 照片存放在 `photos/` 目录
- 构建时使用 Sharp 进行图片优化
- 支持 EXIF 数据读取和 BlurHash

### ESLint

- 基于 `@antfu/eslint-config`
- 已禁用部分规则（见 eslint.config.js）
- 提交前通过 lint-staged 自动修复

## 重要文件

- `vite.config.ts` - Vite 配置、Markdown 插件链、OG 图片生成
- `src/main.ts` - 应用入口、SSG 配置、路由守卫
- `scripts/` - 构建脚本（RSS、字体拷贝、图片压缩、照片管理等）
- `typed-router.d.ts` - 自动生成的类型化路由定义
