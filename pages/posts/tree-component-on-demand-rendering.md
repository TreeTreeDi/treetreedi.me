---
title: 为什么我在 Tree 组件放弃"每次重建"，在复杂场景改用"按需渲染"
date: 2025-11-10T16:00:00.000+00:00
lang: zh
duration: 15min
---

[[toc]]

## TL;DR

- 在 CQ（公司数据库中台）里，**小规模/简单选择器**用“**方案一：一次性构建搜索树**”；**大规模/强交互**用“**方案二：按需加载 + 双路搜索**”

- **阈值**：节点数 `< 1000` 且**无**权限/右键/编辑 → 选方案一；否则选方案二

- 方案二通过 `loadedKeys/expandedKeys + requestManager + 缓存` 复用节点，**连续搜索**与**分页**表现显著更好

- **实测（2025-11-10）**：首次搜索 300ms（方案二） vs 800ms（方案一）；连续搜索 150ms vs 750ms；帧率 58 FPS vs 25 FPS

- 快速原型/MVP 上线：先用方案一；落到生产的主树：切到方案二

---

## 业务背景（CQ 场景）

两个核心用例需要把后端 Schema 渲染成树：

- **数据操作**：连接 → 数据库 → 表 → 列

- **权限管理**：树形选择器（仅选择）

旧实现的**显性问题**：

1. **分页父子错位**：父节点不在首分页而子节点已写入，展开失败

2. **重复渲染**：多次搜索返回相同内容，节点重复

3. **性能差**：大表检索慢、渲染卡顿

4. **频繁重建**：每次搜索整棵树重建，虚拟 DOM diff 成本高

**新目标**：快响应、可中断旧请求、按需加载、不重建整棵树、按场景分治

---

## 方案一：一次性构建搜索树（SelectTreeData）

**思路**：用一次后端搜索结果**直接构建完整树**；搜索树与原始树**分离**。

**核心流程**：

```ts
const results = await searchDatabaseFullText(trimmedValue)
const searchTree = buildTreeFromSearchResults(results, endInConnection, Iconfont)
const expandKeys = extractAllExpandKeys(searchTree)
setTreeData(searchTree)
setTreeExpandedKeys(expandKeys)
setTreeLoadedKeys([])
```

**要点**

- 扁平→层级：`buildTreeFromSearchResults`（标记 `fromSearch: true`）
- 清空搜索立即**恢复原始树**

**优点**

- 简单直观、实现/维护成本低
- 一次请求立刻“有样子”

**缺点/边界**

- 维护两份数据（内存高）
- 不适合权限/右键/编辑
- 要求搜索接口返回**完整路径**

**适用**

- 节点 `< 1000`、功能简单、MVP/原型

- 海量数据、复杂交互

---

## 方案二：按需加载 + 场景区分（Sdt + useDictSearch）

**思路**：**双路搜索**（已加载节点走前端，未加载节点走后端）；用 `loadedKeys/expandedKeys` 托管加载，**不重建**树；**请求可中断**，并复用**缓存**。

**关键技术**

- **双路搜索**：先即时前端结果，后 debounced 后端搜索（标记 `_resultType`）

- **智能 onLoadData**：识别“搜索目标父节点”→ 选择**搜索接口**或**常规接口**

- **requestManager**：相同父路径的过期请求**Abort**

- **状态一致**：搜索与非搜索共用一棵树 + `treeNodeChildrenMap` 缓存

- **分页友好**：`clearSingleNodeData` 清父节点后，用搜索接口补全子树

**优点**

- 高性能：**不重建**，只精确更新变更节点

- 连续搜索体验稳定；分页、并发、缓存全覆盖

- 扩展性强：权限检查、右键、编辑

**成本/风险**

- 需要 Redux/异步管理经验
- 时序/边界调试复杂，耦合度更高

**适用**

- 节点 `> 1000`、权限/右键/编辑、分页、高频搜索导航

- 赶工的极简场景

---

## 取舍表（阈值与触发条件）

| 维度      | 方案一（一次性构建） | 方案二（按需 + 双路） | 触发条件                  |
| --------- | -------------------- | --------------------- | ------------------------- |
| 学习/接入 | 低                   | 中-高                 | 是否有 Redux/异步管理基础 |
| 首次响应  | 快（一次请求）       | 中（双路 + 去抖）     | 是否必须“秒出样子”        |
| 连续搜索  | 低（每次重建）       | 高（复用节点）        | 搜索是否高频              |
| 内存占用  | 高（双份树）         | 低（单树 + 缓存）     | 搜索结果体量              |
| 分页支持  | 否                   | 是                    | 后端是否分页/游标         |
| 扩展性    | 低                   | 高                    | 权限/右键/编辑是否需要    |
| 何时选    | `< 1000` + 简单      | `> 1000` 或强交互     | 节点规模 & 交互复杂度     |

**决策规则**

- **满足任一**：节点 `> 1000`｜有权限/右键/编辑｜需分页 → 选**方案二**
- 否则默认**方案一**；若从 MVP 走向生产，**迁移到方案二**

---

## 实测数据

环境：Chrome 120 / Mac mini M4 / 16GB RAM
规模：1 连接、2000 库、每库 2000 表

| 操作         | 方案一 | 方案二 |                         提升 |
| ------------ | -----: | -----: | ---------------------------: |
| 首次搜索耗时 |  800ms |  300ms |                    **62.5%** |
| 连续搜索耗时 |  750ms |  150ms |                      **80%** |
| 渲染帧率     | 25 FPS | 58 FPS |                    **+132%** |
| 内存占用     |  120MB |   65MB |                   **-45.8%** |
| 清空搜索耗时 |   50ms |  200ms | 方案一更快（代价：状态丢失） |

> 注：方案二的清空较慢，换来状态与缓存一致性

---

## 最小可行实践（MVP 步骤）

### A) 方案一落地（≤ 1000 节点）

1. **接入搜索接口**：确保返回**完整路径**（连接/库/表/列）

2. **构建搜索树**：使用 `buildTreeFromSearchResults`，为节点标记 `fromSearch: true`

3. **展开逻辑**：`extractAllExpandKeys` 一次性展开；清空搜索即恢复原树

4. **懒加载兜底**：搜索态下 `onExpand` 强制 `loadData`，补全缺口

**关键片段**

```ts
const results = await searchDatabaseFullText(q)
const tree = buildTreeFromSearchResults(results, endInConnection, Iconfont)
setTreeData(tree); setTreeExpandedKeys(extractAllExpandKeys(tree)); setTreeLoadedKeys([])
```

### B) 方案二落地（生产主树）

1. **状态托管**：统一 `expandedKeys/loadedKeys/treeNodeChildrenMap`（Redux）

2. **双路搜索**：已加载先搜前端 → 更新下拉；随后 debounce 后端搜索

3. **智能加载**：`onLoadData` 识别“搜索目标父节点”→ 选择搜索接口

4. **请求管理**：对相同父路径启用 `requestManager.abortController`
5. **分页一致**：命中搜索路径时先 `clearSingleNodeData(parent)` 再按搜索接口补全

6. **清空策略**：先清展开/已加载，再延时清缓存，最后恢复展开状态

**关键片段（示意）**

```ts
// 选择后：前端结果直接展开；后端结果标记搜索状态并触发加载
dispatch(setCurrentDictSearch({ searchOption: fullOption, targetParentPath, isActive: true }))
requestManager.abortController(targetParentPath)
dispatch(setExpandedKeys([...new Set([...shouldExpandKeys, ...expandedKeys])]))
```

---

## 常见误区与边界

- **“分页也能用方案一吧？”**
  不建议。搜索结果不在首分页会导致父子错位，难以补齐。

- **“方案二太复杂了”**
  是，但复杂度换来的是**连续交互稳定性**和**可扩展性**（权限/右键/编辑/分页）。
- **“清空搜索方案二更慢”**
  是设计取舍：保持缓存/展开一致，避免后续异常。

---

## 发布与协作（行动项）

- **代码对齐**：统一术语与状态名（见下术语表）
- **基准脚本**：把本页数据规模与脚本放到 `/bench/tree-search-bench`

---

## 术语表（用词统一）

- `fromSearch`: 搜索生成的节点标记

- `treeNodeChildrenMap`: 子节点缓存映射

- `requestManager`: 并发请求中断/去重管理

- “搜索目标父节点”：需要用**搜索接口**而非常规接口加载其子节点的父节点
