# CEX KYC 体系竞品调研报告

## 1. 调研概述
**调研对象**: Binance, OKX, Bybit
**调研目标**: 梳理主流 CEX 的 KYC 分级体系、触发场景及认证流程，为 DeTrade 平台设计提供参考。

---

## 2. 核心竞品 KYC 体系对比

### 2.1 分级体系 (Tiered Structure)

| 维度 | **Binance** | **OKX** | **Bybit** | **DeTrade 建议方案** |
| :--- | :--- | :--- | :--- | :--- |
| **Level 1 (基础)** | **Verified** <br> Req: 身份证 + 人脸 <br> Limit: 充提 50k/日, P2P 限制 | **Basic** <br> Req: 个人信息 (无文档) <br> Limit: 极低, 无法提币 | **Lv.1** <br> Req: 身份证 + 人脸 <br> Limit: 提币 1M/日, P2P 无限 | **Lv.1 (基础认证)** <br> Req: ID + Face <br> Limit: 提币 20k/日 |
| **Level 2 (进阶)** | **Verified Plus** <br> Req: 地址证明 (POA) <br> Limit: 充提 2M/日, 法币无限制 | **Advanced** <br> Req: ID + 人脸 <br> Limit: 提币 10M/日 | **Lv.2** <br> Req: 地址证明 (POA) <br> Limit: 提币 2M/日 | **Lv.2 (高级认证)** <br> Req: POA <br> Limit: 提币 500k/日 |
| **Level 3 (企业/定制)** | **Custom** <br> Req: 资产证明 + 视频面试 | **Video Verification** <br> Req: 视频面试 (高风险触发) | **Business** <br> Req: 企业全套文档 | **Lv.3 (机构/大户)** <br> Req: 人工审核 |

### 2.2 核心触发场景 (Trigger Scenarios)

1.  **注册即强制 (Mandatory)**: Binance 和 OKX 目前均要求**必须完成 Lv.1 才能进行任何交易**（充值、交易、提现全卡）。Bybit 允许注册，但提现必须 KYC。
2.  **提现限额 (Withdrawal Limit)**: 触发特定额度（如 > 1BTC/日）时，强制要求 Lv.2。
3.  **法币交易 (Fiat/P2P)**: 涉及法币出入金，**必须 Lv.1**，部分地区要求 Lv.2。
4.  **Launchpad/理财**: 参与 IEO 打新，必须完成 Lv.2。
5.  **风控触发 (Risk Trigger)**:
    *   **设备指纹异常**: 新设备登录且 IP 跨国。
    *   **资金来源可疑**: 链上 AML 标记黑地址转入。
    *   **高频小额提现**: 疑似蚂蚁搬家。

---

## 3. DeTrade KYC 策略建议

### 3.1 策略定位
*   **初期 (MVP)**: 采用 **"宽进严出"** 策略。注册不强制 KYC，但在**提现**和**参与营销活动（签到/返水）**时强制 KYC。
*   **理由**: 降低新用户门槛，先把人留住；但在资金流出和薅羊毛环节做强风控。

### 3.2 建议分级
*   **Lv.0 (未认证)**: 仅可充值、交易。**不可提现**，不可领签到红包。
*   **Lv.1 (身份认证)**: 身份证/护照 + 人脸。**解锁提现 (20k/日)**，解锁签到/返水。
*   **Lv.2 (地址认证)**: 水电单/银行账单。**解锁高额提现 (500k/日)**，解锁 Launchpad。

---

## 4. 附件
*   *Reference*: [Binance KYC Guide](https://www.binance.com/en/support/faq/how-to-complete-identity-verification-360027287111)
