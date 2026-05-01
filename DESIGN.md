# 《司辰之书》事件攻略可视化 — 设计说明书

## 概览

这是一个针对《Book of Hours》（司辰之书）DLC HOL（House of Light）事件系统的静态网页攻略工具。核心思路是：把游戏原始 JSON 数据抽取成一份结构化的 `data.js`，再由一个无依赖的单页 HTML 渲染成可交互的攻略界面。

```
raw/weatherFactory/raw/gamecontent/    ← 英文游戏数据（只读）
raw/weatherFactory/raw/gamecontent-cn/ ← 中文本地化（只读）
          ↓  scripts/build_incidents_graph.py
exports/incidents-graph/data.js        ← 内嵌的全量数据（window.GRAPH_DATA）
exports/incidents-graph/index.html     ← 渲染 + 交互逻辑（单文件 SPA）
```

---

## 一、数据构建脚本 `build_incidents_graph.py`

脚本读取多个游戏 JSON 文件，组装成一个统一的图结构对象，写出 `data.json` 和 `data.js`（两者内容相同，后者用 `window.GRAPH_DATA = {...}` 包裹，规避 `file://` 协议下 `fetch` 被拦截的问题）。

### 1.1 读取的源文件

| 源文件 | 用途 |
|--------|------|
| `elements/incidents_n.json` (EN+CN) | 事件元素（名称、描述、aspects） |
| `decks/incidents_decks.json` (EN) | 哪些事件在 `incidents` / `incidents.numa` 牌堆 |
| `elements/visitors.json` (CN) | 访客中文名 |
| `recipes/visitors_correspondence_auctions.json` (EN) | `visitor.for.*` 召唤配方；`incident.setup/retire` 流程 |
| `recipes/talk_4b_visitors_specific_incidents.json` (EN+CN) | 访客×事件对话（引言/失败/成功） |
| `elements/DLC_HOL_ns.json` (EN+CN) | HOL lore 状态（n.* 元素） |
| `recipes/DLC_HOL_patch_resurrect_incidents.json` (EN) | incident → n-state 映射 |
| `recipes/DLC_HOL_nrs.json` (EN+CN) | nxi/nxh/nxs 配方 |
| `elements/tomes.json` (CN) | 书籍 ID → 中文名（t.*） |
| `elements/skills_r.json` (CN) | 阅读物 ID → 中文名（r.*） |

### 1.2 输出数据结构 `window.GRAPH_DATA`

```js
{
  incidents: [           // 29 条：事件列表
    { id, label_cn, label_en, desc_cn, icon, achievement,
      deck,             // "incidents" | "incidents.numa" | null
      numatic,          // 是否 Numa 闰时事件
      mysteries }       // ["edge", "grail", ...]  relevance.* aspects
  ],

  visitors: [           // 24 位：出现在 talk_4b 里的访客
    { id, label_cn, desc_cn }
  ],

  pairs: [              // 136 条：访客 × 事件配对
    { incident, visitor,
      intro_recipe, intro_text,          // talk.visitor.intro.*
      failure_recipe, failure_text,      // talk.visitor.incident.failure.*
      success_recipes, success_texts,    // {mystery → text}
      mysteries,                         // 该访客能完成的 mystery 列表
      rewards }                          // effects 里的数值奖励
  ],

  mysteries: [          // 13 个 Mystery（含非标准的 rose/scale/sky）
    { id, label_cn }
  ],

  visitor_for: {        // incident_id → { recipe, interests }
    "incident.hunt.changing": { recipe: "visitor.for.hunt.changing",
                                interests: ["edge", "grail"] }
  },

  setup_chain: {        // incident.setup / incident.retire 等核心配方摘要
    "incident.setup": { label, linked, deckeffects, effects }
  },

  n_states: [           // 137 个 HOL lore 状态
    { id, label_cn, label_en, desc_cn, icon,
      complete }        // 继承 _n.complete 时为 true（终态）
  ],

  n_transitions: [      // 135 条：n-state 状态转移
    { from, via,        // via = nx.* 动作 key
      to }              // 目标状态 id（字符串）
  ],

  incident_to_n: {      // 22 个有 HOL 长链的事件
    "incident.mob":          "n.debt",
    "incident.hunt.changing": "n.skin"
  },

  nrs: [                // 200 条：nxi + nxh 配方
    { id, kind,         // "nxi" | "nxh"
      n_state,          // 所在的 n-state
      visitor,          // 特定访客（或 null 表示通用）
      discipline,       // r.disciplines.* 需求（仅 nxh）
      label_cn, desc_cn, preface_cn,
      effects, aspects, reqs }
  ],

  nxs: [                // 83 条：nxs.* 询问配方
    { id, n_state, visitor, nx_action,
      acted_required,   // ["acted.arthur.hunt.changing.edge", ...]
      acted_excluded,
      label_cn, preface_cn, desc_cn,
      effects }
  ],

  items: {              // 354 条：物品/书籍 CN 名
    "t.dehorisbook3": "《司辰志·卷三》",
    "r.coil.chasm":   "主题：虬蟠与裂隙"
  }
}
```

---

## 二、前端渲染逻辑 `index.html`

单文件，无外部依赖。布局为左右两栏：左侧事件列表，右侧主内容区。

### 2.1 初始化流程

```
main()
 ├─ 加载 window.GRAPH_DATA（已内嵌）或 fetch data.json（服务器模式）
 ├─ 建索引：byId["i:*"] / byId["v:*"] / byId["m:*"] / byId["n:*"]
 ├─ 派生索引：
 │   data.n_outgoing[nid]          → 该 n-state 的所有出边
 │   data.nrs_by_state[nid]        → 该 n-state 的所有 nxi/nxh 配方
 │   data.nxs_by_state_action[k]   → key = "nid|nx.*" 的 nxs 列表
 └─ populateLists()：渲染侧边栏
```

### 2.2 单个事件的渲染流程

用户点击侧边栏 → `renderIncident(iid)` → `buildIncidentHTML(inc)`：

```
buildIncidentHTML
 ├─ 从 pairs 提取本事件的访客集合
 ├─ 计算 nInit（incident_to_n 查询）
 ├─ BFS 展开所有可达 n-state → reach 集合
 ├─ 分类访客：
 │   withChain → reach 内存在 nxs（访客匹配或通用）的访客
 │   noChain   → 其余访客
 ├─ buildFlowSVG()  → 顶部 SVG 流程图
 └─ 所有访客（withChain 优先）→ buildVisitorCard()
```

### 2.3 流程图 `buildFlowSVG`

固定宽度 640px 的 SVG，三列布局：

```
[  事件框  ] ──┬── [访客框] ──→──[结果态]
                │                  ↳ 更深后果1
                ├── [访客框] ──→──[结果态]
                └── [访客框]       (无后续)
```

| 元素 | 颜色 / 样式 |
|------|-------------|
| 事件框 | 蓝色填充 `#2e64a8`，显示中文名（8字截断）+ 英文名 |
| 主干竖线 | 浅蓝 `#8aadd0` |
| 访客框 | 浅蓝填充，可点击（跳转到对应卡片） |
| 箭头标注 | 橙红 `#c05020`，显示动作类型（捕禁/协助/终结…） |
| 结果态框 | 终态绿色 `#e8f5f0`，非终态蓝白 `#eef4fb` |
| 深层后果 | 结果框下方 `↳` 小字（最多3条） |

结果态查询逻辑：优先找访客专属的 `nxs`（`visitor === vid && n_state === nInit`），若无则退到通用 `nxs`（`!visitor && n_state === nInit`）。

### 2.4 访客卡片 `buildVisitorCard`

```
┌─────────────────────────────────────────────┐
│ 访客名  [需刃书] [需锻书]              ▶    │  ← 点击折叠/展开
│ 访客简介（desc_cn）                         │
├─────────────────────────────────────────────┤
│ [对话区 buildTalkSection]                   │  ← 折叠区
│   访客引言 ── intro_text                    │
│   尚未帮助时 ── failure_text                │
│   成功·刃 ── success_texts["edge"]          │
│ ─────────────────────────────────           │
│ [事件结束后] buildConsequenceHTML            │
│   询问：XX 配方名  [需先帮助该访客]          │
│   ↓ 协助                                    │
│   [n-state 卡 buildNStateBlock]             │
│     状态名 / 描述                           │
│     nxi 回顾对话                            │
│     nxh 深层结果（书名 → 新状态）           │
│     ▶ 展开后续分支                          │
└─────────────────────────────────────────────┘
```

有 `has-chain`（蓝框）vs 无链（灰框）视觉区分。

### 2.5 后果链渲染

**`buildConsequenceHTML`** — 找出所有 `reach` 内、访客匹配的 `nxs`，每条走 `buildNxsStep`。

**`buildNxsStep(nxs, incSuffix, vid)`**：
- 显示询问配方标签 + 可选条件标注（`acted_required` / `acted_excluded` 与 `incSuffix` 比对）
- 查 `n_transitions`：`from = nxs.n_state, via = nxs.nx_action` → `targetState`
- 递归进 `buildNStateBlock(targetState, depth=0, vid)`

**`buildNStateBlock(nid, depth, vid)`**：
- 渲染状态名 + 描述
- `nxi` 配方：访客回顾对话（`preface_cn` / `desc_cn`）
- `nxh` 配方：`访客名` + `itemLabel(key)` + `desc_cn` + `→ 新状态名`（通过 `nx.*` aspects 找 xtrigger 目标）
- 深层 `nxs` 折叠展开（最多递归到 `depth < 2`）

---

## 三、关键工具函数

| 函数 | 功能 |
|------|------|
| `nxLabel(nx)` | `nx.sealing` → `"捕禁"` 等中文动作名 |
| `mLabel(id)` | mystery id → 中文（刃/锻/圣杯…） |
| `itemLabel(key)` | `t.*` / `r.*` key → 中文书名/主题名，查 `data.items` |
| `nxhItemKey(r)` | 从 nxh 配方 id 剥去 `nxh.<n_state>.<visitor>.` 前缀，得物品 key |
| `splitTalkText(s)` | 游戏对话格式 `"文本 [提示]"` → `{ quote, hint }` |
| `truncate(s, n)` | SVG 内文字截断（加省略号） |
| `openCard(id)` | 展开并滚动到指定卡片，触发橙色高亮动画 |
| `getVisitorDeepStates(resultId, vid)` | 流程图用：查该结果态的 nxh 能达到的更深状态 |

---

## 四、游戏概念 → 数据字段映射

| 游戏概念 | 数据字段 |
|----------|----------|
| 事件（Incident） | `incident.*` 元素，`deck_of` 确定来源牌堆 |
| 访客匹配 | `visitor.for.<incident>` 配方的 `interest.*` 需求 |
| 帮助条件 | `pair.mysteries` — 访客需要对应 Mystery 书 |
| 对话文本 | `pair.intro_text` / `failure_text` / `success_texts` |
| Lore 状态 | `n.*` 元素，`n_transitions` 记录 xtrigger 转移 |
| 事件 → 初始状态 | `incident_to_n`（来自 `nx.resurrect.*` 配方 effects） |
| 询问回顾 | `nxs.*` 配方，带 `nx_action` 触发状态转移 |
| 先决条件 | `nxs.acted_required`：`acted.<v>.<incSuffix>.<mystery>` |
| 深层奖励 | `nxh.*` 配方，`aspects:{nx.*:1}` 触发 xtrigger，消耗书籍 |
| 终态 | `n_state.complete = true`（继承 `_n.complete`） |

---

## 五、已知边界情况

- **通用 nxs**（`visitor=null`）：部分事件（如"可恕之债" `incident.mob`）的 `nxs.n.debt.inconclusive` 无特定访客，适用于所有访客。`hasAny` 和流程图查询均用 `!nxs.visitor` 兼容。
- **无 HOL 链的事件**：`incident_to_n` 无条目的事件不生成流程图，只显示访客卡片。
- **Numa 事件**：逻辑与常规事件相同，仅在来源牌堆（`incidents.numa`）和标签配色上有区别。
- **对话文本格式**：游戏内 `startdescription` 可能含 `<sprite name=...>` 图标标签，脚本用 `◈` 替代；`[提示文字]` 方括号内容单独渲染为灰色小字。
