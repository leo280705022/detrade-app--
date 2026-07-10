# 后管-内部账本与链上 USDC 对账系统设计方案

## 文档信息

- **版本**: v1.0
- **日期**: 2026-05-19
- **状态**: 初稿完成
- **适用范围**: 后管系统 · 资金对账模块
- **关联系统**: 内部账本系统 / Polymarket CLOB / Polygon 链上数据 / 冷钱包系统

---

## 1. 业务背景与核心逻辑

### 1.1 资金流转全景

```
用户 A (USDT)  ──┐
用户 B (ETH)   ──┤      ┌──────────────┐
用户 C (法币)  ──┼──►   │  平台内部账本  │──► USDC.e 余额记录
                  │      └──────┬───────┘
                  │             │
                  │             ▼
                  │      ┌──────────────┐      ┌──────────────┐
                  │      │  流动性池 /   │      │  冷钱包系统   │
                  └──────┤  资产管理    │─────►│ (Polygon)    │
                         │              │      │  USDC.e 余额 │
                         └──────────────┘      └──────┬───────┘
                                                      │
                                                      ▼
                                               ┌──────────────┐
                                               │ Polymarket   │
                                               │ CLOB 交易    │
                                               │ 下单/吃单     │
                                               └──────┬───────┘
                                                      │
                                                      ▼
                                               ┌──────────────┐
                                               │ 市场结算     │
                                               │ UMA Oracle  │
                                               │ USDC.e 回流  │
                                               └──────────────┘
```

### 1.2 完整资金链路

| 步骤 | 描述 | 资产形态 | 影响系统 |
|------|------|---------|---------|
| **① 用户入金** | 用户充值任意资产（USDT/ETH/BTC/法币） | 原生资产 → USDC.e | 内部账本 + 冷钱包 |
| **② 用户下单** | 用户在平台发起预测交易 | USDC.e 冻结 | 内部账本（order locked） |
| **③ 冷钱包执行** | 冷钱包调用 Polymarket CLOB 下单 | USDC.e → 成交 → Outcome Tokens | 冷钱包 + Polymarket |
| **④ 市场结算** | Polymarket 市场到期，Oracle 裁决 | Outcome Tokens → USDC.e 退款 | Polymarket → 冷钱包 |
| **⑤ 用户结算** | 平台根据结果为用户结算盈亏 | USDC.e 解冻 + 分配 | 内部账本 |
| **⑥ 用户出金** | 用户提现 | USDC.e → 目标资产 | 内部账本 + 冷钱包 |

### 1.3 核心资产模型

```
平台总资产 = Σ (所有用户内部账本余额) + 平台收入 + 待处理流动性
                  │
                  ▼
         必须等于 (可能存在 gas 差额):
         = 冷钱包 USDC.e 余额 + Polymarket Open Positions 价值
```

### 1.4 对账复杂度来源

| 来源 | 原因 | 影响 |
|------|------|------|
| **多资产入金** | 用户充 USDT/ETH/法币，平台汇兑为 USDC.e | 需追踪汇率和汇兑损耗 |
| **冷钱包操作** | 集中式冷钱包操作，存在 gas 费用、时间窗口差 | 链上确认与内部记账有时间差 |
| **Polymarket CLOB** | 订单撮合是链下 + 链上分步完成 | 订单状态存在 intermediate 状态 |
| **Outcome Tokens** | 成交后持有的是 Outcome Token 而非 USDC.e | 结算前 USDC.e 余额会减少 |
| **市场结算** | 多市场并行结算，Oracle 裁决有时间窗口 | 结算事件需逐一匹配 |
| **Gas 费用** | Polygon 链上操作需要 MATIC 作为 gas | 平台需额外管理 MATIC 余额 |
| **USDC.e 精度** | Polygon 上 USDC.e 含 6 位小数 | 精度处理需统一 |

---

## 2. 对账模型设计

### 2.1 核心对账等式

```
┌─────────────────────────────────────────────────────────────────┐
│                  对账核心恒等式                                     │
│                                                                   │
│  冷钱包 USDC.e 余额 + Polymarket 持仓价值 = 内部账本总负债           │
│                                                                   │
│  其中:                                                                │
│  - 冷钱包 USDC.e 余额 = 链上实时查询                                   │
│  - Polymarket 持仓价值 = 所有未结算 Outcome Token 的账面价值            │
│  - 内部账本总负债 = Σ(所有用户可用余额) + Σ(冻结中订单金额) + 平台收入余额 │
│                                                                   │
│  允许误差范围: ≤ 5 USDC.e (覆盖 gas 小额偏差)                         │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 三级对账体系

```
对账系统
│
├── 第一级: 余额级对账 (每日/W6)
│   │   冷钱包余额 vs 内部账本总负债
│   │   粗粒度，快速定位整体偏差
│
├── 第二级: 交易级对账 (每6小时/W12)
│   │   逐笔对比 冷钱包链上交易 ↔ 内部账本流水 ↔ Polymarket 成交
│   │   细粒度，定位具体偏差来源
│
└── 第三级: 持仓级对账 (实时/W24)
        Polymarket Outcome Token 持仓 ↔ 内部用户持仓快照
        校验每个市场的多头/空头是否匹配
```

### 2.3 数据来源矩阵

| 数据源 | 类型 | 获取方式 | 更新频率 | 核心字段 |
|--------|------|---------|---------|---------|
| **内部账本** | DB 流水 | 平台数据库 | 实时 | 用户ID、变动金额、时间戳、类型、关联OrderID |
| **冷钱包链上余额** | On-chain | Polygon RPC | 每5分钟 | 地址、USDC.e 余额、MATIC 余额 |
| **冷钱包链上交易** | On-chain | PolygonScan API / RPC | 每1分钟 | TxHash、From/To、金额、区块时间、状态 |
| **Polymarket 订单状态** | API | CLOB REST API | 实时 | OrderID、市场、方向、价格、数量、状态 |
| **Polymarket 成交记录** | API | CLOB REST API / WS | 实时 | TradeID、OrderID、填充量、价格、时间 |
| **Polymarket 持仓** | API | CLOB REST API | 每5分钟 | TokenID、持有数量、账面价值 |
| **Polymarket 市场结算** | On-chain | Polygon 事件监听 | 事件驱动 | 市场ID、结果、结算金额、结算时间 |

---

## 3. 数据库表设计

### 3.1 核心对账表

#### 对账批次表 `reconciliation_batches`

```sql
CREATE TABLE reconciliation_batches (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    batch_no            VARCHAR(32) NOT NULL UNIQUE COMMENT '对账批次号，格式: REC-YYYYMMDD-HHMMSS',
    recon_type          ENUM('balance', 'transaction', 'position') NOT NULL COMMENT '对账类型',
    status              ENUM('pending', 'running', 'completed', 'failed', 'discrepancy_found') NOT NULL,
    start_time          DATETIME NOT NULL COMMENT '对账开始时间',
    end_time            DATETIME COMMENT '对账结束时间',
    total_ledger_balance DECIMAL(18,6) COMMENT '内部账本总负债',
    total_onchain_balance DECIMAL(18,6) COMMENT '链上 USDC.e 余额',
    total_position_value DECIMAL(18,6) COMMENT 'Polymarket 持仓估值',
    discrepancy_amount  DECIMAL(18,6) COMMENT '差额金额',
    is_passed           TINYINT(1) COMMENT '是否通过 1=通过 0=不通过',
    operator_id         BIGINT COMMENT '操作人ID（手动触发时）',
    remark              VARCHAR(500),
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_batch_no (batch_no),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);
```

#### 对账明细表 `reconciliation_details`

```sql
CREATE TABLE reconciliation_details (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    batch_id            BIGINT NOT NULL COMMENT '关联批次ID',
    recon_dimension     ENUM('balance', 'order', 'position', 'settlement') NOT NULL,
    ledger_item_id      VARCHAR(64) COMMENT '内部账本记录ID',
    onchain_tx_hash     VARCHAR(128) COMMENT '链上交易Hash',
    polymarket_order_id VARCHAR(128) COMMENT 'Polymarket 订单ID',
    market_id           VARCHAR(128) COMMENT '预测市场ID',
    asset               VARCHAR(20) NOT NULL DEFAULT 'USDC.e',
    ledger_amount       DECIMAL(18,6) COMMENT '内部账本金额',
    onchain_amount      DECIMAL(18,6) COMMENT '链上金额',
    discrepancy         DECIMAL(18,6) COMMENT '差额',
    status              ENUM('matched', 'ledger_only', 'onchain_only', 'amount_mismatch') NOT NULL,
    resolution_status   ENUM('unresolved', 'auto_resolved', 'manual_resolved') DEFAULT 'unresolved',
    resolved_by         BIGINT COMMENT '解决人ID',
    resolved_at         DATETIME COMMENT '解决时间',
    remark              TEXT,
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_batch_id (batch_id),
    INDEX idx_status (status),
    INDEX idx_resolution (resolution_status)
);
```

#### 内部账本流水表 `ledger_transactions`

```sql
CREATE TABLE ledger_transactions (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    tx_no               VARCHAR(32) NOT NULL UNIQUE COMMENT '流水号',
    user_id             BIGINT NOT NULL COMMENT '用户ID',
    type                ENUM(
                            'deposit',          -- 充值入金
                            'withdrawal',       -- 提现出金
                            'order_place',      -- 下单冻结
                            'order_cancel',     -- 取消解冻
                            'order_settle_win', -- 结算盈利
                            'order_settle_lose',-- 结算亏损
                            'exchange_in',      -- 汇兑入
                            'exchange_out',     -- 汇兑出
                            'platform_fee',     -- 平台费用
                            'admin_adjust',     -- 管理员调整
                            'gas_fee'           -- Gas 费用扣减
                        ) NOT NULL,
    asset               VARCHAR(20) NOT NULL DEFAULT 'USDC.e',
    amount              DECIMAL(18,6) NOT NULL COMMENT '变动金额（正=入，负=出）',
    balance_before      DECIMAL(18,6) NOT NULL,
    balance_after       DECIMAL(18,6) NOT NULL,
    order_id            VARCHAR(64) COMMENT '关联订单ID',
    market_id           VARCHAR(128) COMMENT '关联市场ID',
    onchain_tx_hash     VARCHAR(128) COMMENT '关联链上交易Hash',
    status              ENUM('pending', 'confirmed', 'failed') NOT NULL DEFAULT 'confirmed',
    remark              VARCHAR(200),
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    confirmed_at        DATETIME,
    INDEX idx_user_id (user_id),
    INDEX idx_type (type),
    INDEX idx_order_id (order_id),
    INDEX idx_onchain_tx (onchain_tx_hash),
    INDEX idx_created_at (created_at)
);
```

#### Polymarket 持仓快照表 `polymarket_positions`

```sql
CREATE TABLE polymarket_positions (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    market_id           VARCHAR(128) NOT NULL COMMENT 'Polymarket 市场ID',
    token_id            VARCHAR(128) NOT NULL COMMENT 'Outcome Token ID (positionId)',
    outcome             ENUM('YES', 'NO') NOT NULL,
    quantity            DECIMAL(30,0) NOT NULL COMMENT '持有数量（原子单位）',
    avg_entry_price     DECIMAL(18,6) COMMENT '平均成交价格',
    current_market_price DECIMAL(18,6) COMMENT '当前市场价格',
    market_value        DECIMAL(18,6) COMMENT '当前估值 (quantity * price / 1e6)',
    settlement_value    DECIMAL(18,6) COMMENT '结算价值',
    market_status       ENUM('active', 'resolved', 'expired') NOT NULL DEFAULT 'active',
    settlement_outcome  VARCHAR(20) COMMENT '结算结果 YES/NO',
    settled             TINYINT(1) DEFAULT 0,
    settled_at          DATETIME,
    last_sync_at        DATETIME NOT NULL COMMENT '最后同步时间',
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_token (token_id),
    INDEX idx_market_id (market_id),
    INDEX idx_settled (settled),
    INDEX idx_last_sync (last_sync_at)
);
```

#### 冷钱包交易记录表 `cold_wallet_transactions`

```sql
CREATE TABLE cold_wallet_transactions (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    tx_hash             VARCHAR(128) NOT NULL COMMENT '链上交易Hash',
    from_address        VARCHAR(64) NOT NULL,
    to_address          VARCHAR(64) NOT NULL,
    asset               VARCHAR(20) NOT NULL DEFAULT 'USDC.e',
    amount              DECIMAL(30,6) NOT NULL COMMENT '交易金额',
    gas_used            DECIMAL(30,6) COMMENT 'Gas 消耗 (MATIC)',
    gas_price_gwei      DECIMAL(30,2) COMMENT 'Gas 价格 (Gwei)',
    tx_type             ENUM(
                            'deposit_collect',    -- 归集用户充值
                            'polymarket_trade',   -- Polymarket 下单
                            'polymarket_settle',  -- 市场结算收款
                            'withdrawal_send',    -- 用户提现发送
                            'wallet_transfer',    -- 钱包间转账
                            'approve',            -- Token 授权
                            'other'
                        ) NOT NULL COMMENT '交易类型',
    block_number        BIGINT NOT NULL,
    block_timestamp     DATETIME NOT NULL,
    status              ENUM('pending', 'success', 'failed') NOT NULL,
    internal_order_id   VARCHAR(64) COMMENT '关联内部订单ID',
    remark              VARCHAR(200),
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_tx_hash (tx_hash),
    INDEX idx_tx_type (tx_type),
    INDEX idx_block_number (block_number),
    INDEX idx_internal_order (internal_order_id),
    INDEX idx_status (status)
);
```

#### 对账异常记录表 `reconciliation_anomalies`

```sql
CREATE TABLE reconciliation_anomalies (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    batch_id            BIGINT NOT NULL,
    anomaly_type        ENUM(
                            'balance_mismatch',       -- 余额不匹配
                            'missing_onchain_tx',     -- 缺少链上交易
                            'missing_ledger_tx',      -- 缺少账本记录
                            'amount_mismatch',        -- 金额不符
                            'order_status_mismatch',  -- 订单状态不一致
                            'position_mismatch',      -- 持仓不匹配
                            'settlement_mismatch',    -- 结算异常
                            'duplicate_tx',           -- 重复交易
                            'unresolved_order',       -- 未解决订单
                            'stale_position'          -- 过期未结持仓
                        ) NOT NULL,
    severity            ENUM('low', 'medium', 'high', 'critical') NOT NULL DEFAULT 'medium',
    detail              TEXT NOT NULL COMMENT '异常详情（含JSON数据）',
    suggested_action    VARCHAR(500) COMMENT '建议处理方式',
    status              ENUM('open', 'investigating', 'resolved', 'ignored') NOT NULL DEFAULT 'open',
    assigned_to         BIGINT COMMENT '处理人',
    resolved_at         DATETIME COMMENT '解决时间',
    resolution_note     TEXT COMMENT '解决说明',
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    INDEX idx_batch_id (batch_id),
    INDEX idx_type (anomaly_type),
    INDEX idx_severity (severity),
    INDEX idx_status (status)
);
```

---

## 4. 对账算法与流程

### 4.1 第一级：余额级对账（粗粒度，每日 W6/W00）

```
┌──────────────────────────────────────────────────────────────────┐
│  余额级对账流程                                                     │
│                                                                   │
│  Step 1: 计算内部账本总负债                                          │
│    total_user_balance = Σ(所有用户 USDC.e 可用余额)                   │
│    total_frozen = Σ(所有用户冻结中的订单金额)                          │
│    platform_income = Σ(平台手续费累计)                               │
│    → ledger_total = total_user_balance + total_frozen + platform_income │
│                                                                   │
│  Step 2: 查询冷钱包 USDC.e 链上余额                                   │
│    onchain_balance = USDC_e_contract.balanceOf(cold_wallet_address) │
│                                                                   │
│  Step 3: 查询 Polymarket 持仓估值                                      │
│    position_value = Σ(每张 Outcome Token 数量 × 当前市场价格)          │
│    (YES Token 按当前卖一价评估，NO Token 按 (1 - 当前买一价) 评估)    │
│                                                                   │
│  Step 4: 计算差额                                                     │
│    total_assets = onchain_balance + position_value                  │
│    discrepancy = total_assets - ledger_total                        │
│                                                                   │
│  Step 5: 判定                                                       │
│    if |discrepancy| ≤ 5 USDC.e: ✅ 通过                              │
│    else: 创建 anomaly → 触发第二级对账                                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 第二级：交易级对账（细粒度，逐个交易匹配）

#### 4.2.1 冷钱包链上交易 ↔ 内部账本流水对账

```
冷钱包交易记录                             内部账本流水
┌─────────────┐                        ┌──────────────┐
│ TX_HASH_001  │──┐                  ┌──│ TX_NO_001    │
│ 存入 5000    │  │                  │  │ 存入 5000    │
│ USDC.e       │  │  按 tx_hash /    │  │ USDC.e       │
├─────────────┤  ├── 金额 匹配 ────┤  ├──────────────┤
│ TX_HASH_002  │  │                  │  │ TX_NO_002    │
│ 转出 1000    │  │                  │  │ 转出 1000    │
│ (下单)       │──┘                  └──│ (下单冻结)    │
└─────────────┘                        └──────────────┘

匹配规则:
  ┌─────────────────────────────────────────────────────────┐
  │ 匹配维度          │ 冷钱包字段       │ 内部账本字段       │
  ├─────────────────────────────────────────────────────────┤
  │ Hash 直接匹配     │ tx_hash         │ onchain_tx_hash  │
  │ 金额 + 时间窗口    │ amount + time   │ amount + time    │
  │ 内部订单号         │ internal_order_id│ order_id         │
  └─────────────────────────────────────────────────────────┘
```

#### 4.2.2 Polymarket API 订单 ↔ 内部订单对账

```
Polymarket CLOB API                       平台内部订单
┌──────────────────┐                     ┌──────────────────┐
│ OrderID: PM-001   │──┐               ┌──│ OrderID: ORD-001 │
│ Market: 0xabc     │  │               │  │ Market: 世界杯  │
│ Side: BUY YES    │  │  按 OrderID   │  │ Amount: 1000    │
│ Size: 1000 x 49¢ │  ├── 匹配 ──────┤  │ Price: 49¢      │
│ Status: FILLED   │  │               │  │ Status: filled  │
├──────────────────┤  │               │  ├──────────────────┤
│ OrderID: PM-002   │  │               │  │ OrderID: ORD-002 │
│ Status: OPEN     │──┘               └──│ Status: pending  │
│ (未成交)          │                     │ (等待成交)       │
└──────────────────┘                     └──────────────────┘

匹配规则:
  1. 按 OrderID 直接关联
  2. 按 (market_id + side + price + amount) 模糊匹配
  3. 订单状态对账:
     - FILLED = matched (已成交) ✓
     - OPEN = pending (未成交)  需要关注超时
     - CANCELED = canceled (已取消) 需要核对取消原因
```

### 4.3 第三级：持仓级对账（实时/W24）

```
┌──────────────────────────────────────────────────────────────────┐
│  持仓级对账流程                                                     │
│                                                                   │
│  Step 1: 从 Polymarket CLOB API 获取所有未结算持仓                     │
│    GET /positions → [{token_id, size, market_id, side, ...}]       │
│                                                                   │
│  Step 2: 从内部系统聚合用户持仓                                         │
│    SELECT market_id, outcome, SUM(quantity) FROM user_positions    │
│    WHERE settled = 0 GROUP BY market_id, outcome                   │
│                                                                   │
│  Step 3: 逐市场匹配                                                   │
│    for each market:                                                 │
│      polymarket_yes = polymarket_positions[market].yes_size         │
│      polymarket_no = polymarket_positions[market].no_size           │
│      internal_yes = internal_positions[market].total_yes            │
│      internal_no = internal_positions[market].total_no              │
│                                                                   │
│      yes_diff = polymarket_yes - internal_yes                      │
│      no_diff = polymarket_no - internal_no                         │
│                                                                   │
│      if yes_diff ≠ 0 or no_diff ≠ 0: → 创建 anomaly               │
│                                                                   │
│  Step 4: 市场结算状态追踪                                              │
│    监听 Polymarket Settlement 事件                                   │
│    结算后:                                                          │
│      1. Outcome Token 销毁，USDC.e 回冷钱包                             │
│      2. 内部账本根据结果为用户结算                                       │
│      3. 对账: polymarket 结算金额 vs 内部账本结算总额                    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 4.4 特殊场景处理

#### 场景1：纵向滑点/部分成交

```
问题: 用户下单 1000 张，冷钱包吃单只成交 600 张

对账处理:
  1. 内部账本: 冻结 1000 张 USDC.e
  2. Polymarket 成交: 600 张
  3. 对账发现: order_status_mismatch
  4. 系统自动: 解冻未成交的 400 张，更新订单状态为 partially_filled
```

#### 场景2：市场结算延迟

```
问题: Polymarket Oracle 结算后，链上 TX 尚未执行，但内部已结算

对账处理:
  1. 内部已结算用户
  2. 冷钱包 USDC.e 尚未收到
  3. 对账发现: balance_mismatch（临时性）
  4. 自动标记为 pending_settlement，下一轮对账自动修正
```

#### 场景3：Gas 费用消耗

```
问题: 冷钱包多次链上操作消耗 MATIC，且 USDC.e 提现偶尔有 gas

对账处理:
  1. 建立 gas 费用跟踪子账户
  2. 对账时排除 gas 费用差额
  3. 定期补充冷钱包 MATIC（从平台收入中划拨）
```

#### 场景4：多资产入金汇兑

```
问题: 用户充 USDT，平台兑换为 USDC.e 时存在汇率差

对账处理:
  1. 记录汇兑流水（exchange_in / exchange_out）
  2. 汇兑差额记入平台收入/支出
  3. 对账时校验: 入金资产价值 vs USDC.e 入账金额一致性
```

---

## 5. 后管对账系统界面设计

### 5.1 对账总览 Dashboard

```
┌──────────────────────────────────────────────────────────────────┐
│  💰 资金对账中心                              [手动触发对账] [自动对账]│
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  上次对账: 2026-05-19 06:00:00  ✅ 通过                           │
│  下次对账: 2026-05-19 12:00:00                                    │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐            │
│  │ 内部账本总负债  │  │ 冷钱包余额    │  │ 持仓估值      │            │
│  │ 12,847,362.50 │  │ 8,234,567.80 │  │ 4,612,789.45 │            │
│  │    USDC.e     │  │    USDC.e    │  │    USDC.e    │            │
│  └──────────────┘  └──────────────┘  └──────────────┘            │
│                                                                   │
│  总资产: 12,847,357.25 USDC.e   |   差额: 5.25 USDC.e ✅           │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  最近对账趋势 (7天)                                        │  │
│  │  ┌────────────────────────────────────────────────┐         │  │
│  │  │  ████████████████████████████████████████░░░░  │         │  │
│  │  │  余额差额保持在 ±5 USDC.e 以内                    │         │  │
│  │  └────────────────────────────────────────────────┘         │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  待处理异常: 3  (高: 0 | 中: 1 | 低: 2)                           │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 对账批次列表

```
┌──────────────────────────────────────────────────────────────────┐
│  📋 对账批次记录                                [筛选] [导出]     │
├──────────────────────────────────────────────────────────────────┤
│  批次号              │ 类型  │ 时间            │ 状态   │ 差额   │
│────────────────────────────────────────────────────────────────┤
│  REC-20260519-0600  │ 余额   │ 05-19 06:00:00 │ ✅ 通过 │ +5.25 │
│  REC-20260519-0000  │ 余额   │ 05-19 00:00:00 │ ✅ 通过 │ -2.10 │
│  REC-20260518-1800  │ 交易   │ 05-18 18:00:00 │ ✅ 通过 │ 0.00  │
│  REC-20260518-1200  │ 交易   │ 05-18 12:00:00 │ ⚠️ 异常 │ -34.50│ ← ▲
│  REC-20260518-0600  │ 余额   │ 05-18 06:00:00 │ ✅ 通过 │ +1.80 │
│  REC-20260518-0500  │ 持仓   │ 05-18 05:00:00 │ ❌ 失败 │ ----  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.3 异常详情页

```
┌──────────────────────────────────────────────────────────────────┐
│  ⚠️ 对账异常详情 | 批次: REC-20260518-1200   [标记处理] [忽略]    │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─── 异常 1: 交易级不匹配 ──────────────────────────────────┐    │
│  │  类型:    amount_mismatch    严重程度: 🔴 高              │    │
│  │  发现:    2026-05-18 12:05:23                              │    │
│  │                                                           │    │
│  │  内部账本:  TX_NO-20260518-0032  -1,000.00 USDC.e (下单冻结)│    │
│  │  链上交易:  0xabc...def          -965.50 USDC.e            │    │
│  │  差额:     -34.50 USDC.e                                   │    │
│  │                                                           │    │
│  │  🧐 分析: 冷钱包下单 965.50 USDC.e 但内部冻结了 1000        │    │
│  │  💡 建议: 解冻差额 34.50 USDC.e 回用户账户                  │    │
│  │                                                           │    │
│  │  [一键修复] [手动处理] [跳过]                                │    │
│  └───────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─── 异常 2: 订单状态不匹配 ────────────────────────────────┐    │
│  │  类型:    order_status_mismatch    严重程度: 🟡 中        │    │
│  │  发现:    2026-05-18 12:05:23                              │    │
│  │                                                           │    │
│  │  内部订单: ORD-20260518-0089  状态: filled                 │    │
│  │  PM 订单:  PM-20260518-0045   状态: open (部分成交 300/1000) │    │
│  │                                                           │    │
│  │  💡 建议: 更新内部订单状态为 partially_filled (300/1000)    │    │
│  │           解冻未成交部分 700×0.49=343 USDC.e                │    │
│  │                                                           │    │
│  │  [一键修复] [手动处理] [跳过]                                │    │
│  └───────────────────────────────────────────────────────────┘    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 5.4 交易明细对账视图

```
┌──────────────────────────────────────────────────────────────────┐
│  🔍 交易明细对账                    [批量匹配] [导出CSV] [刷新]   │
├──────────────────────────────────────────────────────────────────┤
│  内部账本流水                          │ 链上/PM 交易             │ 匹配│
│────────────────────────────────────────────────────────────────┤
│ TX_NO-001 │ 充值 +5000   ✅ confirmed   │ 0xabc... │ 存入 +5000  │ ✅  │
│ TX_NO-002 │ 下单冻结 -1000  ✅ confirmed│ PM-001   │ 成交 1000   │ ✅  │
│ TX_NO-003 │ 下单冻结 -500   ✅ pending  │ ----     │ ----        │ ⏳  │
│           │                              │ PM-002   │ 成交 300   │ ⚠️  │
│ TX_NO-004 │ 结算盈利 +1500 ✅ confirmed  │ 0xdef... │ 存入 +1500  │ ✅  │
│           │ ----                         │ 0x123... │ 转入 +2000  │ ❓  │
└──────────────────────────────────────────────────────────────────┘
  ▸ 筛选: [全部] [已匹配] [仅内部] [仅链上] [金额不符]
  ▸ 标注: ✅=已匹配 ⏳=待确认 ⚠️=数量不符 ❓=未匹配
```

### 5.5 资产流动 Sankey 图

```
┌──────────────────────────────────────────────────────────────────┐
│  🧭 资金流动圖 (Sankey)                           [7天] [30天]   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  用户入金 ──────────────────► 内部账本 ──► 冷钱包 ──► Polymarket   │
│  USDT: 32.5M                │   总流入     │ 下单      │ 市场A     │
│  ETH: 8.2M                  │   48.2M     │ 22.5M    │ 12.3M    │
│  法币: 7.5M                 │             │          │ 市场B     │
│       │                     │   总流出     │ 结算      │ 8.9M     │
│       │                     │   45.8M     │ 20.1M    │ 市场C     │
│       ▼                     │             │          │ 1.3M     │
│  汇兑损耗: 0.15%            │   净头寸     │ 提现      │          │
│                             │   +2.4M     │ 3.2M     │          │
│                             │             │          │          │
└──────────────────────────────────────────────────────────────────┘
```

---

## 6. 定时调度与告警

### 6.1 对账调度方案

| 对账级别 | 执行频率 | 触发方式 | 耗时预估 | SLA |
|---------|---------|---------|---------|-----|
| **余额级** | 每日 W6, W00 | 定时任务 | < 30秒 | 次日报出 |
| **交易级** | 每6小时 (W00/W6/W12/W18) | 定时任务 | < 5分钟 | 次日报出 |
| **持仓级** | 每30分钟 | 定时任务 | < 2分钟 | 实时监控 |
| **实时异常** | 事件驱动 | 链上事件监听 + API WebSocket | < 10秒 | 实时告警 |

### 6.2 告警规则

| 规则 | 触发条件 | 通知方式 | 响应SLA |
|------|---------|---------|---------|
| **余额偏差** | 第一级差额 > 10 USDC.e | 钉钉/企微/Slack + 邮件 | 1小时内 |
| **交易不匹配** | 第二级发现 mismatch > 50 USDC.e | 邮件 + 后管通知 | 4小时内 |
| **持仓偏差** | 第三级发现 持仓数量不一致 | 邮件 | 当天 |
| **连续异常** | 同一类型异常连续 3 次出现 | 电话（值班运维） | 1小时内 |
| **结算异常** | 市场已结算但持仓未处理 > 24h | 邮件 + 后管通知 | 当天 |
| **冷钱包余额不足** | USDC.e 余额 < 10000 | 邮件 | 1小时内 |

### 6.3 自动修复策略

| 异常类型 | 自动修复策略 | 安全范围 |
|---------|------------|---------|
| **部分成交差额** | 自动解冻未成交部分并更新订单状态 | ≤ 1000 USDC.e |
| **小额余额偏差** | 自动创建调整流水（记入平台损益） | ≤ 5 USDC.e |
| **订单状态不一致** | 以 Polymarket API 状态为准更新内部状态 | 全部 |
| **T+1 延迟结算** | 暂时忽略，标记为 pending_settlement，下轮再校验 | 全部 |
| **Gas 费用偏移** | 免除对账，列入 Gas 费用子账户 | ≤ 10 MATIC |

---

### 6.4 异常与业务端订单处理策略

对账异常发现后，**不能一刀切地取消或报错**。需要根据异常的类型和严重程度，选择不同的处理路径。以下是对账异常与业务端订单的完整处理策略

#### 6.4.1 异常四象限分类（按影响范围）

```
                                   异常严重程度
                        轻（≤ 5 USDC.e）        重（> 5 USDC.e）
       ┌────────────────────────────────────────────────────
     能  │   ✅ 象限Ⅰ: 自动修复               🔴 象限Ⅱ: 暂停 + 人工
     自  │   部分成交、滑点偏差、Gas偏移      资金金额不符、缺少链上记录
     动  │   不通知用户，系统自动消化        冻结相关账户，风控审核
     修  │
     复  │
  是  ├────────────────────────────────────────────────────
     否  │   ⏳ 象限Ⅲ: 等待重试               ❌ 象限Ⅳ: 取消退款
        │   链上 pending、API 超时           超时未成交、市场取消
        │   临时状态，设置超时阈值           确认不可执行后取消
        │
```

#### 6.4.2 四类异常处理细则

##### ✅ 象限Ⅰ：可自动修复的异常 → 不报错，不取消

| 异常场景 | 根因 | 处理方式 | 用户感知 |
|---------|------|---------|---------|
| **部分成交**（用户下单 1000 张，只成交 600 张） | CLOB 流动性不足 | 自动解冻未成交部分（400 张），更新订单状态为 `partially_filled` | 无感知，仅看到成交数量 |
| **滑点偏差**（冻结 1000 USDC.e，实际花费 995.50） | 成交均价略优于/劣于预期 | 自动将小额差额（< 5 USDC.e）记入平台损益或用户账户 | 无感知 |
| **订单价格微调**（限价单以略优价格成交） | 对手盘价格更优 | 自动按实际成交价更新订单记录 | 无感知 |
| **Gas 费用偏移** | Polygon 网络波动 | 自动归入 gas 子账户，不从用户扣减 | 无感知 |

**原则**：不影响用户预期结果的小额偏差，系统自动消化，不触及业务订单。

##### ⏳ 象限Ⅱ：临时状态不一致 → 标记等待，不取消，不报错

| 异常场景 | 根因 | 处理方式 | 用户感知 |
|---------|------|---------|---------|
| **链上交易 pending**（冷钱包已发出，未确认） | Polygon 网络拥堵 | 标记为 `pending_confirmation`，下轮对账自动重试 | 订单显示「处理中」 |
| **市场已结算但链上资金未到账** | Oracle 结算后提现延迟 | 标记为 `pending_settlement`，设置超时时间（如 24h） | 订单显示「结算中」 |
| **Polymarket API 临时不可用** | API 限流/网络抖动 | 跳过当前批次，标记重试，累计 N 次后再升级 | 无感知（后管告警） |

**原则**：这些是链上生态的正常延迟，给系统一个自愈窗口，不要贸然取消订单。设置超时阈值，超时后升级为象限Ⅳ。

##### 🔴 象限Ⅲ：资金异常 → 暂停该订单交易，触发风控，不直接取消

| 异常场景 | 根因 | 处理方式 | 用户感知 |
|---------|------|---------|---------|
| **冷钱包下单金额 > 内部冻结金额** | Bug 或并发问题 | **立即暂停**该订单后续操作，冻结相关账户出金，通知运维人工核查 | 订单显示「风控审核中」 |
| **冷钱包缺少对应链上记录** | 内部账本已记账但实际未发链上交易 | **标记为孤儿单**，暂停处理，人工核查是否需补发交易或回滚账本 | 订单冻结，联系客服 |
| **内部有冻结记录但 Polymarket 无对应订单** | 订单提交失败但账本已扣 | **立即解冻**用户资金，通知用户订单未成交 | 看到解冻通知，可重新下单 |
| **链上成交金额与预期严重不符（> 10%）** | 高滑点或合约异常 | **暂停**订单结算，人工核查链上交易详情 | 订单显示「核算中」 |

**原则**：涉及资金安全的不确定性，先暂停（暂停交易/冻结出金），给人工介入时间。不直接取消，因为取消也可能导致资金错配。

##### ❌ 象限Ⅳ：确实不可执行的异常 → 取消订单 + 自动退款

| 异常场景 | 根因 | 处理方式 | 用户感知 |
|---------|------|---------|---------|
| **订单超过 30 分钟仍未在 Polymarket 找到对手盘** | 流动性不足/价格偏离 | **自动取消**，全额解冻，发通知 | 收到「订单超时未成交已取消」通知 |
| **市场被 Polymarket 取消/下架** | 事件提前取消 | **强制取消**所有关联订单，解冻资金 | 收到「市场已关闭」通知 |
| **内部系统重复下单**（同一订单被执行多次） | Bug | **取消多余订单**，回滚资金，告警开发 | 无感知（后台修正） |
| **冷钱包余额不足以执行聚合订单** | 大单超过冷钱包可用额度 | **取消订单**，通知用户「流动性不足」 | 收到取消通知 |

**原则**：确认该订单确实无法正常执行后，以保护用户资金为优先级，先取消退款再排查根因。

#### 6.4.3 异常处理决策树

```
对账发现异常
│
├── 是否为临时状态（链上 pending / API 超时）？
│   ├── 是 → ⏳ 标记等待，下轮重试，设置超时阈值（如 1h）
│   │         超时后仍未恢复 → 升级为资金异常或不可执行
│   └── 否 →
│
├── 是否涉及资金不确定？
│   ├── 否 → ❌ 订单是否确实不可执行？
│   │   ├── 是 → 取消订单 + 全额退款 + 通知用户
│   │   └── 否 → 记录异常到后管，继续监控
│   │
│   └── 是 → 金额是否在小额范围内（≤ 5 USDC.e）？
│       ├── 是 → ✅ 自动修复（记入损益），不通知用户
│       └── 否 → 🔴 暂停订单 + 冻结出金 + 人工介入
│
└── 最终: 所有人工处理的异常必须记录操作日志 + 审批链路
```

#### 6.4.4 用户端订单状态映射

| 对账异常类型 | 订单状态 | 用户端展示 | 可操作 |
|------------|---------|-----------|--------|
| 无异常 | `filled` / `settled` | 正常 | 可查看 |
| 自动修复 | `partially_filled` | 显示实际成交数量 | 可查看 |
| 等待重试 | `processing` | 「处理中，请稍候」 | 不可操作 |
| 暂停核查 | `risk_review` | 「风控审核中，请联系客服」 | 联系客服入口 |
| 确认取消 | `cancelled` | 「已取消」+ 原因 + 退款金额 | 可重新下单 |
| 市场关闭 | `market_closed` | 「市场已关闭，资金已自动退还」 | 可查看 |

#### 6.4.5 异常处理相关数据库表补充

```sql
-- 异常处理操作日志
CREATE TABLE anomaly_action_logs (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    anomaly_id          BIGINT NOT NULL COMMENT '关联异常ID',
    action_type         ENUM(
                            'auto_fix',         -- 自动修复
                            'manual_review',    -- 标记人工审核
                            'cancel_order',     -- 取消订单
                            'refund_user',      -- 退款用户
                            'freeze_account',   -- 冻结账户
                            'unfreeze_account', -- 解冻账户
                            're_execute',       -- 重新执行
                            'ignore'            -- 忽略
                        ) NOT NULL,
    action_detail       TEXT NOT NULL COMMENT '操作详情（含JSON）',
    operator_id         BIGINT COMMENT '操作人ID（系统=0）',
    operator_type       ENUM('system', 'admin') NOT NULL DEFAULT 'system',
    order_id            VARCHAR(64) COMMENT '关联业务订单ID',
    user_id             BIGINT COMMENT '关联用户ID',
    amount              DECIMAL(18,6) COMMENT '关联金额',
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_anomaly_id (anomaly_id),
    INDEX idx_operator (operator_id),
    INDEX idx_order_id (order_id)
);

-- 业务订单异常状态表
CREATE TABLE order_anomaly_status (
    id                  BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id            VARCHAR(64) NOT NULL COMMENT '业务订单ID',
    current_status      ENUM('normal', 'processing', 'risk_review', 'cancelled', 'market_closed') NOT NULL DEFAULT 'normal',
    anomaly_type        VARCHAR(50) COMMENT '最近异常类型',
    is_frozen           TINYINT(1) DEFAULT 0 COMMENT '是否冻结出金',
    freeze_reason       VARCHAR(200) COMMENT '冻结原因',
    resolved_at         DATETIME COMMENT '解决时间',
    created_at          DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    UNIQUE KEY uk_order_id (order_id),
    INDEX idx_status (current_status),
    INDEX idx_frozen (is_frozen)
);
```

---

## 7. 技术实现要点

### 7.1 数据获取方案

```
┌─────────────────────────────────────────────────────────────────────┐
│  数据源        │ 获取方式                    │ 保证机制               │
├─────────────────────────────────────────────────────────────────────┤
│ 内部账本       │ 直连 DB 查询                │ 数据库事务一致           │
│ 冷钱包USDC.e余额 │ Polygon RPC: balanceOf()   │ 多节点 fallback         │
│ 冷钱包交易记录   │ Polygon RPC: getLogs()     │ 从确认块开始扫描         │
│                │ + TxHash 信息补全           │ 确认数 ≥ 12 blocks      │
│ Polymarket订单  │ CLOB REST API + WebSocket  │ API Key 认证 + 限流重试 │
│ Polymarket成交  │ CLOB WS: FILL 事件         │ 本地持久化 + 幂等处理    │
│ Polymarket持仓  │ CLOB API: GET /positions   │ 定期快照                 │
│ Polymarket结算  │ Polygon 事件监听            │ CTF Exchange 合约事件    │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 关键合约地址

| 合约 | Polygon 地址 | 用途 |
|------|------------|------|
| **USDC.e (Bridged USDC)** | `0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174` | 查询冷钱包余额、解析转账事件 |
| **CTF Exchange** | `0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E` | 监听成交/结算事件 |
| **NegRisk_CTFExchange** | `0xC5d563A36AE78145C45a50134d48A1215220f80a` | 多结果市场成交监听 |
| **Conditional Tokens** | `0x4D97DCd97eC945f40cF65F87097ACe5EA0476045` | 查询 Outcome Token 持仓 |

### 7.3 幂等性与一致性

```
┌──────────────────────────────────────────────────────────────────┐
│  关键处理原则                                                      │
│                                                                   │
│  1. 所有对账操作必须幂等: 重复执行不产生副作用                          │
│     - 使用 batch_no + 业务ID 作为唯一约束                            │
│     - "已处理"的记录不再处理                                          │
│                                                                   │
│  2. 冷钱包链上交易处理:                                                │
│     - tx_hash 作为唯一标识                                          │
│     - 对账系统从区块 0 开始扫描，记录最后处理的 block_number               │
│     - 重启后从之前记录的 block_number 继续                              │
│                                                                   │
│  3. Polymarket 订单状态处理:                                          │
│     - WebSocket 实时 + REST API 定期全量校验                          │
│     - 订单状态机: OPEN → (PARTIAL_FILL) → FILLED / CANCELED        │
│                                                                   │
│  4. 内部账本:                                                        │
│     - 使用分布式锁保证同一订单不重复处理                                  │
│     - 所有账本变更记录必须包含 onchain_tx_hash 或 polymarket_order_id  │
└──────────────────────────────────────────────────────────────────┘
```

### 7.4 冷钱包地址管理

```sql
-- 冷钱包地址配置表
CREATE TABLE wallet_addresses (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    wallet_name     VARCHAR(50) NOT NULL COMMENT '钱包名称/标签',
    address         VARCHAR(64) NOT NULL UNIQUE COMMENT '钱包地址',
    wallet_type     ENUM('hot', 'warm', 'cold') NOT NULL COMMENT '钱包类型',
    chain           VARCHAR(20) NOT NULL DEFAULT 'polygon',
    is_active       TINYINT(1) DEFAULT 1,
    description     VARCHAR(200),
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- 插入默认冷钱包
INSERT INTO wallet_addresses (wallet_name, address, wallet_type, chain, description)
VALUES
    ('Cold Wallet - Main', '0x...冷钱包主地址...', 'cold', 'polygon', '冷钱包主体，持有大部分 USDC.e'),
    ('Cold Wallet - Polymarket', '0x...交易专用...', 'cold', 'polygon', '专门与 Polymarket CLOB 交互'),
    ('Hot Wallet - Taker', '0x...热钱包...', 'hot', 'polygon', '实时订单执行，小额撮合'),
    ('Fee Wallet', '0x...手续费...', 'warm', 'polygon', '平台收入归集');
```

---

## 8. 开发实施建议

### 8.1 分阶段实施

| 阶段 | 内容 | 工作量预估 | 优先级 |
|------|------|-----------|--------|
| **P1 - 基础搭建** | 数据库表创建 + 内部账本流水改造 + 冷钱包交易记录模块 | 3-5天 | P0 |
| **P2 - 余额级对账** | 第一级对账算法 + 冷钱包余额查询 + 后管 Dashboard | 3-4天 | P0 |
| **P3 - 交易级对账** | Polymarket 订单对接 + 链上交易扫描 + 二级匹配算法 | 5-7天 | P0 |
| **P4 - 持仓级对账** | Polymarket 持仓查询 + 内部持仓聚合 + 逐市场对账 | 3-5天 | P1 |
| **P5 - 异常处理** | 异常四象限分类 + 业务订单处理策略 + 告警通知 + 自动修复 + 异常操作日志 | 4-5天 | P1 |
| **P6 - 持续优化** | WebSocket 实时监听 + Sankey 资金流 + 数据导出 | 3-5天 | P2 |

### 8.2 依赖项

| 依赖 | 类型 | 说明 |
|------|------|------|
| Polygon RPC Endpoint | 基础设施 | 建议使用付费 RPC (Infura/QuickNode/Alchemy) |
| Polymarket API Key | 第三方 | 注册 Polymarket 开发者账号获取 |
| Polymarket CLOB 授权 | 第三方 | 冷钱包需完成 CLOB API 签名授权 |
| Redis | 基础设施 | 缓存订单簿数据，降低 API 调用频率 |
| 消息队列 (RabbitMQ/Kafka) | 基础设施 | 异步处理链上事件和 WebSocket 消息 |

### 8.3 技术选型建议

| 组件 | 推荐方案 | 备选 | 说明 |
|------|---------|------|------|
| 对账引擎语言 | Python (web3.py + py_clob_client) | Go / Node.js | Python 生态最成熟 |
| 链上数据获取 | QuickNode Streaming / Alchemy WebSocket | PolygonScan API | 付费 RPC 更稳定 |
| Polymarket 对接 | `py-clob-client` 库 | CLOB REST API 直连 | 官方 SDK 省去签名工作 |
| 定时调度 | Crontab / Airflow | XXL-Job | 根据现有技术栈选择 |
| 告警通知 | 钉钉机器人 / Slack Webhook | 邮件 / 短信 | 根据团队沟通工具选择 |
| 数据存储 | MySQL 8.0 + Redis | PostgreSQL | 与现有技术栈一致 |

---

## 9. 风险与注意事项

| 风险 | 影响 | 缓解措施 |
|------|------|---------|
| **Polymarket API 限流** | 对账延迟 | 本地缓存 + WebSocket 订阅 + 限流重试 |
| **链上交易重组** | 交易确认回滚 | 等待 12 个区块确认后再记账 |
| **Polymarket 市场长时间未结算** | 持仓对账持续异常 | 设置过期时间，> 30 天未结算标记为 stale |
| **冷钱包私钥风险** | 资金安全 | 多重签名 + 硬件 HSM + 操作审批流程 |
| **USDC.e 与 USDC 混合** | 金额计算错误 | 统一资产类型为 USDC.e，禁止混用 |
| **时区与时间窗口差** | 对账时间戳不匹配 | 统一使用 UTC 时间，窗口 ±5 分钟 |

---

## 10. 附录

### 10.1 术语表

| 术语 | 说明 |
|------|------|
| **内部账本** | 平台维护的用户资金余额数据库记录 |
| **冷钱包** | 平台控制的 Polygon 链上地址，存储 USDC.e |
| **Outcome Token** | Polymarket 产出代币（YES/NO 各一张），每对对应 1 USDC.e 抵押 |
| **Position** | Outcome Token 持仓 |
| **CLOB** | Central Limit Order Book，Polymarket 的链下限价订单簿 |
| **Oracle** | UMA 协议，负责裁决市场结果 |
| **Reconciliation** | 对账，对比两个系统的数据一致性 |
| **Slippage** | 滑点，成交价与预期价的偏差 |

### 10.2 参考文档

| 文档 | 链接 |
|------|------|
| Polymarket CLOB API 文档 | [https://docs.polymarket.com/api](https://docs.polymarket.com/api) |
| Polymarket 合约地址 | [https://docs.polymarket.com/contracts](https://docs.polymarket.com/contracts) |
| USDC.e Polygon 合约 | [Polygonscan USDC.e](https://polygonscan.com/token/0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174) |
| py-clob-client SDK | [https://github.com/Polymarket/py-clob-client](https://github.com/Polymarket/py-clob-client) |
| Web3.py 文档 | [https://web3py.readthedocs.io/](https://web3py.readthedocs.io/) |

### 10.3 快速对账 SQL 示例

```sql
-- 余额级对账：计算内部账本总负债
SELECT
    SUM(CASE WHEN asset = 'USDC.e' AND type IN ('deposit', 'exchange_in', 'order_settle_win', 'admin_adjust') THEN amount ELSE 0 END)
    - SUM(CASE WHEN asset = 'USDC.e' AND type IN ('withdrawal', 'exchange_out', 'order_place', 'platform_fee') THEN amount ELSE 0 END)
    AS total_ledger_balance
FROM ledger_transactions
WHERE status = 'confirmed'
  AND created_at <= NOW();

-- 交易级对账：查找未匹配的冷钱包交易
SELECT cw.*
FROM cold_wallet_transactions cw
LEFT JOIN ledger_transactions lt ON cw.tx_hash = lt.onchain_tx_hash
WHERE lt.id IS NULL
  AND cw.status = 'success';

-- 持仓级对账：查找内部系统有但 Polymarket 没有的持仓
SELECT up.market_id, up.outcome, SUM(up.quantity) as internal_qty
FROM user_positions up
LEFT JOIN polymarket_positions pp ON up.market_id = pp.market_id AND up.outcome = pp.outcome
WHERE up.settled = 0
  AND (pp.id IS NULL OR pp.quantity != up.quantity)
GROUP BY up.market_id, up.outcome;
```

---

*文档结束*