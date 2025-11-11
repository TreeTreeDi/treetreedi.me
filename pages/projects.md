---
title: Projects - TreeTreeDi
display: Projects
description: List of projects that I am proud of
wrapperClass: 'text-center'
art: dots
projects:
  Current Focus:
    - name: 'CloudQuery'
      link: 'https://cloudquery.club/'
      desc: '数据库一体化操作平台'
      icon: 'i-carbon-data-base'
  Projects:
    - name: 'StreamFusion'
      link: 'https://github.com/TreeTreeDi/StreamFusion'
      desc: 'WebRTC 全栈直播平台'
      icon: 'i-carbon-video'
    - name: 'Multi-Mode Selection'
      link: 'https://github.com/TreeTreeDi/AG-Grid-Multi-Mode-Selection'
      desc: 'AG Grid Community 实现自定义的多模式选择'
      icon: 'i-carbon-grid'
    - name: 'markdown-flow'
      link: 'https://github.com/TreeTreeDi/markdown-flow'
      desc: '流式渲染的 Markdown npm 包'
      icon: 'i-carbon-document'
---

<!-- @layout-full-width -->
<ListProjects :projects="frontmatter.projects" />
