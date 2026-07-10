# DeTrade 充值奖励金风控与反作弊设计方案

## 1. 背景与目标 (Background & Objectives)

**背景**: DeTrade 平台计划上线“充值奖励金”活动（Deposit Bonus），即用户充值后按比例（如 10%-50%）赠送奖励金。
**痛点**: 平台支持 Metamask 钱包无缝登录，注册成本几乎为零。这极易招致“羊毛党”批量生成钱包地址，分散充值领取奖励，通过对敲（Hedging）套利，最后归集资金提现，导致平台资金损失。

**目标**: 
1.  **识别并拦截**批量注册的羊毛党。
2.  设计合理的**奖励金释放/解锁机制**，确保奖励金用于真实交易而非直接套现。
3.  建立**链上资金关联分析**，从资金流向维度进行风控。

---

## 2. 核心风控模型 (Core Risk Model)

针对 Metamask 注册的匿名性，我们采用 **"链下指纹 + 链上图谱 + 行为锁定"** 的三层风控模型。

### Layer 1: 身份与环境层 (Identity & Environment)
*虽然钱包地址无限，但物理设备和网络环境有限。*
*   **设备指纹 (Device Fingerprint)**: 记录 Device ID (基于浏览器 Canvas, Audio, WebGL 等特征)。
    *   **规则**: 同一 Device ID 最多允许 2 个账号领取充值奖励。超出限制的账号，充值到账但**不触发奖励发放**。
*   **IP 限制**: 
    *   **规则**: 同一 IP 段（/24）在 24 小时内领取奖励的账号数超过 5 个 -> 触发人工审核或暂停该 IP 段奖励。
    *   检测 VPN/代理 IP -> 标记为高风险，降低奖励比例或禁止领取。

### Layer 2: 链上资金关联层 (On-Chain Sybil Detection)
*这是针对 Web3 注册最有效的风控手段。羊毛党的资金通常来自同一个“母账户”或“交易所提现地址”，最后又归集到同一个地址。*
*   **资金同源检测 (Source Clustering)**:
    *   **规则**: 若 账号A 和 账号B 的充值资金来自**同一个父地址**（且该父地址非知名交易所热钱包），视为关联账户 (Sybil Cluster)。
    *   **处理**: 关联账户组内的奖励金自动冻结，要求进行视频 KYC 或直接封禁奖励。
*   **资金同去向检测 (Destination Clustering)**:
    *   **规则**: 若 账号A 和 账号B 提现至**同一个目标地址**，系统自动关联。
    *   **处理**: 触发提现拦截，人工审核。

### Layer 3: 奖励金生命周期管理 (Bonus Lifecycle)
*奖励金不是“白送的钱”，而是“交易的杠杆”。*
*   **状态机**: `Locked` (锁定) -> `Active` (可用) -> `Released` (可提现/已消耗) -> `Clawed_Back` (回收)。
*   **本金提现强行回收 (Clawback Rule)**:
    *   **核心逻辑**: 只要奖励金未完全解锁/消耗，用户**提取本金**时，系统必须**按比例或全额扣除**剩余奖励金。
    *   *公式*: `扣除奖励 = 剩余奖励 * (本次提现本金 / 总充值本金)`
    *   *目的*: 防止用户充值 1000U 拿 100U 奖励，不交易直接提现 1000U，白嫖 100U 奖励做无风险博弈。

---

## 3. 详细业务规则 (Detailed Rules)

### 3.1 奖励发放规则 (Distribution)
1.  **首充/复充区分**: 仅首充给予高比例（如 50%），复充给予低比例（如 10%）。
2.  **门槛限制**: 单笔充值 >= 100 USDT 方可激活奖励。
3.  **封顶限制**: 单账户累计奖励上限 5,000 USDT。

### 3.2 交易量解锁规则 (Volume Unlock)
奖励金默认进入 `bonus_balance`，可用于开仓抵扣保证金，但**不可提现**，且**亏损优先扣除奖励金**。
*   **解锁条件**: 需完成 `奖励金额 * N` 的交易量（Turnover）。
    *   *例如*: 充 1000 送 200，N=20。需完成 200 * 20 = 4000 USDT 的交易额。
*   **有效交易定义**:
    *   持仓时间 > 1 分钟。
    *   非对敲交易（即未被风控标记为 Self-Match）。

### 3.3 防对敲/刷量规则 (Anti-Hedging)
引用《深度系统设计文档》中的“AB仓对敲检测”：
*   禁止同一 IP/设备下的多账号进行**反向开仓**。
*   禁止关联账户组（资金同源）在同一时间窗口进行反向交易。

---

## 4. 数据库设计变更 (Database Schema Changes)

在 `user_assets` 表基础上，优化 `bonus_balance` 的管理，并新增风控记录表。

### 4.1 资产表扩展
```sql
ALTER TABLE user_assets ADD COLUMN bonus_locked DECIMAL(20, 8) DEFAULT 0; -- 锁定/待解锁的奖励
ALTER TABLE user_assets ADD COLUMN total_deposit DECIMAL(20, 8) DEFAULT 0; -- 累计充值（用于计算回收比例）
```

### 4.2 充值奖励记录表 (deposit_bonus_logs)
```sql
CREATE TABLE deposit_bonus_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    deposit_tx_hash VARCHAR(66),    -- 充值交易哈希
    deposit_amount DECIMAL(20, 8),  -- 充值金额
    bonus_amount DECIMAL(20, 8),    -- 奖励金额
    unlock_required DECIMAL(20, 8), -- 需要的交易量
    unlock_progress DECIMAL(20, 8), -- 当前已完成交易量
    
    status TINYINT, -- 0: Locked, 1: Active, 2: Clawed_Back, 3: Fully_Unlocked
    
    risk_tags JSON, -- 风控标签，例: ["same_ip", "same_source_address"]
    created_at TIMESTAMP
);
```

### 4.3 链上地址关联表 (chain_address_clusters)
```sql
CREATE TABLE chain_address_clusters (
    cluster_id VARCHAR(64), -- 聚类ID
    address VARCHAR(42),    -- 用户钱包地址
    relation_type TINYINT,  -- 1: 同源 (Source), 2: 同去向 (Destination)
    related_tx_hash VARCHAR(66)
);
```

---

## 5. 用户交互流程 (User Flow)

1.  **充值页面**:
    *   显示活动 Banner: "首充送 50%，最高 1000U"。
    *   提示文案: "*奖励金需完成 20倍 交易量方可提现；提现本金将导致奖励金按比例扣除。*" (明确告知规则，降低客诉)。

2.  **充值到账**:
    *   后端监听链上事件 -> 确认到账。
    *   **风控扫描**: 
        *   检查 Device Fingerprint。
        *   检查 Funding Source Address。
    *   **通过**: 资金入 `cash_balance`，奖励入 `bonus_balance`。
    *   **拒绝**: 仅资金入账，不发奖励，发送站内信说明原因。

3.  **提现请求**:
    *   系统计算 `Max_Withdrawable = Cash_Balance - (Locked_Bonus * Ratio)`。
    *   若用户申请提现金额 > 可提现额度，弹窗提示：“您当前的提现将导致 XX USDT 奖励金被回收，是否继续？”

---

## 7. 合规与用户协议条款 (Compliance & Terms)

*基于 detrade.com (Nexus Play LLC) 的监管背景，风控措施必须有明确的法律条款支持，以应对用户投诉。*

### 7.1 滥用条款 (Abuse Policy)
在用户协议中需明确定义“滥用行为”，包括但不限于：
*   **多账户操作**: "Registration of multiple accounts by a single user or entity to manipulate bonus schemes."
*   **对敲/关联交易**: "Coordinated trading between accounts to exploit market depth or bonus rules."
*   **资金归集**: "Pooling of funds from a single source across multiple accounts."

### 7.2 处置权声明 (Right to Action)
平台必须保留以下权利，且无需提前通知：
*   "Nexus Play LLC reserves the right to **revoke any bonuses** and **confiscate profits** derived from bonus abuse." (收回奖励及由奖励产生的利润)
*   "Accounts flagged for suspicious activity may be subject to **enhanced KYC verification** (Video verification) before withdrawal." (强制视频认证)
*   "We reserve the right to block access from specific IP addresses or device identifiers." (封禁设备/IP)

### 7.3 免责声明 (Disclaimer)
*   "Participation in the Deposit Bonus program constitutes full acceptance of these Anti-Fraud rules."
*   针对受限地区（美国、欧盟等），明确奖励活动不适用，若通过技术手段绕过限制领取，平台有权全额冻结。

---

## 8. 总结
本方案通过**提高攻击成本**（资金占用、交易磨损）和**技术拦截**（指纹、链上分析）相结合，在保障真实用户体验的前提下，最大程度遏制 Metamask 无门槛注册带来的薅羊毛风险。同时，配合完善的**用户协议条款**，确保平台在执行风控拦截时有法可依。
