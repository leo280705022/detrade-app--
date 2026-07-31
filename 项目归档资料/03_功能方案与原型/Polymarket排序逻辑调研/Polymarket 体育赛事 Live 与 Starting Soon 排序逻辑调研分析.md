# Polymarket 体育赛事 Live 与 Starting Soon 排序逻辑调研分析

**调研日期**: 2026-07-31  
**调研范围**: Polymarket 平台体育赛事页面排序机制、API 技术实现、跨分类排序对比  
**调研目的**: 为 Live 和 Starting Soon 排序优化提供竞品参考依据

---

## 一、调研概述

本次调研聚焦 Polymarket 平台在体育赛事（Sports）分类下 Live（进行中）和 Starting Soon（即将开始）预测事件的排序逻辑，同时横向对比其他预测事件分类（Politics、Crypto、Pop Culture、Business）的排序机制，评估体育类排序规则的可复用性。

调研方法包括：直接访问 Polymarket 前端页面观察排序行为、分析 Polymarket Gamma API 官方文档中的排序参数定义、以及跨分类页面的横向对比。

---

## 二、体育赛事 Live 排序逻辑

### 2.1 页面结构

Polymarket 体育赛事页面（`/zh/sports/live`）采用**分区式布局**，将事件按比赛状态分为独立区块：

| 区块 | 内容 | 位置 |
|------|------|------|
| **Live（进行中）** | 已开始但尚未结束的比赛 | 页面顶部，第一区块 |
| **Starting Soon（即将开始）** | 尚未开始但临近开赛时间的比赛 | 紧随 Live 区块之后 |
| **Upcoming（即将到来）** | 更晚开始的比赛 | 在 Starting Soon 之后 |
| **已结束** | 比赛已结束待结算 | 页面底部或不展示 |

这种分区设计是**体育分类独有的**，其他分类页面均无此设计。

### 2.2 Live 区块排序规则

Live 区块内的事件按**赛事时间/临近度**排序，具体规则如下：

**主排序维度：开赛时间（startTime）升序**
- 最早开赛的比赛排在最前面
- 逻辑：已经进行了一段时间的比赛，距离结束更近，结算更快，用户关注度更高

**辅助排序维度（同时间或同联赛内）：**

| 优先级 | 维度 | 说明 |
|--------|------|------|
| 1 | 比赛状态 | 进行中 > 即将开始 |
| 2 | 开赛时间（startTime） | 越早开赛越靠前（升序） |
| 3 | 联赛/赛事重要性 | 通过 `featuredOrder`（运营人工权重）控制，数值越小越靠前 |
| 4 | 24h 交易量（volume24hr） | 交易量高的比赛在同层级内优先展示 |

**关键发现**：Live 区块**不按交易量作为主排序维度**。这与首页和其他分类页面的"Most Active"默认排序有本质区别。体育赛事的核心排序逻辑是**时间驱动**而非**交易量驱动**。

### 2.3 Starting Soon 区块排序规则

Starting Soon 区块的排序逻辑：

**主排序维度：开赛时间（startTime）升序**
- 距离当前时间最近、即将开赛的比赛排在最前面
- 给用户最紧迫的"即将开始"感知

**辅助排序维度：**

| 优先级 | 维度 | 说明 |
|--------|------|------|
| 1 | 开赛时间（startTime） | 最近开赛的排在最前（升序） |
| 2 | 联赛重要性（featuredOrder） | 同等时间条件下，运营标记的重要赛事优先 |
| 3 | 24h 交易量（volume24hr） | 作为最终 tiebreaker |

### 2.4 前端排序切换选项

体育赛事页面提供排序切换功能（通过 "sliders" 按钮触发），支持以下排序模式：

| 排序模式 | 对应 API 参数 | 说明 |
|---------|-------------|------|
| **Most Active** | `order=volume24hr&ascending=false` | 按 24h 交易量降序 |
| **Ending Soonest** | `order=endDate&ascending=true` | 按结束时间升序（即将截止优先） |
| **Newest** | `order=createdAt&ascending=false` | 按创建时间降序（最新优先） |

**默认模式**为按赛事时间排序（即上述 Live/Starting Soon 的时间驱动排序）。切换到 Most Active 后，Live 和 Starting Soon 的分区结构可能被打破，事件按纯交易量排列。

---

## 三、API 技术实现分析

### 3.1 核心排序参数

Polymarket Gamma API 使用两个通用参数控制排序：

| 参数 | 类型 | 说明 |
|------|------|------|
| `order` | string | 排序字段，支持逗号分隔的多字段排序 |
| `ascending` | boolean | 排序方向。`true` = 升序，`false` = 降序 |

### 3.2 可用的排序字段

**Event 级别排序字段：**

| 字段 | 类型 | 体育赛事适用性 | 说明 |
|------|------|-------------|------|
| `startTime` | datetime | 高 | 赛事开始时间，Live/Starting Soon 的核心排序字段 |
| `endDate` | datetime | 高 | 赛事结束时间，用于 "Ending Soonest" 排序 |
| `volume24hr` | number | 高 | 24 小时交易量，用于 "Most Active" 排序 |
| `volume1wk` | number | 中 | 7 天交易量 |
| `volume` | number | 中 | 总交易量 |
| `liquidity` | number | 中 | 总流动性 |
| `featuredOrder` | integer | 高 | 运营人工排序权重，数值越小越靠前 |
| `openInterest` | number | 中 | 未平仓合约量 |
| `createdAt` | datetime | 低 | 创建时间，用于 "Newest" 排序 |
| `competitive` | number | 低 | 竞争度指标 |

**Market 级别排序字段：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `volumeNum` | number | 总交易量（数值型） |
| `volume24hr` | number | 24 小时交易量 |
| `liquidityNum` | number | 流动性（数值型） |
| `curationOrder` | integer | 人工策展排序权重 |
| `score` | integer | 市场评分 |

### 3.3 体育赛事专用过滤参数

API 为体育赛事提供了额外的过滤维度，这些参数在非体育分类中不可用：

| 参数 | 类型 | 说明 |
|------|------|------|
| `live` | boolean | 过滤正在进行的赛事 |
| `event_date` | datetime | 按赛事日期过滤 |
| `event_week` | integer | 按赛事周过滤 |
| `start_time_min` | datetime | 赛事开始时间下限 |
| `start_time_max` | datetime | 赛事开始时间上限 |
| `game_id` | array | 按具体比赛 ID 过滤 |
| `series_id` | array | 按系列赛事 ID 过滤 |
| `tag_id` | array | 按运动类型标签过滤（如 NBA、EPL 等） |

### 3.4 体育赛事实时状态字段

Event 对象包含以下实时状态字段，可用于客户端二次排序：

| 字段 | 类型 | 说明 |
|------|------|------|
| `live` | boolean | 是否正在进行中 |
| `ended` | boolean | 是否已结束 |
| `score` | string | 当前比分 |
| `elapsed` | string | 已进行时间 |
| `period` | string | 当前节/半场 |
| `gameStatus` | string | 比赛状态描述 |

### 3.5 推断的 API 调用模式

基于前端行为和 API 参数分析，Polymarket 体育 Live 页面的 API 调用模式推断如下：

**Live 区块：**
```
GET /events/keyset?live=true&tag_id={sport_tag_id}&order=startTime&ascending=true&closed=false
```

**Starting Soon 区块：**
```
GET /events/keyset?closed=false&tag_id={sport_tag_id}&start_time_min={now}&start_time_max={now+24h}&order=startTime&ascending=true
```

**Most Active 排序（切换后）：**
```
GET /events/keyset?closed=false&tag_id={sport_tag_id}&order=volume24hr&ascending=false
```

### 3.6 多层排序架构

Polymarket 的排序不是单一维度的，而是多层架构：

| 层级 | 机制 | 控制方 | 说明 |
|------|------|--------|------|
| **API 查询级** | `order` + `ascending` 参数 | 前端/开发者 | 决定事件列表的主排序 |
| **运营策展级** | `featuredOrder` 字段 | 运营团队 | 手动设置高优先级赛事的展示位置 |
| **事件内市场级** | Event 的 `sortBy` 字段 | 系统/模板 | 控制一个事件内多个盘口市场的展示顺序 |
| **市场级人工排序** | `curationOrder` 字段 | 运营团队 | 市场级别的人工策展权重 |

---

## 四、其他预测事件类型排序对比

### 4.1 各分类默认排序方式

| 分类 | 默认排序 | Live 独立区块 | Starting Soon 区块 | 排序切换选项 |
|------|---------|-------------|-------------------|------------|
| **Sports（体育）** | 赛事时间/临近度 | 有 | 有 | Most Active / Ending Soonest / Newest |
| **Politics（政治）** | Most Active（交易量） | 无 | 无 | 无显式排序切换 |
| **Crypto（加密货币）** | Most Active（交易量） | 无 | 无 | 按事件类型筛选 Tab（涨/跌/价格区间等） |
| **Pop Culture（流行文化）** | Most Active（交易量） | 无 | 无 | 无显式排序切换 |
| **Business/Finance（商业）** | Most Active（交易量） | 无 | 无 | 按标的类型筛选 Tab（S&P 500/股票/指数等） |

### 4.2 关键差异分析

**共性：**
- 所有分类默认均按某种"活跃度"或"时效性"排序
- 事件卡片均显示赔率/概率，不直接显示交易量或参与人数
- 均提供筛选（filters）和关注列表（watchlist）切换按钮

**差异：**

| 差异维度 | 体育 | 其他分类 |
|---------|------|---------|
| **排序主维度** | 时间驱动（开赛时间） | 交易量驱动（Most Active） |
| **状态分区** | 按比赛状态分区（Live → Starting Soon → Upcoming） | 不分区，所有事件混排 |
| **实时数据** | 展示比分、进行时间等实时数据 | 无实时状态概念 |
| **排序切换** | 提供多种排序模式切换 | 无排序切换，仅支持筛选 |
| **时间敏感性** | 极高（比赛有明确的开始/结束时间） | 较低（事件截止日期相对模糊） |

### 4.3 为什么体育分类需要独特排序

体育赛事与其他预测事件有本质区别，这决定了排序逻辑不能简单复用：

1. **强时间约束**：体育赛事有精确的开赛时间和结束时间，事件的生命周期是确定性的。政治事件、文化事件的"截止时间"则相对模糊。

2. **实时状态变化**：比赛进行中，比分、时间、局势持续变化，用户需要快速找到正在进行的关键比赛。其他分类的事件不存在"进行中"的概念。

3. **时效性价值递减**：一场比赛一旦结束，其预测市场立即失去交易价值。而政治类事件在截止前始终保持交易价值。

4. **用户心智模型**：体育用户期望看到"现在在打什么 → 马上要打什么 → 今天还有什么"，这是时间线思维。其他分类用户更关心"什么最热门"，这是热度思维。

---

## 五、体育排序规则的可复用性评估

### 5.1 可直接复用的规则

| 规则 | 适用场景 | 复用条件 |
|------|---------|---------|
| **时间驱动排序（startTime 升序）** | 任何有明确时间属性的事件 | 事件必须有精确的开始/截止时间 |
| **状态分区设计（Live → Starting Soon → Upcoming）** | 有"进行中"概念的事件类型 | 事件需具备实时状态字段 |
| **多排序模式切换** | 所有分类通用 | 需前端支持排序切换 UI |
| **运营策展权重（featuredOrder）** | 所有分类通用 | 需运营后台支持手动排序 |

### 5.2 需要适配后复用的规则

| 规则 | 体育场景 | 其他场景适配 |
|------|---------|------------|
| **Live 区块定义** | 比赛已开始未结束 | 需定义各类事件的"Live"含义（如：投票进行中、拍卖进行中） |
| **Starting Soon 定义** | 距开赛时间 < N 小时 | N 值需按事件类型调整（政治事件可能需要 24-48h，加密事件可能 1-2h） |
| **交易量作为辅助排序** | 同时间条件下按 volume24hr 排序 | 可作为所有分类的通用 tiebreaker |
| **实时数据展示** | 比分、进行时间 | 需按事件类型定义展示内容（如：投票进度、拍卖倒计时） |

### 5.3 不适合复用的规则

| 规则 | 原因 |
|------|------|
| **按联赛分组** | 联赛概念是体育独有的，其他分类没有等价结构 |
| **比分/盘口排序** | 体育特有的数据维度 |
| **赛事时间窗口过滤（event_date/event_week）** | 强依赖体育赛事的日程结构 |

### 5.4 推荐的通用排序框架

基于以上分析，建议采用**分层排序框架**，可按事件类型灵活配置：

```
排序优先级 = [
  第 1 层: 事件状态分区（Live → Starting Soon → Upcoming → 已结束）
  第 2 层: 时间维度（startTime 升序 / endDate 升序，按分区配置）
  第 3 层: 运营权重（featuredOrder 升序）
  第 4 层: 活跃度（volume24hr 降序）
]
```

各分类的配置差异：

| 分类 | 第 1 层（状态分区） | 第 2 层（时间维度） | 第 3 层（运营权重） | 第 4 层（活跃度） |
|------|------------------|------------------|------------------|------------------|
| **体育** | 启用（Live/Starting Soon/Upcoming） | startTime 升序 | 启用 | 启用 |
| **政治** | 可选（按投票/决议状态） | endDate 升序（截止优先） | 启用 | 启用 |
| **加密** | 不启用（无明确状态） | 不适用 | 启用 | 启用（主排序） |
| **流行文化** | 不启用 | 不适用 | 启用 | 启用（主排序） |
| **商业** | 不启用 | endDate 升序 | 启用 | 启用（主排序） |

---

## 六、对我们产品的建议

### 6.1 Live 排序建议

1. **采用时间驱动作为体育 Live 的主排序**：按开赛时间升序排列，最早开赛的排在最前。这与 Polymarket 的做法一致，也符合用户对体育赛事的时间线心智模型。

2. **引入状态分区**：将事件按状态分为 Live → Starting Soon → Upcoming 三个区块，每个区块内独立排序。

3. **交易量作为辅助排序**：同一时间段内的比赛，按 24h 交易量降序排列，确保热门比赛获得更高曝光。

4. **预留运营干预能力**：通过 `featuredOrder` 类字段，允许运营团队手动提升特定赛事的展示位置。

### 6.2 Starting Soon 排序建议

1. **按距开赛时间升序排列**：最接近开赛的事件排在最前，制造紧迫感。

2. **定义"Starting Soon"的时间窗口**：建议体育类设为开赛前 2 小时内，其他类型可按需调整。

3. **展示倒计时信息**：在事件卡片上显示距开赛的倒计时，强化"即将开始"的视觉感知。

### 6.3 跨分类复用建议

1. **排序框架统一，配置差异化**：采用第五节提出的分层排序框架，各分类通过配置决定启用哪些层级和维度。

2. **非体育分类暂不引入状态分区**：政治、加密、流行文化等分类继续使用 Most Active（交易量）作为主排序，不引入 Live/Starting Soon 分区，避免增加不必要的复杂度。

3. **为未来扩展预留接口**：即使当前不启用状态分区，数据模型中也应预留 `live`、`startTime`、`ended` 等字段，为后续可能的扩展做准备。

---

## 七、总结

| 维度 | 核心发现 |
|------|---------|
| **体育 Live 排序** | 以开赛时间（startTime）升序为主排序，交易量为辅，运营权重为补充 |
| **体育 Starting Soon 排序** | 以距开赛时间升序为主排序，本质是 Live 排序在"未开始"事件上的延伸 |
| **与其他分类的核心差异** | 体育是时间驱动 + 状态分区，其他分类是交易量驱动 + 混排 |
| **可复用性** | 分层排序框架可复用，但状态分区和时间驱动排序仅适用于有时间属性的事件类型 |
| **API 实现** | 通过 `order`+`ascending` 通用参数 + 体育专用过滤参数（`live`、`start_time_min/max`）组合实现 |

---

## 附录：Polymarket Gamma API 排序相关参数速查

### Event 排序字段

| 字段 | 说明 | 排序示例 |
|------|------|---------|
| `startTime` | 赛事开始时间 | `order=startTime&ascending=true` |
| `endDate` | 赛事结束时间 | `order=endDate&ascending=true` |
| `volume24hr` | 24h 交易量 | `order=volume24hr&ascending=false` |
| `volume1wk` | 7 天交易量 | `order=volume1wk&ascending=false` |
| `volume` | 总交易量 | `order=volume&ascending=false` |
| `liquidity` | 总流动性 | `order=liquidity&ascending=false` |
| `featuredOrder` | 运营排序权重 | `order=featuredOrder&ascending=true` |
| `createdAt` | 创建时间 | `order=createdAt&ascending=false` |

### 体育专用过滤参数

| 参数 | 说明 | 示例 |
|------|------|------|
| `live` | 过滤进行中的赛事 | `live=true` |
| `event_date` | 按赛事日期过滤 | `event_date=2026-07-31` |
| `event_week` | 按赛事周过滤 | `event_week=31` |
| `start_time_min` | 开始时间下限 | `start_time_min=2026-07-31T00:00:00Z` |
| `start_time_max` | 开始时间上限 | `start_time_max=2026-07-31T23:59:59Z` |
| `game_id` | 按比赛 ID 过滤 | `game_id=[12345]` |
| `tag_id` | 按运动类型标签过滤 | `tag_id=[745]` |

### 典型 API 调用示例

```bash
# 获取体育 Live 事件（按开赛时间升序）
GET /events/keyset?live=true&tag_id={sport_tag_id}&order=startTime&ascending=true&closed=false&limit=50

# 获取 Starting Soon 事件（未来 24 小时内开赛）
GET /events/keyset?closed=false&tag_id={sport_tag_id}&start_time_min={now}&start_time_max={now+24h}&order=startTime&ascending=true&limit=50

# 获取 Most Active 事件（按 24h 交易量降序）
GET /events/keyset?closed=false&tag_id={sport_tag_id}&order=volume24hr&ascending=false&limit=50

# 获取即将截止的事件（按结束时间升序）
GET /events/keyset?closed=false&tag_id={sport_tag_id}&order=endDate&ascending=true&limit=50
```
