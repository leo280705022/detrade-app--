# PRD：DeTrade C2C 邀请奖励活动

## 1. 文档概述

| 属性 | 内容 |
|------|------|
| **文档版本** | V1.0 |
| **项目名称** | C2C 邀请奖励活动 (Invite Friends Rewards) |
| **核心目标** | 通过阶梯式奖励机制激励用户邀请好友注册、充值并交易，提升平台用户增长 |
| **涉及端** | App/Web/H5 (Telegram Mini App) |

---

## 2. 活动规则

### 2.1 活动名称
**Invite Friends, Unlock Bigger Rewards**  
(邀请好友，解锁更多奖励)

### 2.2 邀请条件
好友需完成以下三个步骤才算**有效邀请**：
1. 通过邀请链接注册账号
2. 首次充值金额 ≥ 20 USDT
3. 完成真实有效交易额 ≥ 200 USDT

**说明**：
- 仅计算真实交易，使用体验金/赠金的交易不计入
- 充值和交易需在注册后 30 天内完成
- 同一 IP、同一钱包地址、同一设备视为同一用户，不重复计算

### 2.3 奖励结构

| 有效邀请人数 | 奖励金额 (USDT) | 累计奖励 (USDT) |
|------------|----------------|----------------|
| 1人 | 5 | 5 |
| 3人 | 15 | 20 |
| 5人 | 30 | 50 |
| 10人 | 70 | 120 |
| 20人 | 150 | 270 |
| 50人 | 500 | 770 |

**奖励重置机制**：
- 每领取一次阶梯奖励，计数器重置为 0
- 用户可继续邀请，解锁下一轮奖励
- 示例：邀请 5 人后领取 30U，计数器归零，再邀请 1 人可领取 5U

---

## 3. 功能模块

### 3.1 活动页面

#### 页面入口
- 个人中心 → 邀请奖励
- 首页 Banner 入口
- 邀请链接直接访问

#### 页面结构

**A. 活动 Banner**
- 活动标题：Invite Friends, Unlock Bigger Rewards
- 活动时间：YYYY-MM-DD ~ YYYY-MM-DD（或"永久有效"）
- 背景：金色渐变 + 邀请主题视觉

**B. 数据统计卡片**
| 卡片 | 字段 | 说明 |
|------|------|------|
| 有效邀请 | 当前有效邀请人数 | 已满足条件的邀请数 |
| 待领取奖励 | 可领取的奖励金额 | 已达到但未领取的奖励 |
| 累计已领取 | 历史累计领取金额 | 已到账的奖励总额 |

**C. 进度条区域**
- 显示当前进度：`当前人数 / 下一阶梯人数`
- 进度条可视化：`███████████░░░░░░`
- 提示文案：`Only X more friends to unlock Y USDT Bonus`
- 示例：5/10，提示 "Only 5 more friends to unlock 70 USDT Bonus"

**D. 阶梯奖励展示**
- 6 个阶梯卡片横向排列（可滑动）
- 每个卡片显示：
  - 邀请人数要求
  - 奖励金额
  - 状态标识：
    - 🔒 未解锁（灰色）
    - 🎁 可领取（金色高亮 + 脉冲动画）
    - ✅ 已领取（绿色）

**E. 邀请分享区**
- 专属邀请链接
- 邀请码展示
- 一键复制按钮
- 分享按钮（WhatsApp / Telegram / Twitter / 复制链接）

**F. 邀请记录列表**
| 字段 | 说明 |
|------|------|
| 受邀用户 | 用户名（脱敏显示） |
| 注册时间 | YYYY-MM-DD HH:MM |
| 充值状态 | 未充值 / 已充值 X USDT |
| 交易状态 | 未交易 / 已交易 X USDT |
| 邀请状态 | 待完成 / 有效邀请 |

**G. 活动规则说明**
- 邀请条件详细说明
- 奖励发放规则
- 风控说明

### 3.2 领取奖励流程

#### 领取入口
- 进度条达到阶梯后，对应卡片显示"领取"按钮
- 点击"领取"按钮触发领取流程

#### 领取流程
1. 用户点击"领取"
2. 系统校验：
   - 是否达到阶梯要求
   - 是否已领取过该阶梯
   - 账户状态是否正常
3. 校验通过：
   - 奖励发放至账户余额
   - 弹窗提示："Congratulations! X USDT has been added to your balance"
   - 计数器重置为 0
   - 阶梯状态更新为"已领取"
4. 校验失败：
   - 提示错误原因

### 3.3 邀请链接生成

#### 链接格式
```
https://detreade.com/register?ref={invite_code}
```

#### 邀请码规则
- 6位字母数字组合
- 用户注册时自动生成
- 唯一性校验

#### 邀请关系绑定
- 用户通过链接注册时自动绑定邀请关系
- 手动输入邀请码注册也绑定关系
- 注册后 24 小时内首次充值可追溯绑定

---

## 4. 风控规则

### 4.1 防刷单机制

**IP 校验**
- 同一 IP 下注册的多账号不计入有效邀请
- 同一 IP 24 小时内注册超过 3 个账号触发人工审核

**钱包地址校验**
- 充值钱包地址与邀请人相同不计入有效邀请
- 同一钱包地址为多个账号充值触发风控

**设备指纹**
- 同一设备指纹下的多账号不计入有效邀请
- 设备指纹包括：Device ID、浏览器指纹、MAC 地址

**行为分析**
- 注册后立即充值并交易，交易后立即提现，触发风控
- 邀请人和被邀请人频繁交互（互相转账）触发风控

### 4.2 有效交易判定

**真实交易要求**
- 必须使用自有资金（非体验金/赠金）
- 交易必须有真实市场对手盘
- 自买自卖不计入有效交易

**交易额计算**
- 仅计算开仓金额，平仓不计入
- 取消订单不计入
- 最低单笔交易金额 ≥ 10 USDT

### 4.3 风控拦截

**自动拦截**
- 触发高风险规则时，邀请状态标记为"审核中"
- 奖励暂停发放，等待人工审核

**人工审核**
- 风控后台可查看可疑邀请列表
- 支持标记为"有效"或"无效"
- 无效邀请不计入统计

---

## 5. 数据模型

### 5.1 邀请记录表 (invite_records)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| inviter_uid | bigint | 邀请人 UID |
| invitee_uid | bigint | 受邀人 UID |
| invite_code | varchar(32) | 邀请码 |
| register_time | datetime | 注册时间 |
| first_deposit_time | datetime | 首次充值时间 |
| first_deposit_amount | decimal | 首次充值金额 |
| total_trade_amount | decimal | 累计交易金额 |
| invite_status | tinyint | 邀请状态：0-待完成 1-有效 2-审核中 3-无效 |
| created_at | datetime | 创建时间 |
| updated_at | datetime | 更新时间 |

### 5.2 奖励记录表 (reward_records)

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| uid | bigint | 用户 UID |
| tier | tinyint | 阶梯等级：1-6 |
| invite_count | int | 达成邀请人数 |
| reward_amount | decimal | 奖励金额 |
| status | tinyint | 状态：0-待领取 1-已领取 2-已过期 |
| claim_time | datetime | 领取时间 |
| round | int | 轮次（第几轮奖励） |
| created_at | datetime | 创建时间 |

### 5.3 用户邀请统计表 (user_invite_stats)

| 字段 | 类型 | 说明 |
|------|------|------|
| uid | bigint | 用户 UID |
| current_valid_count | int | 当前有效邀请数（未领取） |
| total_valid_count | int | 历史总有效邀请数 |
| total_claimed_amount | decimal | 累计已领取奖励 |
| current_round | int | 当前轮次 |
| last_claim_time | datetime | 最后领取时间 |
| created_at | datetime | 创建时间 |
| updated_at | datetime | 更新时间 |

---

## 6. API 接口

### 6.1 获取邀请活动信息
```
GET /api/invite/activity
```

**响应**
```json
{
  "code": 0,
  "data": {
    "activity_name": "Invite Friends, Unlock Bigger Rewards",
    "start_time": "2026-01-01 00:00:00",
    "end_time": null,
    "tiers": [
      {"tier": 1, "invite_count": 1, "reward": 5},
      {"tier": 2, "invite_count": 3, "reward": 15},
      {"tier": 3, "invite_count": 5, "reward": 30},
      {"tier": 4, "invite_count": 10, "reward": 70},
      {"tier": 5, "invite_count": 20, "reward": 150},
      {"tier": 6, "invite_count": 50, "reward": 500}
    ],
    "invite_code": "ABC123",
    "invite_link": "https://detreade.com/register?ref=ABC123"
  }
}
```

### 6.2 获取用户邀请统计
```
GET /api/invite/stats
```

**响应**
```json
{
  "code": 0,
  "data": {
    "current_valid_count": 5,
    "pending_reward": 30,
    "total_claimed_amount": 20,
    "current_round": 1,
    "next_tier": {
      "tier": 4,
      "invite_count": 10,
      "reward": 70,
      "remaining": 5
    }
  }
}
```

### 6.3 获取邀请记录列表
```
GET /api/invite/records?page=1&page_size=20
```

**响应**
```json
{
  "code": 0,
  "data": {
    "total": 8,
    "list": [
      {
        "invitee_uid": 10001,
        "invitee_name": "user***123",
        "register_time": "2026-07-15 10:30:00",
        "first_deposit_amount": 50,
        "total_trade_amount": 300,
        "invite_status": 1
      }
    ]
  }
}
```

### 6.4 领取奖励
```
POST /api/invite/claim
```

**请求**
```json
{
  "tier": 3
}
```

**响应**
```json
{
  "code": 0,
  "message": "Claim successful",
  "data": {
    "reward_amount": 30,
    "new_balance": 1050.50,
    "current_valid_count": 0,
    "current_round": 2
  }
}
```

---

## 7. 页面交互说明

### 7.1 进度条动画
- 进度条填充使用平滑过渡动画（0.5s ease-out）
- 达到新阶梯时触发庆祝动画（金色粒子效果）

### 7.2 领取按钮状态
- 未达到阶梯：灰色禁用，显示锁图标
- 达到阶梯：金色高亮，脉冲动画，显示"领取"
- 已领取：绿色，显示"已领取"

### 7.3 分享功能
- 点击分享按钮弹出分享面板
- 支持平台：WhatsApp / Telegram / Twitter / 复制链接
- 分享文案：`Join me on DeTrade! Use my link to get started: {invite_link}`

### 7.4 邀请记录筛选
- 支持按状态筛选：全部 / 待完成 / 有效邀请
- 支持下拉刷新
- 支持上拉加载更多

---

## 8. 多语言支持

### 8.1 中文
- 活动标题：邀请好友，解锁更多奖励
- 邀请条件：好友需完成注册、首次充值≥20U、交易额≥200U
- 进度提示：还需邀请 X 位好友，解锁 Y USDT 奖励
- 领取按钮：领取 / 已领取

### 8.2 英文
- Activity Title: Invite Friends, Unlock Bigger Rewards
- Invite Conditions: Friends must register, first deposit ≥20U, trading volume ≥200U
- Progress Hint: Only X more friends to unlock Y USDT Bonus
- Claim Button: Claim / Claimed

---

## 9. 风控后台（后管端）

### 9.1 功能模块
- 可疑邀请列表：展示触发风控的邀请记录
- 审核操作：标记为有效/无效
- 数据统计：总邀请数、有效邀请数、风控拦截数
- 用户黑名单：添加/移除黑名单用户

### 9.2 风控规则配置
- IP 限制：同 IP 注册数量阈值
- 钱包限制：同钱包充值次数阈值
- 设备限制：同设备注册数量阈值
- 交易金额阈值：最低有效交易金额

---

## 10. 上线计划

### Phase 1 (MVP)
- 基础邀请功能
- 阶梯奖励领取
- 基础风控规则

### Phase 2
- 分享功能优化
- 邀请排行榜
- 风控规则增强

### Phase 3
- 邀请数据分析
- A/B 测试支持
- 动态奖励配置

---

## 11. 附录

### 11.1 奖励计算示例

**场景**：用户 A 邀请好友

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | A 邀请 B 注册 | B 状态：待完成 |
| 2 | B 充值 50U | B 状态：待完成 |
| 3 | B 交易 300U | B 状态：有效邀请 |
| 4 | A 有效邀请数 +1 | 当前：1人 |
| 5 | A 领取第一阶梯奖励 | 获得 5U，计数器归零 |

**多轮奖励示例**：
- 第 1 轮：邀请 5 人 → 领取 30U → 计数器归零
- 第 2 轮：邀请 10 人 → 领取 70U → 计数器归零
- 第 3 轮：邀请 50 人 → 领取 500U → 计数器归零
- 累计获得：600U
