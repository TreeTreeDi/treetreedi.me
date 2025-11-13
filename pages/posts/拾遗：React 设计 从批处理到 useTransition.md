---
title: 拾遗：React 设计 从批处理到 useTransition
date: 2025-11-13T10:00:00.000+00:00
lang: zh
duration: 10min
---

[[toc]]

看在微信读书看到了 《Vue.js 设计与实现》
现在也许是时间， 除开 Claude & Codex & AI提效，返璞归真重新整理一下 React 的原理了。

说起 React 性能，我会很本能地列一堆东西：

>     useMemo / useCallback / React.memo / 虚拟列表 / startTransition…

这套东西都没错，但它们都看起来太局部了。 很难理解为什么 React 18 需要并发模式、Fiber、批处理？
从浏览器渲染原理，再看 React 设计， 看起来就清晰多了。

---

## 二、批处理怎么来的（和宏任务 & 微任务的关系）

先看事件循环。
浏览器大致是这一套节奏：

1. 取一个宏任务
2. 跑完 JS，在这轮里把所有微任务（Promise.then 等）也清掉
3. 如果还有时间，再 layout / paint 一帧

**重点在这句：只要你还没跳到下一个宏任务，浏览器通常还没重绘。**

React 18 的批处理，其实就是利用了这一点：

- 同一事件回调 + 这条 async 链里的 `setState`，统统堆起来，一次性算完再渲染。
- 一旦 `setTimeout` 出去，切到下一个宏任务，就意味着：
  - 上一轮必须先处理完成（commit 一次）
  - 新的那次 `setState`，另起一轮算

所以批处理的看起来就是：

> **浏览器一帧只给我一次"改 DOM"的机会，
> React 在这一帧里尽可能把"能合并的更新"合起来。**

那么就要刻意避免这种写法：

```ts
setTimeout(() => {
  setA(1)
}, 0)

setTimeout(() => {
  setB(2)
}, 0)
```

这等于在强行把同一波逻辑拆成两批渲染。

---

## 三、Fiber：渲染必须能被打断 加快每一帧的数独

另一个硬约束是：**一帧的 JS 不能太长**。
老的 stack reconciler 是递归，一旦开始从根往下走，中途是停不下来的。

所以 React 利用了切片，使用了 Filber 既然浏览器可以在宏任务之间可以给帧渲染留下时间。

1. **一棵可遍历的树**
   `child / sibling / return`，方便做「下一个节点」的遍历。

2. **双缓冲的两棵树**
   - current 树：现在 DOM 对应的那一版
   - workInProgress 树：下一版正在算的

3. **一个全局指针 `nextUnitOfWork`**
   每次只处理一个 Fiber，干完就问一句：**要不要让出主线程？**

简化就是：

```ts
while (nextUnitOfWork && !shouldYield()) {
  nextUnitOfWork = performUnitOfWork(nextUnitOfWork)
}
```

只要 `shouldYield()` 就表示有更高优先级的来了，
React 就把 `nextUnitOfWork` 留着，等浏览器画完这一帧，再接着算。

而因为有 **current / workInProgress 两棵树**：

- 中断的时候，屏幕上用的还是 current 树对应的 DOM
- 半截算完的 workInProgress直接就结束掉了。

一句话概括下来

> Fiber 不是为了"优雅的结构"，
> 是让渲染变成一串可以随时「停下来、重跑、丢掉」的小任务。

---

## 四、处理并发的副作用: render 必须纯，副作用使用 effect

渲染可以随时被打断 / 重跑，有一个非常直接的副作用：

> **render 里面不能做有副作用的事。**

- 如果你在函数组件 body 里发请求、改全局变量、直接操作 DOM：
  - 一次中断 + 重跑，就等于重复执行
  - 并发模式下更离谱：可能算了一半就丢掉，再重来一遍

这时候时间线就很关键了：

1. 调 `setState`
2. Render 阶段：算新的 Fiber 树（可打断）一旦被打断就会重现开始执行， 这个时候就会再次调用请求 & 全局变量修改（副作用）
3. Commit 阶段：
   - 改 DOM
   - 跑 `useLayoutEffect` cleanup + create（同步）
4. JS 结束，浏览器做 layout / paint
5. 下一轮任务里，React 再跑 `useEffect`

StrictMode 在这里的作用就变得好理解了：模拟"渲染被打断和重做， 避免多次的事件绑定。（比如多次的 listen）。

---

## 五、startTransition 直接和优先级沟通

有了 Fiber 和可中断渲染，React 还可以再往前走一步：
浏览器只在意"主线程有没有被占满"，
但 React 可以再细一点：**哪些更新必须先做，哪些可以往后排。**

`startTransition`，就是把「用户立刻能感知的交互」和「可以慢一拍的重渲染」拆开。

典型场景：输入 + 重列表筛选。

最原始写法：

```ts
const [keyword, setKeyword] = useState('')
const [list, setList] = useState(allItems)

function onChange(e) {
  const v = e.target.value
  setKeyword(v)
  setList(expensiveFilter(allItems, v)) // 重活
}
```

没做任何事的时候：

- React 批处理
- 但整块计算 + 渲染仍然都是高优先级会影响加载

下面的写法就非常的 FID 和 INP 友好了。

```ts
const [keyword, setKeyword] = useState('')
const [list, setList] = useState(allItems)
const [isPending, startTransition] = useTransition()

function onChange(e) {
  const v = e.target.value
  setKeyword(v) // 紧急

  startTransition(() => {
    setList(expensiveFilter(allItems, v)) // 不紧急
  })
}
```

总结

那么直接就可以在现有的 React 优化路径上， 补充对 React 18 的理解了。

1. 需要更规范的代码， 满足 strictMode 的要求。
2. 直接使用 startTransition 来和优先级沟通。
