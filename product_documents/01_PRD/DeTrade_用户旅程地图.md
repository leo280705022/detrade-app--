# DeTrade 签到与返水系统用户旅程地图 (User Journey Map)

## 1. 旅程概述 (Overview)

本旅程地图展示了用户从 **每日登录签到** -> **受激励交易** -> **产生亏损** -> **次日获得返水** -> **连续登录解锁资金** 的完整闭环。

核心设计目标是利用 **"限时免息"** 刺激开单，利用 **"锁仓返水"** 锁死次日留存。

## 2. 核心流程图 (Flowchart)

```mermaid
flowchart TD
    %% Nodes
    Start([用户登录 Day T]) --> CheckAssets{资产 > 10U?}
    
    subgraph Daily_CheckIn [系统 A: 每日签到 (促活)]
        CheckAssets -- 否 --> PromptDeposit[提示充值]
        CheckAssets -- 是 --> ClickCheckIn[点击签到]
        ClickCheckIn --> GetReward[获得积分/盲盒]
        GetReward --> TriggerHappyHour[触发限时免息\n(1小时手续费 5 折)]
    end
    
    subgraph Trading_Activity [交易阶段]
        TriggerHappyHour --> Trade[用户交易开单]
        Trade --> Settle[每日结算\n(UTC 00:00)]
        Settle --> CalcUNV{计算 UNV\n& 净亏损}
    end
    
    subgraph Lossback_System [系统 B: 亏损返水 (留存)]
        CalcUNV -- 无亏损 --> NoRebate[无返水]
        CalcUNV -- 净亏损 > 0 --> CalcRebate[计算返水金额\n(基于 UNV)]
        
        CalcRebate --> AmountCheck{金额 >= 5U?}
        
        AmountCheck -- 否 (< 5U) --> InstantAll[100% 立即到账]
        
        AmountCheck -- 是 (>= 5U) --> SplitRelease[分批释放模式]
        SplitRelease --> InstantPart[20% 立即到账]
        SplitRelease --> LockedPart[80% 进入 7日回血包]
    end
    
    subgraph Vesting_Calendar [7日锁仓日历 (粘性闭环)]
        LockedPart --> LoginT1{Day T+1 登录?}
        LoginT1 -- 是 --> Unlock1[解锁 1/7]
        LoginT1 -- 否 --> Burn1[当日份额销毁]
        
        Unlock1 --> LoginT2{Day T+2 登录?}
        LoginT2 -- 是 --> Unlock2[解锁 1/7]
        LoginT2 -- 否 --> Burn2[当日份额销毁]
        
        Unlock2 -.-> EndLoop[...]
    end
    
    InstantAll --> UserBalance[用户余额\n(现金)]
    InstantPart --> UserBalance
    Unlock1 --> UserBalance
    Unlock2 --> UserBalance
    
    %% Styling
    style Start fill:#f9f,stroke:#333,stroke-width:2px
    style TriggerHappyHour fill:#ff9,stroke:#f66,stroke-width:2px,stroke-dasharray: 5 5
    style LockedPart fill:#bbf,stroke:#333,stroke-width:2px
    style UserBalance fill:#9f9,stroke:#333,stroke-width:2px
```

## 3. 关键触点详解 (Touchpoints)

### 阶段一：签到与促活 (Day T 09:00)
*   **用户行为**: 打开 APP，点击首页 "每日签到"。
*   **系统反馈**:
    1.  弹出盲盒动画，获得 "100 积分"。
    2.  **关键时刻 (MOT)**: 弹窗提示 *"恭喜触发限时免息！未来 1 小时开单，手续费 5 折！"*
*   **用户心理**: "反正要交易，不如趁现在免息开一单。" -> **成功促进交易**。

### 阶段二：交易与亏损 (Day T 09:30 - 23:59)
*   **用户行为**: 进行合约交易。不幸遭遇行情波动，净亏损 1000 U。
*   **后台逻辑**:
    *   UTC 00:00 结算。
    *   计算 `UNV = 1000 - (代理佣金) - (平台利润) = 50 U`。
    *   计算返水: `50 * 50% = 25 U`。

### 阶段三：返水发放与锁仓 (Day T+1 09:00)
*   **用户行为**: 次日再次登录。
*   **系统反馈**:
    *   弹窗: *"昨日亏损补偿 25 U 已发放！"*
    *   **20% (5 U)**: 叮！钱包余额增加 5 U。
    *   **80% (20 U)**: 进入 "7日回血包"。
*   **用户心理**: "还有 20 U 没拿完，明天还得来。" -> **成功锁死留存**。

### 阶段四：持续解锁 (Day T+2 ~ T+7)
*   **用户行为**: 每天登录点击 "领取"。
*   **系统反馈**: 每天到账 `20 / 7 ≈ 2.85 U`。
*   **风险控制**: 若某天用户未登录，当日的 2.85 U 自动销毁，不可找回。

## 4. 总结
该旅程通过 **"签到诱导交易"** 和 **"返水强制留存"** 两个闭环，将用户的活跃度与平台的利益深度绑定。
