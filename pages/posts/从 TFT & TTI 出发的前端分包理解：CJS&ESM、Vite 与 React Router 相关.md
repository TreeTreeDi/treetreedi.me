---
title: 从 TFT & TTI 出发的前端分包理解：CJS&ESM、Vite 与 React Router 相关
date: 2025-11-11T10:50:00.000+00:00
lang: zh
duration: 25min
---

[[toc]]

> 用 **用户体验指标**（TFT/TTI）来反推工程方案，而不是从工具出发。本文给出一条"指标→策略→落地→验收"的最短路径。

---

- **TFT（Time to First Task）**：用户首次可完成关键任务（点击、搜索、切换语言等）的时间。可把它理解为“业务可用”的阈值。

- **TTI（Time to Interactive）**：页面达到完整交互能力的时间。它通常晚于 TFT，但不能晚太多，否则“可用但卡顿”。

> **分包的本质**： 让 **首屏路径** 只携带“达成 TFT 必要”的脚本和资源； 把其他功能延后到“靠近交互发生时”再加载，以压低 TTI 并避免主线程拥堵。

---

## 1. 为什么分包能改善 TFT/TTI

分包降低三类成本：

1. **下载**（减少初始字节，配合 HTTP/2 多路复用）；

2. **解析与编译**（JS 的 parse/compile 开销与体积近似线性）；

3. **执行**（初始化大型库/运行副作用）。

只要把“非首屏任务”对应的模块移动到**路由懒加载**或**用户行为触发的 import()**，就能让 **TFT 更快、TTI 更稳**。

---

## 2. 模块规范选择：CJS vs ESM（站在指标侧的取舍）

| 维度                | CJS (`require`)  | ESM（`import`/`import()`） |
| ------------------- | ---------------- | -------------------------- |
| 可静态分析          | 弱（运行时决定） | 强（构建期可解）           |
| 代码分割            | 间接/需黑魔法    | `import()` 天然分割点      |
| Tree-shaking        | 受限             | 友好                       |
| 对 TFT/TTI 的可控性 | 难控             | 易控                       |

**结论**：追求 TFT/TTI 的团队，优先 **ESM + \*\*\*\***``。CJS 在动态路径、按需加载场景下会阻碍分包的可预测性。

---

## 3. 经典例子：i18n 按需加载，CJS 的坑与 ESM 的解法

**目标**：语言包按需加载，避免把全部字典塞进首屏 bundle（直接影响 TFT）。

### 3.1 CJS 的常见坑

```js
// 构建期不可枚举，易被整包打入或运行时报错
const dict = require(`./locales/${lang}.json`)
```

- 构建器无法静态分析，常退化为“全量打包”；

- 首屏字节暴涨 → TFT 上升。

### 3.2 ESM 的使用方式

```ts
// 方式 A：动态 import
export async function loadDict(lang: string) {
  return (await import(`./locales/${lang}.json`)).default
}

// 方式 B：import.meta.glob（推荐，可控且可枚举）
const dictMods = import.meta.glob('./locales/*.json', { eager: false })
export async function loadDictByLocale(locale: string) {
  const key = `./locales/${locale}.json`
  const mod = dictMods[key]
  return mod ? (await mod()).default : {}
}
```

> **效果**：以 `import()` 为分割点，语言包成为**独立 chunk**，只在语言切换或路由激活时加载。首屏更轻，TFT 更快。

---

## 4. Vite 的分包策略：先自动、后手动（面向指标调参）

### 4.1 自动分包（零配置）

- 每个 `import()` → 独立 chunk；

- 公共依赖自动提取共享块；

- 自动注入 `modulepreload` 提示，减少请求瀑布。

> **适用**：先让“路由懒加载 + 大组件懒加载”跑起来，通常能覆盖 80% 需求。

### 4.2 什么时候需要手动分包（`manualChunks`）

- **超大依赖**（echarts/xlsx/monaco 等）但非首屏必需：独立成块，避免拖慢 TFT；

- **跨路由共享的重包**：抽公共 chunk，防止多份重复；

- **稳定缓存策略**：低频更新的大包单独缓存，提升二次访问 TTI。

```ts
// vite.config.ts
import { defineConfig } from 'vite'

export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          react: ['react', 'react-dom'],
          echarts: ['echarts'],
          xlsx: ['xlsx'],
          editor: ['monaco-editor'],
        },
      },
    },
  },
})
```

> **度量导向**：
>
> - 手分包前后对比 **Main bundle 体积** 与 **Lighthouse bootup-time**，以是否改善 TFT/TTI 为准。

---

## 5. 路由分包 + 大模块分包：与 React Router 的协作

### 5.1 路由层懒加载

```tsx
import { createBrowserRouter } from 'react-router-dom'

export const router = createBrowserRouter([
  { path: '/', lazy: () => import('./pages/Home') },
  { path: '/about', lazy: () => import('./pages/About') },
])
```

### 5.2 页面内“大组件”二次懒加载

```tsx
import { lazy, Suspense } from 'react'

const BigChart = lazy(() => import('./components/BigChart'))
export default function Dashboard() {
  return (
    <Suspense fallback={<div>Loading chart...</div>}>
      <BigChart />
    </Suspense>
  )
}
```

### 5.3 为什么能“正确”分包（协作链）

```
用户代码:      lazy: () => import('./About')
           ↓
Vite 构建:     识别 import() → 生成独立 chunk
           ↓
manifest.json:  记录 chunk 依赖关系
           ↓
RR Vite Plugin: 读 manifest → 生成路由 manifest → 注入 HTML
           ↓
Router 运行时:  导航触发 lazy() → 浏览器加载 chunk
           ↓
浏览器:        动态插入 <script type="module"> → React 渲染
```

**关键点**：

- Rollup 将 `import()` 当作分割点；

- Router 在**导航阶段**拉取资源，避免渲染锁；

- 缓存与 CSS 去重降低后续导航的 TTI。

---

## 6. 手动分包怎么处理

- **必须拆（红灯）**：单包 ≥ 300KB（gzip 后仍大）且非首屏必需；跨 ≥3 路由共享；

- **建议拆（黄灯）**：图表/编辑器/富文本/地图/PDF 渲染器；

- **不必拆（绿灯）**：< 50KB 的工具包，交给自动分包。

> 决策依据：拆分后 **Main bundle 体积下降** 与 **bootup-time 降低** 能被数据证明。

---

## 7. Vite 自动分包的内部逻辑与最佳实践

- `import()` = 分割点；公共依赖自动抽取；

- 对动态集合用 `import.meta.glob` 显式声明边界（**可枚举才好分**）；

- 使用 rollup 可视化（如 `rollup-plugin-visualizer`）观察热路径与块体积；

- 预加载策略：在**即将导航**或**悬停**时触发 `lazy()` 进行轻预取，平衡 TFT/TTI 与网络消耗。

**实践清单**：

- 路由全部 `lazy()`；

- 大组件二次懒加载；

- `import.meta.glob` 管动态资源与 i18n；

- 仅对“大且共享”的依赖手动分包；

- 构建后看块图谱，再精细化；

- 用指标兜底（见下）。

---

## 8. 验收与守护：把“体验阈值”拉进 CI（预算）

### 8.1 包体预算（size-limit）

```bash
pnpm add -D size-limit @size-limit/file
```

```json
{
  "size-limit": [
    { "name": "Main bundle", "path": "dist/assets/*.js", "limit": "180 KB" },
    { "name": "Echarts chunk", "path": "dist/assets/echarts-*.js", "limit": "300 KB" }
  ],
  "scripts": { "build": "vite build", "size": "size-limit" }
}
```

### 8.2 Boot-up 预算（Lighthouse 的 `bootup-time`）

什么是 Boot-up

- 它是 Lighthouse 的一项性能审计，**统计页面加载阶段主线程用于 JavaScript 的解析、编译与执行总时长**（不含网络传输时间），单位毫秒。

- 具体包含：脚本的 **parse/compile/evaluate**、模块初始化、顶层副作用、同步任务队列等；会按脚本/库来源进行归因。

- 与体验关系：`bootup-time` 高 → 主线程被脚本占用时间长，**长任务（Long Tasks）**增多，直接推迟 **TFT/TTI**。

- 常见成因：首屏 bundle 过大、顶层同步初始化（图表/编辑器/富文本）、多 polyfill/重复依赖、无控制的全局副作用。

- 优化抓手：**路由懒加载**、**组件级二次懒加载**、`import.meta.glob` 管动态集合、`manualChunks` 抽出大库、移除顶层副作用、将重计算放 **Web Worker**、开启 **tree-shaking** 与按需引入、以分摊启动成本。

```bash
pnpm add -D @lhci/cli serve
```

```json
{
  "ci": {
    "collect": {
      "staticDistDir": "./dist",
      "numberOfRuns": 3,
      "settings": {
        "formFactor": "desktop",
        "screenEmulation": { "disabled": false },
        "throttling": { "rttMs": 40, "throughputKbps": 10240, "cpuSlowdownMultiplier": 1 },
        "disableStorageReset": false
      }
    },
    "assert": {
      "assertions": {
        "bootup-time": ["error", { "maxNumericValue": 800 }],
        "mainthread-work-breakdown": ["warn", { "maxNumericValue": 1200 }],
        "total-byte-weight": ["error", { "maxNumericValue": 350000 }],
        "unused-javascript": ["warn", { "maxNumericValue": 80000 }]
      }
    }
  }
}
```

### 8.3 GitHub Actions 来检查指标

```yaml
name: Budgets
on:
  pull_request: {branches: [main]}
  push: {branches: [main]}
jobs:
  check-budgets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Use Node
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: pnpm
      - name: Install deps
        run: |
          corepack enable
          pnpm i --frozen-lockfile
      - name: Build
        run: pnpm build
      - name: Check bundle size (size-limit)
        run: pnpm size
      - name: Run Lighthouse CI (static dist)
        run: npx lhci autorun --config=lighthouserc.json
```

> 若页面需接口才能跑：改用 `collect.url` 对临时预览地址跑 `lhci collect`。

---

## 9. 复盘：把“工程动作”映射回指标

- **路由懒加载** → 降 Main bundle → 降下载与 parse → **TFT 下降**；

- **大组件二次懒加载** → 推迟初始化重库 → **bootup-time 下降** → **TTI 更早**；

- **手动抽公共大块** → 避免重复体积 + 稳定缓存 → **二次访问更快**；

- **i18n 按需** → 语言切换体验稳定，同时不污染首屏；

- **CI 预算** → 防止回归；一旦 PR 让 TFT/TTI 退步，立刻可见。
