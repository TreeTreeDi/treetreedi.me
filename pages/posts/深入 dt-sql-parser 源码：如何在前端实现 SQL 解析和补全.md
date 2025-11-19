---
title: 深入 dt-sql-parser 源码：如何在前端实现 SQL 解析和补全
date: 2025-11-19T10:00:00.000+00:00
lang: zh
duration: 15min
---

[[toc]]

这篇文章从源码出发，说明 `dt-sql-parser` 是怎么在浏览器里做 SQL 解析、校验和补全的。提升前端理解。 重点放在几个具体问题上：

- 为什么要在前端跑 Parser，而不是完全依赖后端？

- 为了保证编辑体验，内部做了哪些缓存和增量解析的设计？

- 补全时是怎么定位光标、切分上下文、再用 `antlr4-c3` 做预测的？

- 除了补全，还能从 AST 里挖出哪些有用的信息（比如表名）？

---

## 一、为什么要在浏览器里跑 Parser？

把 Parser 放到前端，本质上是把一部分“编译工作”提前做。

主要考虑有三个：

1. **交互延迟**
   语法错误最好在几毫秒级别反馈。比如少了一个分号，编辑器直接在本地标红，而不是发一次 HTTP 请求再等响应。

2. **后端负载**
   语法层面的错误（Syntax Error）可以在前端就拦截掉，后端只保留相对昂贵的语义分析（比如表是否存在、权限是否够）。

3. **离线体验**
   即便网络不稳定，只要 Parser 能跑，基本的语法高亮、错误提示、部分补全还是可以正常工作。

对应地，也会引入一些成本：

- 前端执行 Parser 的性能不如服务端（尤其是 Java/C++） -> 需要控制执行消耗。
- 前端要维护一套自己的“编译上下文”，和后端逻辑保持一致。

`dt-sql-parser` 的实现思路是：把解析过程拆开、能缓存的都缓存，减少每次键盘输入触发的工作量；补全则通过裁剪输入、缩小上下文范围来控制消耗。

---

## 二、核心架构：缓存 + 受控刷新

ANTLR 的完整链路是：

> 字符串 → Lexer → TokenStream → Parser → Parse Tree(AST)

如果每次输入都从头跑一遍，体验会非常差。`dt-sql-parser` 在基类里做了一个简单的约束：**只要输入字符串不变，中间产物就都复用**。

### 1. validate 只负责“触发解析 + 读取结果”

先看 `validate`，可以看到它自己并不做解析，而是全部交给 `parseWithCache`：

<!-- eslint-skip -->

```ts
// 利用缓存机制，避免重复计算
validate(input: string): ParseError[] {
  this.parseWithCache(input)
  return this._parseErrors
}
```

这里有两个点：

- `validate` 保证了“先解析再读错误”，调用方不用关心内部缓存细节；
- 在注释中有看到， 维护者有考量这里的问题， 也就是作为一个工具函数执行了部分数据缓存， 但是如果在整个系统的设计中， 还是有可以取舍的。
- 缓存逻辑集中放在 `parseWithCache`，后面 `getAllTokens`、补全等能力也都复用这套机制。

### 2. parseWithCache 入口文件

从 sqlBase.ts 访问 `parseWithCache`, 在解析的过程缓存数据 ：

<!-- eslint-skip -->

```ts
parseWithCache(input: string, errorListener?: ErrorListener): PRC {
  // 如果输入没变，且不需要额外的 Listener，直接复用上次的 ParseTree
  if (this._parsedInput === input && !errorListener && this._parseTree) {
    return this._parseTree
  }

  // 1. 清空错误集合
  this._parseErrors = []

  // 2. 创建新的 Parser，内部会顺带刷新 Lexer 和 TokenStream
  const parser = this.createParserWithCache(input)
  this._parsedInput = input

  // 3. 挂载错误监听器
  parser.removeErrorListeners()
  parser.addErrorListener(this.createErrorListener(this._errorListener))

  // 4. 生成新的 ParseTree 并缓存
  this._parseTree = parser.program()
  return this._parseTree
}
```

---

## 三、核心逻辑 语法补全：定位、裁剪、预测

补全比校验复杂得多。校验只要回答“有没有错误”，补全要回答“在这里用户可以写什么”。

`dt-sql-parser` 的补全过程可以拆成三个问题：

1. 光标对应哪一个 Token？
2. 这一段补全最少需要什么上下文？
3. 在这个上下文里，`antlr4-c3` 给出了哪些候选？

### 1. 光标定位：从 (行, 列) 到 Token Index

编辑器通常给出的是 `(line, column)`，但 ANTLR 这边处理的是 Token 数组，所以需要一个映射。

二分查找定位光标：

```ts
export function findCaretTokenIndex(
  caretPosition: CaretPosition,
  allTokens: Token[]
): number | undefined {
  const { lineNumber: caretLine, column: caretCol } = caretPosition
  let left = 0
  let right = allTokens.length - 1

  while (left <= right) {
    const mid = left + ((right - left) >> 1)
    const token = allTokens[mid]

    // 条件 1：Token 在光标之后（或同一行但列号更靠后）
    if (token.line > caretLine || (token.line === caretLine && token.column + 1 >= caretCol)) {
      right = mid - 1
    }
    // 条件 2：Token 在光标之前（或同一行但已经结束）
    else if (
      token.line < caretLine
      || (token.line === caretLine && token.column + (token.text?.length ?? 0) + 1 < caretCol)
    ) {
      left = mid + 1
    }
    else {
      // 条件 3：光标落在当前 Token 内部
      return allTokens[mid].tokenIndex
    }
  }
  return void 0
}
```

这里的边界处理有几个点值得注意：

- 用 `token.column + 1` / `+ text.length + 1` 来处理“不在 token 内而刚好在左右空隙”的情况；
- 返回值用 `tokenIndex` 而不是数组下标，方便后续和 ParseTree 里的引用对齐。

### 2. 输入切片：收缩到“最小安全上下文”

如果用户的 SQL 有 5000 行，而光标在最后一行，直接把全文交给 `antlr4-c3`，性能会很吃紧，而且前面某一行的语法错误可能会“污染”后面的补全。

`dt-sql-parser` 的做法是：基于语句边界，把输入裁剪到一个“包含光标、又尽量小”的区间。伪代码流程大致是：

```ts
// 根据完整 ParseTree 拿到语句列表
const splitListener = this.splitListener
this.listen(splitListener, originParseTree)
const statementsContext = splitListener.statementsContext

// ... 根据 caretTokenIndex 找到语句索引 index ...

// 判断前后语句是否“健康”
const isPrevCtxValid = index === 0 || !statementsContext[index - 1]?.exception
const isNextCtxValid = index === statementCount - 1 || !statementsContext[index + 1]?.exception

// 根据健康情况，决定切片范围
const startIndex = startStatement?.start?.start ?? 0
const stopIndex = stopStatement?.stop?.stop ?? inputSlice.length
```

这里有两个设计点：

- **按语句切，而不是按行切**
  用语法树配合 Listener，先把 SQL 切成“语句”粒度，再找出光标所在的那一条或几条。

- **前后带一些“缓冲区”**
  如果前一条或后一条语句有异常，可能会影响当前语句的解析，这时候会把它们一起包含进切片，不完全只看当前一条。

最后，补全只针对这个切片再跑一次解析与 `antlr4-c3` 分析，减少了需要处理的 Token 数量，也尽量降低了远处错误对当前补全的影响， 这一部分很可能是团队根据使用场景迭代的结果。

### 3. 基于上下文的预测结果整形

拿到切片对应的 Token 流后，`antlr4-c3` 会给出一组候选：可能的规则、关键字等。`dt-sql-parser` 再把这些结果映射成业务更易用的结构。

简化后的处理大致如下：

```ts
const syntaxSuggestions: SyntaxSuggestion<WordRange>[] = originalSuggestions.syntax.map(
  (syntaxCtx) => {
    const wordRanges: WordRange[] = syntaxCtx.wordRanges.map((token) => {
      return tokenToWord(token, this._parsedInput)
    })
    return {
      // 将底层 Rule 映射到业务上的语义类型
      syntaxContextType: syntaxCtx.syntaxContextType,
      wordRanges,
    }
  }
)
return {
  syntax: syntaxSuggestions,
  keywords: originalSuggestions.keywords,
}
```

可以看到这里刻意做了一个拆分：

1. `keywords`：静态关键字，比如 `SELECT` / `FROM` 等 （直接语法补全）；
2. `syntax`：语法上下文类型，比如“这里需要一个表名”、“这里需要列名”（和后端直接沟通）。

有了 `syntaxContextType`，前端就可以只负责 UI 和交互：

- 在 `FROM` 后面出现 “需要 table / view”的上下文时，去请求后端元数据，给出实际的表名列表；

- 在 `SELECT` 列表位置出现 “需要 column”的上下文时，用当前已选表的列来做补全。

Parser 不负责业务数据，只负责解决在这个位置需要什么类型的东西。

---

## 四、Visitor 模式：在 AST 上做数据采集

除了做错误提示和补全之外，前端经常还想知道：用户到底用了哪些表、哪些字段。这类需求更适合用 Visitor 做。

通过 antlr-4g 来实现的。 这个部分相当于封装。 https://github.com/mike-lischke/antlr4ng

`dt-sql-parser` 对应的 Visitor 用法比较标准，下面是一个提取 MySQL 表名的例子：

```ts
import { MySQL, MySqlParserVisitor } from 'dt-sql-parser'

const mysql = new MySQL()
const sql = `select id, name from user1;`
const parseTree = mysql.parse(sql)

// 自定义 Visitor，继承自动生成的基类
class MyVisitor extends MySqlParserVisitor<string> {
  // 默认返回值
  defaultResult(): string {
    return ''
  }

  // 聚合多个子节点的遍历结果
  aggregateResult(aggregate: string, nextResult: string): string {
    return aggregate + nextResult
  }

  // Program 节点：继续遍历子节点
  visitProgram = (ctx) => {
    return this.visitChildren(ctx)
  }

  // TableName 节点：直接取文本
  visitTableName = (ctx) => {
    return ctx.getText()
  }
}

const visitor = new MyVisitor()
const result = visitor.visit(parseTree)
console.log(result) // user1
```

---

## 五、适用范围与边界

`dt-sql-parser` 的定位比较明确：**把语法层能力前移到前端**。有几个场景更适合用它，也有一些不适合的情况。

| 场景                              | 建议                               | 说明                         |
| --------------------------------- | ---------------------------------- | ---------------------------- |
| 语法高亮 / 简单关键字识别         | 直接用 Monarch / 正则              | Parser 过重，不划算          |
| 实时语法错误提示                  | 使用 `dt-sql-parser`               | 需要 AST 才能做到精确定位    |
| 语法级补全（比如关键字、结构）    | 使用 `dt-sql-parser` + `antlr4-c3` | 需要语法上下文               |
| 复杂语义校验（表/列存在性、权限） | 放在后端                           | 前端没全量元数据，很难做全   |
| 超大 SQL（> 1 万行）              | 视情况改为后端异步分析             | 即使做了切片，仍可能卡主线程 |

如果你只想做基础高亮和简单关键字提示，`dt-sql-parser` 设计上可能不太好处理；但如果需要在浏览器里做较完整的语法检查和补全，它会比较合适。

---

## 六、小结

完成了对 sql-parser 的解读。
在普通的解析场景， 就可以直接用前端来完成， 当然在企业场景需要有非常多的考量。 但是在项目设计上非常值得学习。

ANTLR 的前端 Parser 补充，可以从这几步开始：

- 拿一个简单方言（例如 MySQL）对应的 `.g4`； https://github.com/antlr/grammars-v4
- 编译生成 TypeScript Parser 代码；
- 仿照上面的缓存、补全、Visitor 结构，慢慢往编辑器里接。
- 从 jest 测试文件开始 debug 函数。

![sqlparser 1763551836359](https://raw.githubusercontent.com/TreeTreeDi/my-image-here/main/sqlparser-1763551836359.png)
