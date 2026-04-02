# Vue3 消息流场景虚拟列表调研（双向滚动 + 动态高度 + 锚点跳转）

> 调研时间：2026-04-02（UTC）
>
> 目标能力：
> 1) 上下双向加载历史/新消息；
> 2) 消息气泡动态高度；
> 3) 锚点跳转（按消息 ID / index 定位）；
> 4) 在消息持续插入时尽量保持滚动稳定。

## 候选组件结论（先看结论）

- **首推（工程可控）**：`@tanstack/vue-virtual`
  - 原因：Headless，可精细控制滚动锚点、动态测量、回填历史时的滚动补偿；复杂度高但上限最高。
- **次选（上手快）**：`vue-virtual-scroller`（Vue3 线）
  - 原因：`DynamicScroller` 对动态高度支持直接，开发速度快；但消息流“严格锚点稳定”通常要自己补滚动逻辑。
- **专项可选（聊天特化）**：`@virtual-list/vue`（VList）
  - 原因：官网明确有 `Reverse Mode` 与 Messaging 示例，聊天语义友好；生态体量相对更小，团队需接受社区规模。

## 能力对比

| 组件 | Vue3 支持 | 动态高度 | 双向消息流 | 锚点跳转 | 接入成本 | 成熟度判断 |
|---|---|---|---|---|---|---|
| `@tanstack/vue-virtual` | ✅ 官方 Vue 适配器 | ✅ `measureElement` / `resizeItem` | ✅（需自己实现 prepend/append 策略） | ✅ `scrollToIndex` / `scrollToOffset` | **高**（Headless） | **高**（下载量/活跃度高） |
| `vue-virtual-scroller` | ✅ Vue3 线 | ✅ `DynamicScroller` | ✅（可做，但需业务补偿） | ✅（可通过实例方法/索引定位实现） | **中** | **高**（历史久、社区大） |
| `@virtual-list/vue` | ✅ | ✅（官网强调 variable sizes + DOM measurement） | ✅（官网有 Reverse Mode） | ✅（官网示例含 scroll-to） | **中-低**（聊天特性直接） | **中**（功能成熟，社区规模较小） |

## 接入成本拆解（你最关心的）

### 1) `@tanstack/vue-virtual`（高成本，高控制）

- 你要自己管：容器、占位层、每行定位、测量回调、滚动补偿。
- 但好处是：任何复杂产品需求（未读分割线、消息分组、富媒体重排、精确锚点）都能做。
- 典型工期：
  - POC：0.5~1.5 天
  - 可上线：3~7 天（含各种滚动稳定边界）

### 2) `vue-virtual-scroller`（中成本，快速交付）

- `DynamicScroller` 原生处理“未知高度”，很适合消息气泡。
- 你主要补：
  - 顶部加载历史时的视觉不跳动（保存/恢复锚点）
  - 持续新消息插入时“在底部则跟随，不在底部则提示未读”
- 典型工期：
  - POC：0.5~1 天
  - 可上线：2~5 天

### 3) `@virtual-list/vue`（中低成本，聊天语义友好）

- 官网直接给出了 Messaging / Reverse Mode 示例，语义贴近聊天。
- 如果你的团队可以接受相对小一些的社区规模，落地速度会很快。
- 典型工期：
  - POC：0.5~1 天
  - 可上线：2~4 天

## 成熟度评估（2026-04-02 采样）

> 说明：这里把“下载量 + GitHub 活跃 + 最近发版”作为工程侧可用性信号，不等于绝对优劣。

| 包名 | npm 最近30天下载 | npm 最近修改时间 | GitHub stars | 最近 push |
|---|---:|---|---:|---|
| `@tanstack/vue-virtual` | 6,610,148 | 2026-03-16 | 6,793（TanStack/virtual） | 2026-03-26 |
| `vue-virtual-scroller` | 1,799,806 | 2026-03-31 | 10,648（Akryum/vue-virtual-scroller） | 2026-03-31 |
| `@virtual-list/vue` | 56,116 | 2024-12-24 | 21（phphe/virtual-list） | 2024-12-24 |
| `vue-virtual-scroll-grid` | 22,635 | 2023-10-31 | 325 | 2023-10-31 |

## 关键证据（官方文档/仓库）

- TanStack Virtual API：`measureElement`、`scrollToIndex`、`scrollToOffset`、`shouldAdjustScrollPositionOnItemSizeChange`。  
  https://tanstack.com/virtual/latest/docs/api/virtualizer
- TanStack Vue 适配器：`@tanstack/vue-virtual` 是 core 的 Vue wrapper。  
  https://tanstack.com/virtual/latest/docs/framework/vue/vue-virtual
- Vue Virtual Scroller 首页：提供 `DynamicScroller`，用于未知高度测量。  
  https://vue-virtual-scroller.netlify.app/
- VList 首页：明确包含 `Reverse Mode`（聊天从底部开始、prepend 历史、auto-scroll）、variable sizes、scroll-to 示例。  
  https://vlist.dev/
- Vue Virtual Scroller 仓库：Vue3 线、最近版本 `v2.0.0 (2026-03-31)`。  
  https://github.com/Akryum/vue-virtual-scroller

## 推荐落地方案（按你这个需求）

1. **主方案：TanStack（推荐）**
   - 当你们消息类型复杂（图片、引用回复、折叠块、异步渲染）且对“滚动不跳”要求高。
2. **快交付方案：Vue Virtual Scroller**
   - 当你们希望快速上线，先满足 80% 聊天体验，再逐步打磨边界。
3. **聊天特化方案：VList**
   - 当你们希望用更少样板代码拿到 reverse + prepend 场景。

---

## Demo 1：`@tanstack/vue-virtual`（推荐主方案）

文件：`demos/TanStackChatDemo.vue`

特点：
- 支持 prepend 历史消息并保持视口稳定。
- 用 `virtualizer.measureElement` 支持动态高度。
- 支持按消息 ID 锚点跳转（内部转 index + `scrollToIndex`）。

## Demo 2：`vue-virtual-scroller`（快速交付）

文件：`demos/VueVirtualScrollerChatDemo.vue`

特点：
- `DynamicScroller + DynamicScrollerItem` 自动测量气泡高度。
- 提供“跳转到消息 ID”的锚点定位。
- 顶部加载历史时做滚动补偿，减少抖动。

## Demo 3：混合架构（生产上很常见）

文件：`demos/HybridRecentAndHistoryDemo.vue`

特点：
- 最近 N 条（如 200）保持真实 DOM（编辑、选择、复制、动画体验更稳）。
- 历史区走虚拟列表（大幅降低 DOM 压力）。
- 支持按消息 ID 优先在最近区锚点，否则切换到历史虚拟区再定位。

