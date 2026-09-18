# Product

<!-- impeccable:product-schema 1 -->

## Platform

web

## Users

[inferred from published pages] Primary visitors come to read TreeTreeDi's technical writing or look up TreeTreeDi's public work.

Published posts are in Chinese and cover frontend engineering, tooling, and adjacent systems topics.

Other audiences (conference organizers, sponsors, English-only readers) are not established by the published identity.

## Product Purpose

[from site copy] [https://treetreedi.me](https://treetreedi.me) is TreeTreeDi's personal website: a public home, a blog, a project list, and a way to find TreeTreeDi on GitHub.

Success is that a visitor can read the writing, see the listed work, and reach the GitHub profile.

This is not a hosted product, marketplace, or company site.

## Positioning

[inferred] A first-person site for TreeTreeDi's own notes and projects.

The mechanism is authorship: the posts and the listed projects are the product.

A neighboring engineer blog can copy the layout. It cannot truthfully publish this writing or this project list as its own.

## Operating Context

- Public site: [https://treetreedi.me](https://treetreedi.me)
- Primary chrome: Home, Blog, Projects, Life, GitHub, RSS, theme toggle
- Content lives as Markdown pages and is built to a static site
- RSS is advertised as TreeTreeDi's Blog (`/feed.xml`)
- The feed generator currently includes only `lang: en` posts; every dated post in this repo is `lang: zh`
- Feed author email on file: `hi@treetreedi.me`
- Local development: `pnpm dev` (Vite, port 3333)

## Capabilities and Constraints

Confirmed on the published surfaces:

- Home introduces TreeTreeDi and links GitHub
- Blog lists dated Markdown posts
- Projects lists CloudQuery, StreamFusion, Multi-Mode Selection, and markdown-flow, using only the one-line descriptions already on `/projects`
- Light/dark theme, image lightbox on prose and photo pages, Mermaid in posts, and RSS/Atom/JSON feed files
- Code is MIT; words and images are CC BY-NC-SA 4.0 (2025–present © TreeTreeDi)

Constraints and leftovers:

- This repo is a fork of Anthony Fu's antfu.me
- Leftover routes still exist (talks, sponsors, use, chat, podcasts, demos, notes, bookmarks, streams, giving-talks)
- Those leftover pages still name Anthony Fu and are not TreeTreeDi product truth
- Photos / Life is in the nav; `photos/` has no published image files in this checkout, and the page title still says Anthony Fu
- Post share links still point at antfu.me / `@antfu7`
- The site publishes no pricing, testimonials, headcount, or unpublished product roadmap
- CloudQuery is listed as current focus with the published line "数据库一体化操作平台" and [https://cloudquery.club/](https://cloudquery.club/). Do not expand that into unpublished product claims

Undecided:

- How leftover Anthony Fu routes should be retired or rewritten
- Whether English posts or a working Chinese RSS feed are intended
- Whether Life / photos is a surface TreeTreeDi will fill

## Brand Commitments

- Name: TreeTreeDi
- Site: treetreedi.me
- Homepage voice: short, first person, English greeting ("Hey! I'm TreeTreeDi.")
- Blog index copy: "我的博客文章"
- Footer: `CC BY-NC-SA 4.0 2025-PRESENT © TreeTreeDi`
- GitHub: [https://github.com/treetreedi](https://github.com/treetreedi)
- Site title "TreeTreeDi"; favicon `/favicon.svg`

## Evidence on Hand

Published dated posts (all `lang: zh`):

- Claude Code 记忆系统工程指南 (2026-02-20)
- 深入 dt-sql-parser 源码 (2025-11-19)
- 拾遗：React 设计 从批处理到 useTransition (2025-11-13)
- Claude Code Hooks (2025-11-11)
- 从 TFT & TTI 出发的前端分包理解 (2025-11-11)
- streaming-markdown 三层 Memoization (2025-11-11)
- 数据中台：数据库操作模块的内存治理 (2025-11-10)
- Tree 组件按需渲染 (2025-11-10)
- 从 0 到 1 体验 CI/CD (2024-11-10)

Published project list: `pages/projects.md`

Do not fabricate testimonials, press, usage numbers, or customers.

## Product Principles

1. Publish only TreeTreeDi's own writing and listed work. Leftover fork copy is not identity.
2. Keep claims as short as the site: one-line project descriptions, no invented product stories.
3. Serve readers who came to learn from the posts, not to convert through marketing.
4. Preserve dual licensing: MIT for code, CC BY-NC-SA 4.0 for words and images.
5. Treat Chinese technical posts as the current content language. Do not require English unless TreeTreeDi writes it.
