# 奖池瓜分系统 PRD (Product Requirements Document) V1.1

## 1. 概述 (Summary)
本 PRD 旨在定义“奖池瓜分系统”的产品需求。该系统允许平台针对特定玩法（Futures, UpDown, HighLow, Spread, Tap Trading）和交易对设置奖池，通过从用户交易手续费或净利润中抽取一定比例累积奖池金额。用户在活动周期内根据交易表现（如交易量、盈亏额）进行排名，并在周期结束时根据排名按比例瓜分奖池。系统包含业务端（用户展示与交互）和后台管理端（奖池配置与历史查询）。

## 2. 联系人 (Contacts)
*   **产品经理**: [您的名字]
*   **研发负责人**: [待定]
*   **UI/UX 设计**: [待定]
*   **测试负责人**: [待定]

## 3. 背景 (Background)
*   **背景**: 当前平台缺乏一种能够长期激励用户进行高频、大额交易的机制。传统的固定奖励任务对活跃用户的持续刺激不足。
*   **机会**: 引入“奖池”机制，将平台部分收益（手续费/利润）回馈给头部活跃用户，既能制造“大奖”噱头吸引流量，又能通过排行榜激发用户的竞争心理。
*   **目标**: 打造一个自动化的、周期性的、基于排名的激励系统。

## 4. 目标 (Objective)
*   **商业目标**: 提升特定交易对/玩法的交易量，增加用户粘性与活跃时长。
*   **关键结果 (Key Results)**: 相关交易对日均交易量提升 20%；参与用户数达到活跃用户数的 30%。

## 5. 目标用户 (Market Segment)
*   **高频/大额交易用户**: 寻求额外返利与分润。
*   **投机型用户**: 被大奖池吸引，试图冲榜。

## 6. 价值主张 (Value Propositions)
*   **对用户**: 额外收益、荣誉感（排行榜）、规则透明。
*   **对平台**: 收入增长（交易量覆盖成本）、生态闭环。

## 7. 解决方案 (Solution)

### 7.1 业务端 (User Side)

#### 7.1.1 奖池入口 (Entry Point)
*   **位置**: 交易页右侧操作栏顶部 (Top of Right Panel)。
*   **样式**:
    *   **Jackpot Widget**: 黑金风格卡片，展示实时滚动的奖池金额（带光晕动效）。
    *   **关键信息**: 显示“Live”状态、倒计时（如 02d 14h）、用户当前排名及预计分润（如 My Rank: No.12, Est. $350）。
    *   **交互**: 点击 Widget 弹出 **奖池详情模态框 (Modal)**。

#### 7.1.2 奖池详情模态框 (Prize Pool Modal)
*   **头部**:
    *   **奖池总额**: 超大字体展示，带呼吸光效。
    *   **周期信息**: 显示当前期数（如 第 202342 期）及剩余时间。
    *   **规则入口**: 点击弹出规则说明弹窗。
    *   **往期查询**: 下拉菜单选择历史期数，查看往期排名与分配记录。
*   **排行榜 (Leaderboard)**:
    *   **Top 3 领奖台**: 3D 风格展示前三名（头像、昵称、奖金、皇冠图标）。
    *   **列表 (4-100名)**: 紧凑列表展示排名、用户、分润金额。
*   **我的战绩 (My Stats)**:
    *   **位置**: 底部固定区域 (Safe Area)。
    *   **内容**: 我的排名（含升降趋势）、预计分润金额。
    *   **CTA**: “立即交易提升排名”按钮，点击关闭弹窗回到交易页。

### 7.2 后台管理端 (Admin Side)

#### 7.2.1 奖池配置 (Prize Pool Config)
*   **菜单位置**: `玩法管理` -> `奖池配置`。
*   **配置字段**:
    *   **基础信息**:
        *   `奖池名称`: 输入框。
        *   `关联玩法`: [下拉: Futures, UpDown, HighLow, Spread, Tap Trading]。
        *   `关联交易对`: [多选] (如 BTC/USDT, ETH/USDT)。
        *   `状态`: [开启 / 关闭]。
            *   *约束*: 开启状态下，仅允许修改“风控设置”，其他字段锁定不可编辑。
    *   **资金来源**:
        *   `资金来源方式`: [单选: 基础奖池金额 / 抽成配置 / 基础金额+抽成配置]。
        *   `基础奖池金额`: [数字] (若选择了含基础金额的方式则显示)。
        *   `抽成基数`: (若选择了含抽成配置的方式则显示)
            *   若玩法为 Futures/UpDown -> 强制为 **手续费 (Fee)**。
            *   若玩法为 HighLow/Spread/Tap Trading -> 强制为 **净利润 (Profit)**。
            *   *备注说明*: 抽成基数 = 平台收益 - 代理返佣。
        *   `抽成比例`: [数字 %] (如 5%)。
    *   **周期设置**:
        *   `周期类型`: [下拉: 每小时, 每日, 每周, 每月, 自定义]。
        *   `自定义时长`: 若选自定义，输入小时数。
        *   `开始时间`: 日期时间选择器。
        *   `结束时间`: 根据周期类型和开始时间自动计算（自然天、自然周、自然月）。
            *   自然天：当天结束（23:59:59）
            *   自然周：本周日结束（23:59:59）
            *   自然月：本月最后一天结束（23:59:59）
            *   自定义：开始时间加上自定义时长
    *   **分配规则 (Distribution)**:
        *   `排名依据`: [单选: 交易量 / 盈亏额 / 利润绝对值]。
        *   **自定义区间**: 支持添加多行规则 (Start Rank - End Rank : Percentage)。
        *   *示例*: 第 1-1 名 20%；第 2-5 名 30% (均分)；第 6-100 名 50% (均分)。
        *   **校验**: 总比例不能超过 100%（剩余滚入下期）。
    *   **风控设置**:
        *   `开奖门槛`: 奖池金额 < X 时不需开奖。

#### 7.2.2 列表与操作
*   **列表展示**: 包含关联活动列（Popover 展示关联活动名称）。
*   **状态列**: 支持直接快捷开启和关闭奖池状态。
*   **历史记录**: 弹窗展示历史期数、关联活动名称、大奖得主。
*   **名单配置**: 独立的黑名单配置功能，用于配置奖池的黑名单用户（配置后，这些黑名单用户就不能参与到奖池瓜分活动）。
*   *(注：暂时移除删除功能)*

## 8. 详细设计方案 (Detailed Design)

### 8.1 数据库设计 (Database Schema)

#### 8.1.1 奖池配置表 (`prize_pool_config`)
| 字段名 | 类型 | 说明 |
| :--- | :--- | :--- |
| `id` | BIGINT | 主键 |
| `name` | VARCHAR | 奖池名称 |
| `game_type` | VARCHAR | 关联玩法 (futures/updown/highlow...) |
| `pairs` | JSON | 关联交易对 |
| `status` | INT | 状态 (1:active, 0:inactive) |
| `fund_source_type` | VARCHAR | 资金来源方式 (base/ratio/both) |
| `base_amount` | DECIMAL | 基础奖池金额 |
| `fee_source` | VARCHAR | 抽成基数 (fee/profit) |
| `fee_ratio` | DECIMAL | 抽成比例 |
| `cycle_type` | VARCHAR | 周期类型 (hourly/daily/weekly/custom) |
| `custom_hours` | INT | 自定义周期小时数 |
| `start_time` | DATETIME | 开始时间 |
| `rank_by` | VARCHAR | 排名依据 (volume/pnl/profit_abs) |
| `distribution_rules` | JSON | 分配规则 `[{start:1, end:1, percent:20}, ...]` |
| `min_pool_amount` | DECIMAL | 开奖门槛 |

#### 8.1.2 奖池周期实例表 (`prize_pool_round`)
| 字段名 | 类型 | 说明 |
| :--- | :--- | :--- |
| `id` | BIGINT | 主键 |
| `pool_config_id` | BIGINT | 关联配置ID |
| `round_number` | VARCHAR | 期数 (20231001) |
| `start_time` | DATETIME | 开始时间 |
| `end_time` | DATETIME | 结束时间 |
| `current_amount` | DECIMAL | 当前累积金额 |
| `status` | INT | 状态 (0:running, 1:settling, 2:completed) |

#### 8.1.3 用户奖池排名表 (`prize_pool_rank`)
| 字段名 | 类型 | 说明 |
| :--- | :--- | :--- |
| `id` | BIGINT | 主键 |
| `round_id` | BIGINT | 关联周期实例ID |
| `user_id` | BIGINT | 用户ID |
| `score` | DECIMAL | 排名依据数值 |
| `rank` | INT | 排名 |
| `est_reward` | DECIMAL | 预计奖励 |

#### 8.1.4 奖池黑名单表 (`prize_pool_blacklist`)
| 字段名 | 类型 | 说明 |
| :--- | :--- | :--- |
| `id` | BIGINT | 主键 |
| `pool_config_id` | BIGINT | 关联奖池配置ID |
| `user_id` | BIGINT | 用户ID |
| `created_at` | DATETIME | 创建时间 |

### 8.2 核心逻辑 (Core Logic)
1.  **资金累积**:
    *   监听 `TradeSettledEvent`。
    *   根据 `game_type` 和 `pairs` 匹配奖池配置。
    *   若 `fee_source=fee`，取交易手续费 * ratio；若 `fee_source=profit`，取平台净利（用户亏损 - 用户盈利） * ratio（若为负则不累积）。
    *   更新 Redis 缓存及数据库。
2.  **结算逻辑**:
    *   定时任务触发周期结束。
    *   检查 `current_amount >= min_pool_amount`。
    *   若满足，按 `distribution_rules` 计算 Top N 用户奖励，生成发放流水。
    *   若不满足，金额滚入下一期。

## 9. 发布计划 (Release Plan)
*   **V1.0**: 完成上述核心功能（Futures/UpDown 手续费抽成，自定义分配规则，前端 Widget 及详情页）。
*   **V1.1**: 支持“平台净利润”抽成逻辑（需对接风控/清算系统）。