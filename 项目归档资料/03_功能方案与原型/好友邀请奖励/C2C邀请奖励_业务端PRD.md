# C2C 邀请奖励 - 业务端 PRD

> 版本：v2.0  
> 日期：2026-07-21  
> 状态：待评审  
> 变更说明：v2.0 对齐后管 v4.1 设计，补充活动动态加载、无活动状态、文案自动生成  
> 适用端：Web / H5 / APP

---

## 1. 概述

### 1.1 背景

为提升 C2C 业务用户增长，通过阶梯式邀请奖励机制激励现有用户邀请新好友注册并完成交易，实现用户裂变式增长。

### 1.2 与后管的关联

后管（活动管理 > C2C 邀请奖励）负责活动的创建、配置和上下线管理。业务端根据后管配置**动态加载**当前在线活动并展示。

| 后管操作 | 业务端影响 |
|---------|-----------|
| 新建活动（未上线） | 无影响，业务端不展示 |
| 活动上线 | 业务端实时加载新活动配置，替换展示 |
| 活动下线 | 业务端切换为「无活动」状态 |
| 活动到期（结束时间到达） | 系统自动标记为已结束，业务端切换为「无活动」状态 |
| 修改阶梯/条件/风控 | 业务端实时同步新配置 |

### 1.3 核心约束

- **全局仅一个活动在线**：业务端同一时间只展示一个活动
- **活动配置后管可控**：业务端不硬编码任何活动参数，全部从接口读取
- **无活动时友好提示**：当前无在线活动时，展示「暂无活动」页面

---

## 2. 活动规则

### 2.1 活动展示

活动标题和规则说明**不由后管填写**，而是由业务端根据后管配置的阶梯数据自动生成。

**活动标题生成规则：**

| 语言 | 模板 | 示例 |
|------|------|------|
| EN | Invite Friends, Earn up to {max_reward} USDT | Invite Friends, Earn up to 770 USDT |
| ZH | 邀请好友，最高赚取 {max_reward} USDT | 邀请好友，最高赚取 770 USDT |

> `{max_reward}` = 所有阶梯奖励金额之和（累计奖励上限）

**活动副标题：**

| 语言 | 模板 |
|------|------|
| EN | Complete {start_date} - {end_date} |
| ZH | 活动时间：{start_date} - {end_date} |

### 2.2 有效邀请条件

被邀请好友需**同时满足**以下条件（条件值由后管配置）：

| 步骤 | 条件 | 后管配置字段 | 默认值 |
|------|------|------------|--------|
| 1 | 注册 | - | - |
| 2 | 首次充值 ≥ {deposit_threshold} USDT | 首次充值门槛 | 20 USDT |
| 3 | 真实有效交易额 ≥ {trade_threshold} USDT | 交易额门槛 | 200 USDT |

> 条件值从后管接口读取，非硬编码。

### 2.3 阶梯奖励结构

阶梯数据完全由后管配置，业务端动态渲染。示例（后管配置 6 个阶梯时）：

| 阶梯 | 有效邀请人数 | 奖励金额 (USDT) | 解锁方式 |
|------|-------------|----------------|---------|
| 1 | 1 人 | 5 | 自动解锁 |
| 2 | 3 人 | 15 | 自动解锁 |
| 3 | 5 人 | 30 | 自动解锁 |
| 4 | 10 人 | 70 | 自动解锁 |
| 5 | 20 人 | 150 | 自动解锁 |
| 6 | 50 人 | 500 | 自动解锁 |

**累计奖励上限：所有阶梯金额之和**

### 2.4 奖励计算规则

- 阶梯制：达到对应人数即解锁该层奖励
- 奖励可叠加：邀请 5 人时，已解锁 5U + 15U + 30U = 50U
- 奖励发放：达到条件后自动发放至用户账户余额
- 每个阶梯仅可领取一次

---

## 3. 页面状态机

### 3.1 页面状态

业务端邀请页面有 3 种状态：

```
页面加载
   ↓
调用接口获取当前活动
   ↓
┌──────────────┐
│ 有在线活动？  │
└──────┬───────┘
   Yes ↓        No ↓
┌──────────┐  ┌──────────────┐
│ 活动页面  │  │ 暂无活动页面  │
└──────────┘  └──────────────┘
```

### 3.2 状态说明

| 状态 | 触发条件 | 展示内容 |
|------|---------|---------|
| 活动页面 | 接口返回有效活动（status=已上线） | 完整活动页面（Banner + 进度 + 阶梯 + 邀请 + 记录） |
| 暂无活动 | 接口返回无活动 / 活动已过期 | 空状态页：「No active event at the moment. Stay tuned!」 |
| 加载中 | 接口请求中 | Loading 骨架屏 |
| 接口异常 | 接口超时/报错 | 错误提示 + 重试按钮 |

### 3.3 暂无活动页面

```
┌─────────────────────────────────────┐
│                                     │
│          [插画/图标]                 │
│                                     │
│    No active event at the moment.   │
│    Stay tuned for upcoming events!  │
│                                     │
│         [ Refresh ]                 │
│                                     │
└─────────────────────────────────────┘
```

---

## 4. 页面功能设计（活动页面）

### 4.1 页面结构

```
┌─────────────────────────────────────┐
│  A. 活动 Banner / Header            │
│     标题（自动生成）+ 活动时间        │
├─────────────────────────────────────┤
│  B. 邀请进度区域                     │
│     进度条 + 当前进度 + 冲刺提示      │
├─────────────────────────────────────┤
│  C. 阶梯奖励卡片区域                 │
│     动态渲染阶梯卡片（数量由后管决定）  │
├─────────────────────────────────────┤
│  D. 邀请操作区域                     │
│     邀请链接 / 邀请码 / 二维码 / 分享  │
├─────────────────────────────────────┤
│  E. 活动规则说明（自动生成）          │
│     根据配置动态生成规则文案           │
├─────────────────────────────────────┤
│  F. 我的邀请记录                     │
│     好友列表（状态/进度）             │
├─────────────────────────────────────┤
│  G. 我的奖励记录                     │
│     奖励明细列表                      │
└─────────────────────────────────────┘
```

### 4.2 各区域功能说明

**A. 活动 Banner**

| 元素 | 数据来源 | 说明 |
|------|---------|------|
| 背景图 | 后管 Banner 上传（可选） | 有 Banner 用 Banner，无 Banner 用默认品牌背景 |
| 活动标题 | 自动生成 | "Invite Friends, Earn up to {max_reward} USDT" |
| 活动时间 | 后管配置 | 显示活动有效期 |

**B. 邀请进度区域**

| 元素 | 说明 |
|------|------|
| 进度条 | 可视化显示当前邀请进度 |
| 当前进度 | "X / Y" 格式，X=当前有效邀请数，Y=下一阶梯目标 |
| 提示文案 | "Only N more friends to unlock Z USDT Bonus." |
| 已达最高阶梯 | 显示 "Congratulations! All rewards unlocked!" |

**C. 阶梯奖励卡片**

| 元素 | 说明 |
|------|------|
| 卡片数量 | **动态**，由后管配置的阶梯数决定 |
| 奖励金额 | 显示每个阶梯的 USDT 奖励 |
| 状态标识 | ✅ 已解锁 / 🔒 未解锁 / 🎯 当前目标 |
| 邀请人数 | "1 friend" / "3 friends" 等 |
| 当前目标高亮 | 正在冲刺的阶梯卡片高亮显示 |

**D. 邀请操作区域**

| 元素 | 说明 |
|------|------|
| 邀请链接 | 带邀请码的专属链接 |
| 复制链接 | 一键复制 |
| 分享按钮 | 社交平台分享 / 生成海报 |
| 邀请码 | 用户专属邀请码 |
| 二维码 | 专属邀请二维码，支持保存 |

**E. 活动规则说明（自动生成 + 后管可覆盖）**

折叠/展开形式。规则内容来源优先级：

1. **后管自定义规则**：若后管配置了 `customRules` 字段且非空，直接使用该内容展示
2. **自动生成模板**：若 `customRules` 为空，按以下模板自动生成：

| 语言 | 生成模板 |
|------|---------|
| EN | "Invite friends to register. Each friend must complete a first deposit of {deposit} USDT and trading volume of {trade} USDT to count as a valid invite. Rewards are automatically distributed to your balance upon reaching each tier threshold." |
| ZH | "邀请好友注册，每位好友需完成首次充值 {deposit} USDT 且交易额达到 {trade} USDT 方计为有效邀请。达到阶梯条件后奖励自动发放至账户余额。" |

> `{deposit}` = `conditions.depositThreshold`，`{trade}` = `conditions.tradeThreshold`，均从后管接口读取。

**F. 我的邀请记录**

| 字段 | 说明 |
|------|------|
| 好友信息 | 脱敏显示（如 138****1234 或 user***@gmail.com） |
| 注册时间 | 好友注册时间 |
| 进度 | 合并显示：充值状态 + 交易额（如 "已充值 500U / 交易 500U"） |
| 邀请状态 | 进行中 / 有效 / 无效 |
| 无效原因 | 触发风控原因（如 IP 相同、钱包相同等） |

**G. 我的奖励记录**

| 字段 | 说明 |
|------|------|
| 奖励阶梯 | "Tier 1 - 1 friend" |
| 奖励金额 | +5 USDT |
| 发放时间 | 解锁时间 |
| 状态 | 已发放 / 待发放 |

---

## 5. 交互流程

### 5.1 页面加载流程

```
用户进入邀请页面
       ↓
显示 Loading 骨架屏
       ↓
调用 GET /api/invite/activity
       ↓
┌──────────────────┐
│ 返回有活动数据？   │
└────┬─────────────┘
 Yes ↓          No ↓
渲染活动页面    渲染「暂无活动」页面
       ↓
调用 GET /api/invite/my-progress
       ↓
渲染用户进度、邀请记录、奖励记录
```

### 5.2 用户邀请流程

```
用户A 点击邀请 → 生成/展示邀请链接
       ↓
分享链接给用户B（复制链接/分享/二维码）
       ↓
用户B 通过链接注册 → 系统记录邀请关系
       ↓
用户B 完成首次充值 ≥ {deposit_threshold} USDT → 进度更新
       ↓
用户B 完成交易额 ≥ {trade_threshold} USDT → 标记为有效邀请
       ↓
系统检查阶梯奖励 → 达到条件自动发放
       ↓
用户A 收到奖励通知 → 奖励到账
```

### 5.3 进度展示逻辑

```
当前有效邀请数 = N
阶梯配置 = [{tier: 1, count: 1, reward: 5}, {tier: 2, count: 3, reward: 15}, ...]

找到下一个未解锁阶梯 next_tier:
  IF 无未解锁阶梯:
    显示: "Congratulations! All rewards unlocked!"
    进度条: 满
  ELSE:
    显示: "Only {next_tier.count - N} more friends to unlock {next_tier.reward} USDT"
    进度: N / next_tier.count
```

---

## 6. 风控规则

### 6.1 校验维度（与后管对齐）

| 维度 | 规则 | 后管开关 | 默认 |
|------|------|---------|------|
| IP 校验 | 邀请人与被邀请人 IP 不能相同 | IP 校验 | 开启 |
| 钱包地址 | 不能使用相同充值钱包 | 钱包地址校验 | 开启 |
| 设备指纹 | 同设备只能绑定一个邀请关系 | 设备指纹校验 | 开启 |
| 交易行为 | 检测对冲交易/刷量行为 | 交易对冲检测 | 开启 |
| 邀请频率 | 单用户每日不超过上限 | 单日邀请上限 | 20 |

> 以上 5 项均由后管配置开关，业务端不直接展示风控细节，仅在邀请被判定无效时展示原因。

### 6.2 无效邀请原因展示

| 原因码 | 展示文案 (EN) | 展示文案 (ZH) |
|--------|-------------|-------------|
| SAME_IP | Same IP address | 相同 IP 地址 |
| SAME_WALLET | Same deposit wallet | 相同充值钱包 |
| SAME_DEVICE | Same device | 相同设备 |
| HEDGING | Suspicious trading behavior | 可疑交易行为 |
| BELOW_THRESHOLD | Requirements not met | 未达标 |
| DAILY_LIMIT | Daily invite limit reached | 已达每日邀请上限 |

### 6.3 异常处理

| 场景 | 处理方式 |
|------|---------|
| 触发风控 | 标记为无效邀请，不计算进度，展示原因 |
| 申诉 | 用户可提交申诉，人工审核 |
| 批量异常 | 冻结奖励，触发后管人工审核 |

---

## 7. 通知机制

| 触发事件 | 通知方式 | 通知内容 (EN) |
|---------|---------|-------------|
| 好友完成注册 | 站内信 / Push | "Your friend has registered! Complete deposit & trading to unlock rewards." |
| 好友完成充值 | 站内信 / Push | "Your friend has deposited! Keep trading to unlock rewards." |
| 好友成为有效邀请 | 站内信 / Push | "Congratulations! Your friend completed all tasks. Invite more for bigger rewards!" |
| 阶梯奖励解锁 | 站内信 / Push | "You've unlocked Tier {X} reward: {Y} USDT has been added to your balance." |
| 达到最高阶梯 | 站内信 / Push | "Amazing! You've unlocked all rewards!" |
| 活动即将结束 | 站内信 | "The invite event ends in {N} days. Hurry and invite more friends!" |

---

## 8. 接口设计

| 接口 | 方法 | 说明 |
|------|------|------|
| /api/invite/activity | GET | 获取当前在线活动配置（阶梯/条件/时间/Banner） |
| /api/invite/my-progress | GET | 获取我的邀请进度和奖励状态 |
| /api/invite/link | GET | 获取我的邀请链接/邀请码 |
| /api/invite/friends | GET | 获取邀请好友列表（分页） |
| /api/invite/rewards | GET | 获取奖励记录列表（分页） |
| /api/invite/share | POST | 生成分享海报/链接 |

### 8.1 /api/invite/activity 返回结构

```json
{
  "hasActivity": true,
  "activity": {
    "id": 1001,
    "name": "Invite & Earn Q3",
    "startTime": "2026-07-01T00:00:00Z",
    "endTime": "2026-09-30T23:59:59Z",
    "bannerUrl": "https://cdn.xxx.com/banner.jpg",
    "tiers": [
      { "tier": 1, "inviteCount": 1, "reward": 5 },
      { "tier": 2, "inviteCount": 3, "reward": 15 },
      { "tier": 3, "inviteCount": 5, "reward": 30 },
      { "tier": 4, "inviteCount": 10, "reward": 70 },
      { "tier": 5, "inviteCount": 20, "reward": 150 },
      { "tier": 6, "inviteCount": 50, "reward": 500 }
    ],
    "conditions": {
      "depositThreshold": 20,
      "tradeThreshold": 200,
      "kycRequired": "KYC1"
    },
    "customRules": "",
    "totalRewardCap": 770
  }
}
```

> 无活动时返回 `{ "hasActivity": false, "activity": null }`

---

## 9. 多端适配

| 端 | 适配要点 |
|----|---------|
| Web | 最大宽度 1200px 居中，阶梯卡片横向排列，邀请记录表格形式 |
| H5 / TG Mini App | 全屏移动端，阶梯卡片 2 列网格或横滑，卡片列表形式，系统原生分享 |
| APP | 原生导航栏，Push 通知奖励到账，二维码保存到相册 |

---

## 10. 数据埋点

| 事件 | 参数 | 说明 |
|------|------|------|
| invite_page_view | source, platform, activity_id | 邀请页访问 |
| invite_no_activity | platform | 无活动页面展示 |
| invite_link_copy | user_id, activity_id | 复制链接 |
| invite_share | share_type, activity_id | 分享操作 |
| invite_friend_register | inviter_id, invitee_id | 好友注册 |
| invite_tier_unlock | user_id, tier, amount | 阶梯解锁 |
| invite_reward_claim | user_id, tier, amount | 奖励发放 |

---

## 11. 多语言关键文案

| 中文 | English |
|------|---------|
| 邀请好友，最高赚取 {X} USDT | Invite Friends, Earn up to {X} USDT |
| 暂无活动，敬请期待 | No active event at the moment. Stay tuned! |
| 有效邀请 | Valid Invites |
| 邀请进度 | Invite Progress |
| 阶梯奖励 | Tiered Rewards |
| 已解锁 | Unlocked |
| 未解锁 | Locked |
| 当前目标 | Current Goal |
| 复制邀请链接 | Copy Invite Link |
| 分享 | Share |
| 邀请码 | Invite Code |
| 我的邀请记录 | My Invite Records |
| 我的奖励记录 | My Reward History |
| 进行中 | In Progress |
| 有效 | Valid |
| 无效 | Invalid |
| 已发放 | Distributed |
| 待发放 | Pending |
| 活动规则 | Activity Rules |
| 恭喜！所有奖励已解锁！ | Congratulations! All rewards unlocked! |
| 还需邀请 {N} 位好友解锁 {X} USDT | Only {N} more friends to unlock {X} USDT |
| 活动即将结束，还剩 {N} 天 | Event ends in {N} days. Hurry! |

---

## 12. 非功能需求

| 维度 | 要求 |
|------|------|
| 性能 | 页面首屏加载 < 2s |
| 并发 | 支持 1000 QPS 邀请进度查询 |
| 数据一致性 | 奖励发放保证最终一致性 |
| 安全 | 邀请链接防篡改，接口防刷 |
| 活动切换 | 活动上下线后业务端需在 1 分钟内生效（缓存策略） |
| 兼容性 | Web: Chrome/Safari/Firefox 最近 2 版本；H5: iOS 13+ / Android 8+ |
