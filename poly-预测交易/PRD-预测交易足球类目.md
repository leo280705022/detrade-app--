# PRD：预测交易足球类目产品需求文档

## 文档信息
- **版本**: v2.0（基于 Polymarket 实际数据调研后重写）
- **日期**: 2026-05-13
- **状态**: 调研完成，初稿
- **目标**: 接入 Polymarket 预测交易数据，为 6 月世界杯上线足球类目

---

## 1. 产品概述

### 1.1 背景与调研结论

#### Polymarket 体育-足球类目调研总结

经过对 Polymarket 体育-足球类目的全面调研，核心结论如下：

**Polymarket 三层数据架构**（来源：[Polymarket 官方文档](https://docs.polymarket.us/concepts/events-and-markets)）：

```
Series（系列/联赛）       例如：FIFA World Cup 2026, Premier League 2025-26
  └── Event（事件/比赛）  例如：Man City vs Arsenal — May 14, 2026
        ├── Market 1: Moneyline（独赢）    "Will Man City win?"
        ├── Market 2: Spread（让分盘）     "Man City -1.5 goals?"
        ├── Market 3: Total（总进球）      "Total goals over 2.5?"
        └── Market 4+: Props（趣味盘）     "Haaland to score anytime?"
```

**Games（赛事）vs Props（趣味投注）的本质区别：**

| 维度 | Games（赛事页面） | Props（趣味投注页面） |
|------|-------------------|----------------------|
| **页面 URL** | `/sports/soccer/games` | `/sports/soccer/props` |
| **组织维度** | 以比赛 Event 为维度聚合展示 | 以市场类型/赛事为维度独立展示 |
| **核心市场** | Moneyline（独赢）、Spread（让分）、Total（总进球） | 球员表现、赛事冠军、特殊事件、比赛趣味盘 |
| **是否关联单场胜负** | 直接关联比赛输赢/比分 | 不直接关联/部分关联比赛结果 |
| **典型问题** | "Will Team A win?" / "Total goals over 2.5?" | "Golden Boot winner?" / "Mbappe to score?" |

**关键认知：Games 和 Props 不是互斥的市场分类，而是前端展示的两个 Tab。** 同一场比赛可以同时拥有 Games 类市场（Moneyline/Spread/Total）和 Props 类市场（球员进球、角球等）。Polymarket 前端通过两个页面 Tab 来区分——Games Tab 展示核心盘口，Props Tab 展示趣味市场。

#### 足球类目在 Polymarket 的规模

- **65 个实时足球预测市场**（截至2026年5月）
- **192+ 个世界杯活跃市场**（含冠军、小组赛、单场等）
- 世界杯冠军市场交易量已超 **$6.36亿**
- 2026 世界杯预计总交易量将超 **$20亿**（超级碗的 3 倍）
- 覆盖 **120+ 个足球联赛代码**（sport codes）

#### 覆盖的足球联赛（Polymarket 已支持）

| 代码 | 联赛 | 优先级 |
|------|------|--------|
| `fifa` | FIFA World Cup 2026（世界杯） | **P0 最高** |
| `epl` | English Premier League（英超） | P0 |
| `lal` | La Liga（西甲） | P0 |
| `sea` | Serie A（意甲） | P0 |
| `bun` | Bundesliga（德甲） | P0 |
| `fl1` | Ligue 1（法甲） | P0 |
| `ucl` | UEFA Champions League（欧冠） | P0 |
| `uel` | UEFA Europa League（欧联） | P1 |
| `mls` | Major League Soccer（美职联） | P1 |

### 1.2 目标
- **重点：为 2026 年 6 月世界杯提供完整的足球预测交易体验**
- 接入 Polymarket 真实的足球预测市场数据
- 按 Polymarket 原生市场结构组织产品（Series → Events → Markets）
- 覆盖赛前→赛中→赛后的全周期交易需求

### 1.3 目标用户
- 加密货币交易者（熟悉 Polymarket 模式）
- 足球爱好者（世界杯新增用户）
- 预测市场参与者

---

## 2. 产品架构

### 2.1 整体信息架构（对齐 Polymarket 三层模型）

```
足球预测交易
│
├── Series（联赛/赛季层级）
│   ├── FIFA World Cup 2026      ← 世界杯专题（重点）
│   ├── Premier League 2025-26
│   ├── La Liga 2025-26
│   ├── Champions League 2025-26
│   └── ... 其他联赛
│
├── Games Tab（比赛 - 以 Event 为维度）
│   │
│   └── Event（单场比赛）
│       ├── Moneyline（独赢 / 1X2）
│       │   "Will [Team A] win?" → Yes/No
│       │   三选项：主胜 / 平局 / 客胜
│       │
│       ├── Spread（让球盘）
│       │   "Will [Team A] win by 2+ goals?" → Yes/No
│       │
│       └── Total / Over-Under（总进球）
│           "Total goals over 2.5?" → Over/Under
│
└── Props Tab（趣味投注 - 多样化预测）
    │
    ├── Player Props（球员表现）
    │   ├── Anytime Goalscorer（任意时间进球）
    │   ├── First Goalscorer（首球射手）
    │   ├── Player Shots on Target（射正次数）
    │   ├── Player Assists（助攻）
    │   └── Player Cards（球员吃牌）
    │
    ├── Game Props（比赛趣味盘 - 关联比赛但不看胜负）
    │   ├── Both Teams to Score（双方是否都进球）
    │   ├── Total Corners（总角球数）
    │   ├── Total Cards（总黄牌/红牌数）
    │   ├── Correct Score（正确比分）
    │   ├── Will there be a Penalty?（是否有点球）
    │   └── Goal in First 15 Minutes?（前15分钟进球）
    │
    └── Tournament/Season Props（赛事/赛季盘 - 期间性）
        ├── Outright Winner（冠军）
        ├── Golden Boot（金靴奖）
        ├── Tournament MVP（最佳球员）
        ├── Group Winner（小组第一）
        ├── To Qualify from Group（小组出线）
        ├── To Reach Final（进入决赛）
        ├── Top 4 Finish（联赛前四）
        ├── Relegation（降级）
        └── Stage of Elimination（被淘汰阶段）
```

### 2.2 市场类型详解（对标 Polymarket sports_market_types）

| 市场类型 | Polymarket API 字段 | 说明 | 足球特有 |
|----------|---------------------|------|----------|
| **Moneyline** | `moneyline` | 独赢盘，选胜者 | 含平局选项（1X2） |
| **Spread** | `spread` | 让分盘 | 让球数（如-1.5球） |
| **Total / Over-Under** | `over_under` | 总分盘 | 总进球数 |
| **Player Props** | 归入 props 类 | 球员个人表现 | 进球/助攻/吃牌等 |
| **Game Props** | 归入 props 类 | 比赛事件趣味盘 | 角球/双方进球等 |
| **Tournament Props** | 归入 props 类 | 赛事级别预测 | 冠军/金靴等 |
| **Futures** | 归入 props 类 | 远期预测 | 赛季冠军等 |

---

## 3. 功能模块详述

### 3.1 Games Tab（比赛预测）

#### 3.1.1 页面布局

比赛以 Event（事件）为维度展示，每个比赛卡片包含该场比赛下的三种核心市场：

```
┌──────────────────────────────────────────┐
│ 🇪🇸 La Liga  ·  明天 01:00               │
│                                          │
│  [比利亚雷亚尔]  VS  [塞维利亚]           │
│                                          │
│  ── 盘口选择器 ──                         │
│  ● Moneyline  ○ Spread  ○ Total          │
│                                          │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐    │
│  │ 主胜    │ │ 平局    │ │ 客胜    │    │
│  │ 49¢     │ │ 28¢     │ │ 25¢     │    │
│  │ (49%)   │ │ (28%)   │ │ (25%)   │    │
│  └─────────┘ └─────────┘ └─────────┘    │
│                                          │
│  💰 交易量 $890K  👥 567 持仓             │
└──────────────────────────────────────────┘
```

#### 3.1.2 市场切换交互

每个比赛卡片支持在三种市场类型间切换：
- **Moneyline（默认）**：显示主胜/平局/客胜三个选项
- **Spread**：显示让球选项（如 "主队 -1.5" Yes/No 或 Over/Under 形式）
- **Total**：显示总进球大小球选项（如 "Over 2.5 / Under 2.5"）

#### 3.1.3 比赛列表功能
- **筛选条件**：全部 / 实时 / 即将开始 / 远期
- **联赛筛选**：左侧联赛导航（按大洲分组）
- **排序方式**：热门程度、交易量、最新上线、即将截止
- **搜索**：支持球队名、联赛名搜索

#### 3.1.4 比赛详情页
- 比赛基本信息（时间、场地、联赛）
- 三种市场类型的完整赔率
- 订单簿（Order Book）
- 近期交易记录
- 持仓管理

### 3.2 Props Tab（趣味投注）

#### 3.2.1 页面布局

Props 不按比赛矩阵展示，而是按**市场类型**和**赛事**分组展示：

```
趣味投注页面结构：
  ├── 世界杯专题区（置顶）
  │   ├── 冠军预测卡片
  │   ├── 金靴预测卡片
  │   ├── 小组赛预测卡片
  │   └── 特殊事件卡片
  │
  ├── 五大联赛 Props
  │   ├── 冠军预测
  │   ├── 前四/降级预测
  │   └── 金靴预测
  │
  ├── 杯赛 Props
  │   ├── 欧冠冠军
  │   └── 欧联冠军
  │
  └── 球员 Props
      ├── 球员进球预测
      ├── 球员转会预测
      └── 教练任免预测
```

#### 3.2.2 Props 卡片类型

**类型 A：多项选择卡片（冠军/排名类）**
```
┌──────────────────────────────────┐
│ 🏆 2026世界杯冠军    [热门]     │
│ 💰 交易量 $636M                  │
│                                  │
│  🇪🇸 西班牙    17.2%             │
│  🇫🇷 法国      16.6%             │
│  🏴 英格兰    11.2%             │
│  🇦🇷 阿根廷    8.8%              │
│                                  │
│  查看全部 48 支球队 →            │
└──────────────────────────────────┘
```

**类型 B：Yes/No 卡片（特殊事件类）**
```
┌──────────────────────────────────┐
│ 🤔 内马尔会参加2026世界杯吗？     │
│ 💰 交易量 $817K                  │
│                                  │
│  ✓ Yes  55¢ (55%)               │
│  ✗ No   45¢ (45%)               │
└──────────────────────────────────┘
```

**类型 C：球员表现卡片**
```
┌──────────────────────────────────┐
│ ⚽ 英超金靴得主                   │
│ 💰 交易量 $4M                    │
│                                  │
│  🤖 哈兰德    99%               │
│  ⚽ 伊戈尔·蒂亚戈  1%            │
│                                  │
│  查看全部候选人 →                │
└──────────────────────────────────┘
```

#### 3.2.3 Props 筛选
- **类型筛选**：全部 / 冠军 / 排名 / 金靴 / 球员 / 特殊
- **赛事筛选**：世界杯 / 英超 / 西甲 / 欧冠...

### 3.3 交易功能

#### 3.3.1 Polymarket 交易机制

Polymarket 采用**二进制 Yes/No 合约**机制：
- 每股价格 0¢ - 100¢（$0.00 - $1.00 USDC）
- 价格 = 市场认定的隐含概率
- 正确预测 → 每股结算 $1.00
- 错误预测 → 每股结算 $0.00
- 用户以 USDC 交易

**费用结构**（Polymarket 官方公式）：
- 费率 = `0.0625 × price × (1 - price)`
- 50¢ 合约费率最高 ≈ 1.56%（远低于传统博彩 4-8%）
- 10¢ 或 90¢ 合约费率 ≈ 0.56%

#### 3.3.2 交易面板设计

```
┌────────────────────────────────────┐
│ 交易      比利亚雷亚尔 vs 塞维利亚  │
│                                    │
│  市场: Moneyline                   │
│  ─────────────────────────────     │
│  预测结果: 主胜 (Home Win)         │
│  当前价格: 49¢    隐含概率: 49%    │
│                                    │
│  订单类型: ● 市价单  ○ 限价单      │
│                                    │
│  价格        数量                  │
│  [49¢]  [100]                      │
│  ━━━━━━━━━━━━━━━━━                │
│  -100  -10  +10  +100  +1000       │
│                                    │
│  预计支出: $49.00                  │
│  潜在收益: $51.00 (104%)           │
│  交易费用: ~$0.77 (1.56%)          │
│                                    │
│  [      确认交易      ]            │
└────────────────────────────────────┘
```

#### 3.3.3 订单类型
- **市价单 (Market Order)**：以当前最优价格成交
- **限价单 (Limit Order)**：设定目标价格等待成交

#### 3.3.4 持仓管理
- 当前持仓列表（含盈亏）
- 历史交易记录
- 结算状态追踪（已结算/待结算）

---

## 4. 2026 世界杯专题（核心模块）

### 4.1 世界杯数据规模
- **192+ 个活跃预测市场**
- 冠军市场交易量：**$6.36亿+**（数据来源：polymarket101.com）
- 48 支参赛队伍，12 个小组
- 96 场小组赛 + 淘汰赛

### 4.2 世界杯市场全景

```
2026 FIFA World Cup 预测市场全景（对标 Polymarket）
│
├── 冠军预测（Outright Winner）
│   └── 48 个选项（每支参赛队伍）
│       当前热门: 🇪🇸西班牙 17.2% | 🇫🇷法国 16.6% | 🏴英格兰 11.2%
│
├── 小组赛预测（Group Stage）
│   ├── 12 个小组 × 小组冠军
│   ├── 12 个小组 × 出线资格（前2+最佳第3）
│   └── 96 场小组赛 × 单场市场
│
├── 淘汰赛预测（Knockout Stage）
│   ├── 32 强 → 16 强 → 8 强 → 半决赛 → 决赛
│   └── 每场比赛的 Moneyline / Spread / Total
│
├── 球员 Props
│   ├── Golden Boot（金靴奖）- 32+ 候选人
│   ├── Tournament MVP（赛事最佳球员）
│   └── 单场球员进球/助攻预测
│
└── 特殊 Props
    ├── 哪个大洲夺冠？（欧洲 73% | 南美 21%）
    ├── 是否有 0-0 决赛？
    ├── 首次帽子戏法在哪轮出现？
    ├── 主办国表现（🇺🇸🇨🇦🇲🇽）
    └── 最大比分差
```

### 4.3 世界杯专属页面结构

```
┌────────────────────────────────────────────────────────┐
│  🏆 2026 FIFA World Cup 预测中心                        │
│  💰 总交易量 $960M+ | 📊 192+ 活跃市场 | ⚽ 6.11开赛   │
├────────────────────────────────────────────────────────┤
│                                                        │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐            │
│  │冠军 │ │金靴 │ │小组 │ │淘汰 │ │特殊 │            │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘            │
│                                                        │
│  [冠军预测卡片 - Top 10 球队概率]                        │
│  [小组赛出线 - 6 个热门小组]                             │
│  [即将开赛 - 世界杯热身赛/开幕战]                        │
│  [社区热议 - 评论区聚合]                                 │
│                                                        │
└────────────────────────────────────────────────────────┘
```

### 4.4 世界杯运营节奏

| 阶段 | 时间 | 上线内容 | 运营重点 |
|------|------|---------|---------|
| **预热期** | 5月（现在） | 冠军预测、金靴、小组出线预测 | 早期教育，种子用户培养 |
| **升温期** | 5月底-6月初 | 全部小组赛单场市场 | 大规模推广，世界杯专题页 |
| **爆发期** | 6.11-7.19 | 实时赔率，赛中交易，社区互动 | 交易返佣，预测排行榜 |
| **延续期** | 7月后 | 新赛季联赛市场 | 用户留存，联赛预测 |

---

## 5. 数据与 API 接入

### 5.1 Polymarket API 架构

Polymarket 提供三层 API：

| API 层 | 用途 | 需要认证？ | 基础 URL |
|--------|------|-----------|----------|
| **Gamma API** | 市场元数据、事件列表、价格 | 否（公开） | `gamma-api.polymarket.com` |
| **CLOB API** | 订单簿、下单交易 | 是（钱包） | `clob.polymarket.com` |
| **Data API** | 历史交易数据 | 否 | `data-api.polymarket.com` |

### 5.2 核心接口

#### 获取足球市场列表
```
GET /markets?tag=soccer&closed=false&limit=50
GET /markets?sports_market_types=moneyline,spread,over_under
```

#### 获取比赛事件
```
GET /events?tag=soccer&active=true
GET /events/{event_id}  →  返回该事件下的所有市场
```

#### 获取赔率/订单簿
```
GET /order-book/{clob_token_id}  →  实时买卖盘
```

#### 实时数据 WebSocket
- 体育 WebSocket：实时比分、赔率更新、比赛状态
- 市场 WebSocket：实时成交价、订单簿变化

### 5.3 数据同步策略

| 数据类型 | 同步方式 | 更新频率 |
|---------|---------|---------|
| 市场列表 | REST API 定时拉取 | 每小时 |
| 赔率 | WebSocket 实时推送 | 实时（< 500ms） |
| 交易量 | REST API 定时拉取 + WebSocket 增量 | 每分钟 |
| 比赛状态/比分 | Sports WebSocket | 实时 |
| 结算结果 | UMA Oracle 事件监听 | 事件驱动 |
| 联赛/赛事信息 | Gamma API 定时同步 | 每天 |

---

## 6. 界面设计

### 6.1 导航层级

```
顶部主Tab: 热门 | 体育 | 加密 | 政治 | 电竞
    ↓
体育子Tab: NBA | MLB | 足球 | 网球 | UFC | F1
    ↓
足球子Tab: [ Games（比赛）| Props（趣味投注）] + 世界杯入口
    ↓
左侧联赛筛选: 世界杯 | 英超 | 西甲 | 意甲 | 德甲 | 法甲 | 欧冠 | ...
    ↓
右侧内容区: 比赛卡片 / Props 卡片
```

### 6.2 比赛卡片设计

**Games Tab 卡片包含：**
- 联赛标识 + 比赛时间
- 主队/客队 Logo + 名称
- 盘口类型选择器（Moneyline / Spread / Total）
- 对应赔率按钮（含百分比）
- 交易量 + 持仓人数
- VS 对阵分隔

**Props Tab 卡片包含：**
- Props 标题/问题
- 分类标签（冠军/金靴/球员/特殊）
- 交易量
- 前 3-5 个热门选项的概率排行
- "查看更多"入口

### 6.3 响应式设计
- **桌面端 (≥1280px)**：左侧联赛导航 + 右侧 3 列卡片网格
- **平板端 (768px-1279px)**：顶部联赛导航 + 2 列卡片
- **移动端 (<768px)**：顶部 Tab 导航 + 1 列卡片 + 底部固定交易面板

---

## 7. 技术需求

### 7.1 前端
- React/Vue 框架（与现有平台一致）
- WebSocket 实时数据连接
- 响应式设计（桌面/平板/手机三端适配）
- 性能优化：虚拟列表（市场 > 50 个时）、懒加载

### 7.2 后端
- Polymarket Gamma API 数据聚合层
- WebSocket 实时赔率中继服务
- 订单系统（对接 Polymarket CLOB 或自建交易撮合）
- 结算系统（监听 UMA Oracle 结算事件）
- 数据缓存层（Redis，降低 API 调用频率）

### 7.3 数据处理
- 赔率展示转换（$0.XX → XX¢ → XX%）
- 交易量文本格式化（$XXXK / $X.XM）
- 多语言球队名/联赛名映射
- 时区处理（比赛时间本地化）

---

## 8. 里程碑

| 里程碑 | 时间 | 交付内容 |
|--------|------|---------|
| M1 - 基础搭建 | 5月14日-5月18日 | Polymarket Gamma API 接入，基础数据模型搭建 |
| M2 - Games上线 | 5月19日-5月25日 | Games 页面（比赛列表+详情+Moneyline/Spread/Total交易） |
| M3 - Props+世界杯 | 5月26日-6月3日 | Props 页面（冠军/金靴/小组出线等），世界杯专题页 |
| M4 - 交易闭环 | 6月4日-6月10日 | 完整交易流程（下单/持仓/结算），WebSocket 实时赔率 |
| M5 - 世界杯开赛 | 6月11日 | 世界杯全面上线，运营活动启动 |
| M6 - 持续优化 | 6月-7月 | 赛中功能迭代，新赛季联赛市场接入 |

---

## 9. 成功指标

### 9.1 业务指标
- 世界杯期间 DAU 目标
- 世界杯冠军市场交易量
- 新用户注册数（世界杯引流）
- 用户首笔交易转化率

### 9.2 产品指标
- 页面加载时间 < 2s
- 赔率更新延迟 < 500ms
- 交易成功率 > 99%
- App Store/Google Play 评分 ≥ 4.5

---

## 10. 附录

### 10.1 术语对照表

| 中文 | 英文 | Polymarket 术语 | 说明 |
|------|------|----------------|------|
| 系列 | Series | `series` | 联赛/赛季层级 |
| 事件 | Event | `event` | 单场比赛 |
| 市场 | Market | `market` | 具体预测问题 |
| 独赢 | Moneyline | `moneyline` | 预测胜者 |
| 让分盘 | Spread | `spread` | 让球数预测 |
| 总进球 | Total/Over-Under | `over_under` | 总进球数大小 |
| 趣味投注 | Props | Props 类市场 | 非胜负预测 |
| 隐含概率 | Implied Probability | `price` | 价格即概率 |

### 10.2 调研参考来源
- [Polymarket 官方 Events & Markets 文档](https://docs.polymarket.us/concepts/events-and-markets)
- [Polymarket 体育-足球 Games 页面](https://polymarket.com/zh/sports/soccer/games)
- [Polymarket 体育-足球 Props 页面](https://polymarket.com/zh/sports/soccer/props)
- [Polymarket Gamma API 文档](https://polymarket-data.com/gamma/markets)
- [Polymarket API Guide (pm.wiki)](https://pm.wiki/learn/polymarket-api)
- [Polymarket Sports Markets 2026 Guide (Laika Labs)](https://laikalabs.ai/prediction-markets/polymarket-sports-markets)
- [PolyWorldCup 世界杯追踪](https://www.polyworldcup.info/)
- [Polymarket101 World Cup Trading Guide](https://www.polymarket101.com/en/docs/strategies/world-cup-trading-2026/)
- [machina-sports Polymarket API Sports Codes](https://github.com/machina-sports/sports-skills)

---

**文档结束**