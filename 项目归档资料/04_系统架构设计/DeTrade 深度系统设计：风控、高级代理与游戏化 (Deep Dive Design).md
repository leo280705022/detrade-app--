# DeTrade 深度系统设计：风控、高级代理与游戏化 (Deep Dive Design)

## 1. 文档概述
**文档版本**: v1.0  
**更新日期**: 2026-01-05  
**目标**: 在基础功能与增长体系之上，针对 DeTrade 的**高杠杆/二元期权**属性，深入设计核心的**风控防刷体系**、**高级代理商模式**以及**游戏化交易玩法**。这些是离岸交易所生存与盈利的关键护城河。

---

## 2. 核心风控体系设计 (Risk Control & Anti-Fraud)
*目标：在撒钱做增长的同时，防止“羊毛党”和“攻击者”掏空平台资金。*

### 2.1 体验金防套利系统 (Bonus Abuse Detection)
体验金（Trial Fund）最大的风险是用户通过 **"AB仓对敲"** (Hedging) 锁定利润。即：账号A开多，账号B开空，无论涨跌，总有一个账号盈利并提现。

**防御策略**:
1.  **设备与环境指纹 (Device Fingerprint)**:
    *   集成指纹服务 (如 FingerprintJS Pro)，采集 Device ID, IP, Canvas 指纹, 电池电量, 屏幕分辨率等。
    *   **规则**: 同一 Device ID 或 相似指纹 注册超过 2 个账号 -> **自动封禁**或**取消奖励资格**。
2.  **同频交易检测 (Coordinated Trading)**:
    *   监控 **同一时间窗口 (e.g., 1秒内)**、**同一交易对**、**反向开仓** 的行为。
    *   **规则**: 若 账号A 和 账号B 经常在同一秒内一多一空 -> 标记为关联账号 (Sybil Cluster)，冻结资金。
3.  **持仓限制 (Position Limit)**:
    *   体验金仓位 **禁止双向持仓** (Hedge Mode Disabled)。
    *   体验金仓位 **最大持仓时间限制** (e.g., 24小时强制平仓)，增加对敲难度。

### 2.2 提现风控 (Withdrawal Risk Engine)
提现是最后一道防线。
*   **T+N 限制**: 使用体验金盈利的部分，需完成 **X 倍交易额 (Turnover)** (e.g., 盈利额的 10 倍) 方可解锁提现。
*   **C-Score 用户画像**:
    *   基于用户行为打分。正常交易用户 (高分) -> 自动提现 (Auto-Withdraw)。
    *   疑似刷单用户 (低分) -> 进入人工审核队列 (Manual Review)。
    *   *扣分项*: 注册即全仓梭哈、只做二元期权不做合约、IP 归属地为高风险地区。

---

## 3. 高级代理商系统 (Advanced Affiliate System)
*目标：不仅让 KOL 赚钱，更让他们拥有“自己开交易所”的感觉。*

### 3.1 独立代理后台 (Agent Portal)
除了 APP 内的简单邀请页，为 Lv.3 以上代理商提供 **Web 端独立后台 (Agent SAAS)**。
*   **功能**:
    *   **下级透视**: 查看所有下级用户的注册时间、入金量、交易量、持仓偏好。
    *   **佣金自定义**: 允许大代理商**手动下放佣金比例**给下级。
        *   *例*: 平台给 A 代理 70%，A 可以给他的下级 B 设置 40%，自己吃 30% 差价。
    *   **营销支持**: 代理商可申请生成“注册送 10U”的专属链接，成本由代理商或平台分摊。

### 3.2 亏损分红模式 (Loss Share / 客损模式) - *高风险高收益*
这是离岸盘最核心的激励模式，仅对**顶级信任代理**开放。
*   **逻辑**: 代理商不拿手续费返佣，而是拿 **用户亏损 (Net Loss) 的分成**。
*   **比例**: 通常 30% - 50%。
*   **结算**: 月结 (需扣除用户盈利部分)。
*   *注意*: 此模式极易引发道德风险 (代理商带单故意让用户亏损)，需严格风控。

---

## 4. 游戏化与社交交易深度 (Gamification & Social)
*目标：利用“赌性”提升粘性，DeTrade 的核心差异化。*

### 4.1 PVP 对战 (Bull vs Bear Battle)
将二元期权包装成多人对战游戏。
*   **玩法**:
    *   进入“比特币 1分钟 PVP 房间”。
    *   用户选择阵营：**多军 (Bull)** vs **空军 (Bear)**。
    *   双方下注，奖池累计。
    *   1分钟后，若涨，多军平分空军的下注金 (平台抽水 5%)。
*   **优势**: 强社交属性，即时反馈，极易上瘾。

### 4.2 跟单带单系统 (Copy Trading 2.0)
*   **带单员 (Master Trader) 激励**:
    *   **10% 盈利分红 (Profit Share)**: 跟单者赚钱了，分 10% 给带单员。
    *   **带单补贴**: 即使亏损，只要交易量达标，平台给予基础补贴。
*   **一键跟单**:
    *   支持 **“定额跟单”** (每单跟 10U) 和 **“比例跟单”** (按资金比例)。
    *   风控：跟单者可设置最大止损线 (Max Drawdown)，触达即自动停止跟单。

---

## 5. 核心数据模型设计 (Database Schema Concept)

为了支撑上述复杂的资金与返佣逻辑，数据库设计需采用 **双账户 + 流水账** 模式。

### 5.1 资产账户表 (Assets)
```sql
TABLE user_assets (
    user_id BIGINT,
    currency VARCHAR(10) (e.g., 'USDT'),
    
    -- 现金账户 (可提现)
    cash_balance DECIMAL(20, 8),
    frozen_balance DECIMAL(20, 8), -- 下单冻结
    
    -- 体验金账户 (不可提现，优先扣除)
    bonus_balance DECIMAL(20, 8),
    
    -- 抵扣金账户
    coupon_balance DECIMAL(20, 8),
    
    updated_at TIMESTAMP
);
```

### 5.2 返佣流水表 (Commission_Logs)
```sql
TABLE commission_logs (
    log_id BIGINT PRIMARY KEY,
    source_user_id BIGINT,      -- 贡献者 (交易的人)
    target_user_id BIGINT,      -- 受益者 (代理商)
    order_id BIGINT,            -- 关联订单
    
    commission_type TINYINT,    -- 1: 手续费返佣, 2: 亏损分红
    amount DECIMAL(20, 8),      -- 返佣金额
    tier_level TINYINT,         -- 返佣层级 (1级/2级)
    
    status TINYINT,             -- 0: 待结算, 1: 已发放, 2: 风控拦截
    settle_time TIMESTAMP       -- 结算时间 (实时或T+1)
);
```

### 5.3 资金流转逻辑 (State Machine)
1.  **开仓**: 优先扣除 `bonus_balance` -> 其次扣除 `cash_balance`。
2.  **平仓 (盈利)**: 盈利部分全部进入 `cash_balance` (变为可提现)。
3.  **平仓 (亏损)**: 优先扣除 `bonus_balance` -> 不够再扣 `cash_balance`。
4.  **提现**: 检查 `cash_balance` 是否充足，且触发 `Risk_Engine` 检查 `Turnover` 是否达标。

---

## 6. 总结
本深度设计文档补充了 DeTrade 作为一个商业化交易所最隐秘但也最重要的部分：
1.  **怎么防止被薅羊毛** (风控)。
2.  **怎么让代理商疯狂推人** (独立后台+自定义佣金)。
3.  **怎么让用户疯狂交易** (PVP+跟单)。
4.  **底层钱怎么算** (双账户模型)。

这将构建起 DeTrade 的核心竞争力。
