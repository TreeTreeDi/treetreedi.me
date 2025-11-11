---
title: streaming-markdown 三层 Memoization，Markdown 优化渲染速度
date: 2025-11-11T10:00:00.000+00:00
lang: zh
duration: 8min
---

[[toc]]

## TL;DR

- 把渲染拆成 **容器 / 块 / 原子组件** 三层，每层各自 memo，对应不同粒度的变化源。
- **块级切分**靠 Lexer（`marked`）；**变更判断**在组件层用 **AST 位置对比**（O(1)）。
- 流式场景下追加文本仅新增块重渲染，旧块 90%+ 命中缓存；原子组件命中率 ~99%。
- 代码高亮用 **Shiki 单例** + **静态语言/主题注册表**，降打包内存。

---

## 背景与判断

流式渲染的核心矛盾：**输入频繁增量变化**，但**已渲染多数内容不变**。单个 `ReactMarkdown` 重新跑一遍很亏；仅做字符串比较很脆弱；在组件里深度比 AST 又太重。
我的取舍是把“变化”分三档处理：

- **Layer 1（容器）**：解析最贵 → 缓一次，**输入没变就别再切块**。
- **Layer 2（块）**：追加时大多是“**新增块**”，老块直接复用。
- **Layer 3（原子组件）**：细颗粒度判断“**这段 AST 的位置是否变了**”，位置不变视为等价，避免深比较。
  这套划分让每层都只对付“它该负责的变化”。

---

## 架构总览（三层 Memoization）

- **Layer 1：Container/useMemo**
  用 `parseMarkdownIntoBlocks(markdown)` 把文本切成语义块；依赖 `displayedText`。

- **Layer 2：Block/React.memo + content 比较**
  每块独立渲染；只要 `content` 不变就跳过。

- **Layer 3：Atom Components/React.memo + AST 位置比较**
  `sameNodePosition` 比四个数字（start/end 的 line/column），**O(1)**。

> 反模式提醒：不要在渲染期临时拼 `components={...}` 对象，**会破坏缓存**。请 `useMemo` 固定它。

---

## 实现要点

### 1）块级切分：用 Lexer 而不是正则

- 保护脚注引用/定义、配对 HTML、配对 `$$` 数学块，**切分不破语义**。

- 签名：`parseMarkdownIntoBlocks(markdown: string): string[]`

> 这样设计的好处是**天然适配流式**：追加文本只会多出新的块，不影响前面的 token 边界。

### 2）原子组件变更判断：AST 位置 O(1) 对比

- `sameNodePosition(prev, next)` 同比 `start/end` 的 `line/column`。

- 对链接/图片等，再加 `href/title` 等关键属性对比。

> 核心观点：**位置没变=内容没变**（在 `react-markdown` 的位置信息稳定前提下）。避免深比较带来的时间炸弹。

### 3）Shiki 高亮：单例 + 静态注册表

- 一个管理器维护 light/dark 两个实例；**语言按需加载**；主题变更时再重建。

- 语言与主题用静态映射，绕开 Turbopack 动态导入限制。

> 这样做把“多块代码 = 多个 8MB 高亮器实例”的内存降低，真实项目里差距非常大。

---

## 与 vercel/streamdown 的关系

这次思路受 vercel streamdown **分层流式渲染** 的启发：把“可增量更新”的边界往上抽到“块/组件级”，让 React 的 diff 真正命中“**只更新发生变化的那部分**”。同时，我在块切分、AST 位置对比和 Shiki 资源管理上做了工程化落地，适配前端常用构建链路与流式 UI 场景。

---

## 关键代码片段（节选）

**Block 组件（Layer 2）**

```tsx
export const Block = memo<BlockProps>(
  ({ content, components, remarkPlugins, rehypePlugins, className }) => (
    <ReactMarkdown
      className={className}
      components={components}
      remarkPlugins={remarkPlugins}
      rehypePlugins={rehypePlugins}
    >
      {content}
    </ReactMarkdown>
  ),
  (p, n) => p.content === n.content && p.className === n.className
)
```

**AST 位置对比（Layer 3）**

```ts
export function sameNodePosition(prev?: MarkdownNode, next?: MarkdownNode) {
  if (!(prev?.position || next?.position))
    return true
  if (!(prev?.position && next?.position))
    return false
  const ps = prev.position.start; const pe = prev.position.end
  const ns = next.position.start; const ne = next.position.end
  return ps.line === ns.line && ps.column === ns.column && pe.line === ne.line && pe.column === ne.column
}
```

**容器（Layer 1）**

```tsx
const blocks = useMemo(() => parseMarkdownIntoBlocks(displayedText), [displayedText])
const components = useMemo(() => ({
  h1: MemoH1,
  h2: MemoH2, /* ... */
  code: MemoCode
}), [/* 保持依赖稳定 */])

return (
  <div className={className}>
    {blocks.map((b, i) => (
      <Block key={`${id}-b-${i}`} content={b} components={components} remarkPlugins={[remarkGfm]} />
    ))}
  </div>
)
```

---

## 数学块与 HTML 的特殊处理（工程化细节）

- **数学块**：统计 `$$` 奇偶，**奇数 → 延迟合并**，保证块级公式不被截断；行内数学不要单独成块。

- **HTML 片段**：栈式配对 `<tag>...</tag>`，整段并入同一块，避免被流式增量破坏 DOM 结构。

- **脚注**：`[^id]` 与 `[^id]:` 视作绑定，**禁止跨块拆开**。
  这些规则都放在 **Lexer 切分阶段**，后面两层就能“放心偷懒”。

代码地址：
https://github.com/TreeTreeDi/markdown-flow
