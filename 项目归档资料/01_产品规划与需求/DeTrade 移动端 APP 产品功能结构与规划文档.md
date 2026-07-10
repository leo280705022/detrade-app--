# DeTrade 移动端 APP 产品功能结构与规划文档

## 1. 文档概述
**项目名称**: DeTrade Mobile App  
**文档版本**: v1.0  
**更新日期**: 2026-01-05  
**文档状态**: 初稿  
**目标**: 基于 DeTrade 交易所 H5 现有业务逻辑，规划移动端 APP (iOS/Android) 的功能架构与版本迭代路线。

---

## 2. 产品功能结构图 (Information Architecture)

APP 采用经典的底部导航栏 (Bottom Tab Bar) 结构，分为 5 个核心模块：**首页 (Home)**、**行情 (Markets)**、**交易 (Trade)**、**资产 (Wallet)**、**我的 (Profile)**。

### 2.1 首页 (Home) - 流量分发与营销
*   **顶部导航栏**
    *   **搜索 (Search)**: 支持搜索币种代码 (BTC, ETH)、交易对。
    *   **客服入口 (Support)**: 跳转在线客服系统 (Zendesk/Intercom)。
    *   **消息中心 (Notification)**: 系统公告、充提通知、爆仓/强平预警。
*   **核心运营区**
    *   **Banner 轮播**: 运营活动、新币上线、主要市场新闻。
    *   **跑马灯公告**: 滚动展示简短的紧急或重要通知。
*   **金刚区 (Quick Actions)**
    *   **充值 (Deposit)**: 快捷跳转充值页面。
    *   **邀请返佣 (Referral)**: 跳转合伙人推广页面。
    *   **帮助中心 (Help)**: 新手教程、常见问题 (FAQ)。
    *   **福利中心 (Rewards)**: 签到、任务中心（如有）。
*   **行情看板 (Market Overview)**
    *   **涨幅榜 (Top Gainers)**: 24h 涨幅前 10。
    *   **跌幅榜 (Top Losers)**: 24h 跌幅前 10。
    *   **成交榜 (Volume)**: 24h 成交额排名。
    *   **热门 (Hot)**: 平台推荐或搜索热度高的币种。

### 2.2 行情 (Markets) - 数据聚合
*   **顶部筛选**
    *   **自选 (Favorites)**: 用户收藏的交易对，支持编辑排序。
    *   **现货 (Spot)**: 现货交易对列表。
    *   **合约 (Futures)**: 永续合约交易对列表。
    *   **期权 (Options)**: 二元期权标的列表。
*   **板块分类 (Sectors)**
    *   Meme, DeFi, Layer2, Storage, Gaming 等概念板块筛选。
*   **列表项展示**
    *   交易对名称 (e.g., BTC/USDT)。
    *   最新价格 (Last Price)。
    *   24h 涨跌幅 (Change%) - *支持红涨绿跌/绿涨红跌切换*。

### 2.3 交易 (Trade) - 核心业务
此模块为 APP 最核心部分，需支持高频刷新与低延迟交互。
*   **顶部切换**
    *   **U本位合约 (Futures)**
    *   **二元期权 (Options / Rhythm Master)**
*   **K线图表区 (Chart)**
    *   **图表库**: 集成 TradingView (Lightweight Charts)。
    *   **周期切换**: 1m, 5m, 15m, 1h, 4h, 1d, 1w。
    *   **指标工具**: MA, EMA, BOLL, MACD, RSI, KDJ。
    *   **深度图**: 买卖盘口深度可视化。
*   **操作区 (Order Panel)**
    *   **合约模式**:
        *   **仓位模式**: 全仓 (Cross) / 逐仓 (Isolated)。
        *   **杠杆调节**: 滑块或输入框，支持 1x - 1000x。
        *   **订单类型**: 市价单 (Market)、限价单 (Limit)、计划委托 (Trigger)。
        *   **方向**: 开多 (Open Long) / 开空 (Open Short)。
        *   **资产展示**: 可用保证金余额。
    *   **期权模式**:
        *   **到期时间**: 30s, 60s, 120s, 5m。
        *   **收益率**: 动态展示当前赔率 (e.g., 180%)。
        *   **方向**: 买涨 (Call) / 买跌 (Put)。
*   **底部数据区**
    *   **当前持仓 (Positions)**: 标的、方向、杠杆、持仓量、开仓价、标记价、强平价、未结盈亏 (ROE)。
    *   **当前委托 (Open Orders)**: 未成交的限价/计划单，支持撤单。
    *   **历史订单 (Order History)**: 历史成交记录。

### 2.4 资产 (Wallet) - 资金管理
*   **总览卡片**
    *   总资产估值 (折合 USDT/USD)。
    *   今日盈亏 (Today's PNL)。
    *   **账户结构**: 资金账户 (Funding)、交易账户 (Trading)。
*   **核心功能**
    *   **充值 (Deposit)**:
        *   链选择: TRC20, ERC20, BEP20, TON。
        *   二维码/地址复制。
    *   **提现 (Withdraw)**:
        *   提币地址输入/粘贴。
        *   数量输入、手续费计算。
        *   安全验证 (2FA, 邮箱验证码)。
    *   **划转 (Transfer)**: 资金账户 <-> 交易账户。
    *   **闪兑 (Convert)**: 小额资产兑换或法币出入金(如有)。
*   **历史记录 (History)**
    *   充提记录、资金流水、划转记录。

### 2.5 我的 (Profile) - 账户体系
*   **个人信息头部**
    *   头像、昵称、UID。
    *   VIP 等级 / 认证状态 (KYC)。
*   **安全中心 (Security)**
    *   修改登录密码。
    *   绑定/修改手机号、邮箱。
    *   谷歌验证器 (Google Authenticator)。
    *   资金密码。
    *   指纹/面容登录 (Biometrics)。
*   **合伙人 (Affiliate)**
    *   我的邀请码/海报。
    *   返佣数据统计。
    *   下级列表。
*   **通用设置 (Settings)**
    *   多语言 (Language): English, 中文, etc.
    *   计价货币 (Currency): USD, CNY, KRW, JPY.
    *   涨跌色 (Color Preference)。
    *   清除缓存。
    *   版本信息 (Version)。

---

## 3. 版本迭代规划 (Roadmap)

### Phase 1: MVP 核心交易版 (Core Trading)
**目标**: 完成最小闭环，支持基础合约交易，确保资金进出通畅。
*   **周期**: 4-6 周
*   **功能**:
    *   基础注册/登录 (邮箱/手机)。
    *   行情列表与基础 K 线 (TradingView 基础版)。
    *   U本位合约交易 (市价/限价，1-100x)。
    *   资产充值 (USDT-TRC20) 与 提现。
    *   个人中心基础设置。

### Phase 2: 完整运营版 (Full Operations)
**目标**: 对标 H5 全功能，引入高杠杆与二元期权，开启市场推广。
*   **周期**: +4 周
*   **功能**:
    *   **二元期权 (Binary Options)** 模块上线。
    *   合约杠杆上限提升至 **1000x**。
    *   **邀请返佣系统** (裂变核心)。
    *   增加 **TON 链** 充值支持。
    *   应用内推送通知 (Push Notification)。

### Phase 3: 生态增强版 (Ecosystem Growth)
**目标**: 提升留存与用户体验，构建社区。
*   **周期**: 持续迭代
*   **功能**:
    *   **Web3 钱包登录** (Connect Wallet)。
    *   **跟单交易 (Copy Trading)**。
    *   **模拟盘 (Demo Account)**。
    *   社区聊天室/讨论区。
    *   生物识别登录 (FaceID/TouchID)。

---

## 4. 技术栈建议 (Tech Stack)

*   **开发框架**: **React Native** (推荐)
    *   *理由*: 能够复用 H5 (React) 的大量业务逻辑代码 (Hooks, Utils, State Management)，开发效率最高，且支持热更新 (Code Push)。
*   **UI 组件库**: Tamagui 或 NativeBase (适配移动端交互)。
*   **图表库**: `react-native-webview` + TradingView Lightweight Charts (保证图表性能与交互体验)。
*   **状态管理**: Zustand / Redux Toolkit (与 H5 保持一致)。
*   **网络请求**: Axios / TanStack Query。
*   **WebSocket**: 原生 WebSocket 或 ReconnectingWebSocket (处理行情推送)。

