# PRD：TG Bot 菜单后台配置与实时同步方案

## 文档信息
- **版本**: v1.0
- **日期**: 2026-07-01
- **状态**: 技术方案
- **目标**: 在现有后台管理系统中实现 TG Bot 菜单的可视化配置，配置后实时同步到 TG Bot 对话页面

---

## 一、需求分析

### 1.1 现状（图 1）

当前 Bot `/start` 返回的是 **ReplyKeyboardMarkup**（回复键盘），按钮显示在输入框下方：

```
─────────────────────────────────┐
│  Open in app                    │
│  Detrade Group    Detrade X     │
│  Deposit          Withdraw      │
│  Refer            Redemption    │
└─────────────────────────────────┘
```

**问题**：
- 按钮样式单一，无图标
- 布局固定，无法灵活调整
- 修改需要改代码、重新部署
- 无法实时生效

### 1.2 目标（图 2）

改造为 **Inline Keyboard**（内联键盘），按钮嵌入消息内部，支持 3 列网格布局：

```
┌──────────────────────────────────────────────────┐
│  🏦 My Wallet  │  🔄 Exchange  │  💳 Buy&Sell   │
│   Exclusive  │  🎯 Group     │  🔧 Tools      │
│  ⚙️ Bot Set    │  💡 New Func  │  🔗 Download   │
──────────────────────────────────────────────────┘
```

**优势**：
- 支持 emoji 图标，视觉更丰富
- 3 列网格布局，信息密度更高
- 按钮可配置跳转 URL / 打开 Mini App / 触发回调
- **后台可视化配置，实时生效，无需改代码**

---

## 二、核心架构

### 2.1 整体数据流

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  后台管理系统  │────▶│   数据库      │◀────│  Bot 服务    │
│  (Admin UI)  │     │  (MySQL/PG)  │     │  (TG Bot)   │
└─────────────┘     └──────────────     └─────────────┘
       │                    │                    │
       │  1. 配置菜单        │                    │
       │  2. 保存配置        │                    │
       │                    │  3. 写入数据库      │
       │                    │                    │  4. 拉取最新配置
       │                    │                    │  5. 渲染 Inline Keyboard
       │                    │                    │  6. 用户 /start 时展示
```

### 2.2 实时同步机制（两种方案）

#### 方案 A：Webhook 推送（推荐）

```
后台保存配置 → 调用 Bot 服务 API → Bot 服务重新加载菜单 → 下次 /start 生效
```

- **延迟**：< 1 秒
- **实现**：后台保存后 HTTP POST 通知 Bot 服务刷新缓存
- **适合**：Bot 服务有独立 API 接口

#### 方案 B：定时拉取

```
Bot 服务每 N 分钟从数据库/缓存拉取最新菜单配置
```

- **延迟**：N 分钟（建议 1-5 分钟）
- **实现**：Bot 服务定时查询数据库，Redis 缓存
- **适合**：Bot 服务与后台共享数据库

#### 方案 C：Redis Pub/Sub（最优）

```
后台保存配置 → 写入数据库 + 发布 Redis 消息 → Bot 服务订阅消息 → 立即刷新缓存
```

- **延迟**：< 100ms
- **实现**：Redis Pub/Sub 消息通知
- **适合**：已有 Redis 基础设施

---

## 三、数据库设计

### 3.1 菜单配置表

```sql
CREATE TABLE tg_bot_menu (
    id            BIGINT PRIMARY KEY AUTO_INCREMENT,
    bot_id        VARCHAR(64) NOT NULL COMMENT 'Bot 标识（支持多 Bot）',
    menu_group    VARCHAR(32) NOT NULL DEFAULT 'start' COMMENT '菜单分组：start/inline/custom',
    label         VARCHAR(128) NOT NULL COMMENT '按钮显示文本',
    label_en      VARCHAR(128) DEFAULT NULL COMMENT '英文标签（多语言）',
    icon          VARCHAR(16) DEFAULT '' COMMENT 'emoji 图标',
    button_type   VARCHAR(16) NOT NULL COMMENT '按钮类型：url/callback/web_app/switch_inline',
    action_value  VARCHAR(512) NOT NULL COMMENT '动作值：URL / callback_data / Mini App URL',
    row_index     INT NOT NULL DEFAULT 0 COMMENT '所在行号（从 0 开始）',
    col_index     INT NOT NULL DEFAULT 0 COMMENT '所在列号（从 0 开始）',
    sort_order    INT NOT NULL DEFAULT 0 COMMENT '排序权重',
    is_enabled    TINYINT NOT NULL DEFAULT 1 COMMENT '是否启用：0=禁用 1=启用',
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at    DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_bot_group (bot_id, menu_group),
    INDEX idx_sort (bot_id, menu_group, sort_order)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='TG Bot 菜单配置表';
```

### 3.2 菜单数据示例

```sql
-- 图 2 目标菜单的配置数据
INSERT INTO tg_bot_menu (bot_id, menu_group, label, label_en, icon, button_type, action_value, row_index, col_index, sort_order, is_enabled) VALUES
('detrade_bot', 'start', '我的钱包', 'My Wallet', '🏦', 'web_app', 'https://detrade.com/app/wallet', 0, 0, 1, 1),
('detrade_bot', 'start', '兑换', 'Exchange', '🔄', 'web_app', 'https://detrade.com/app/exchange', 0, 1, 2, 1),
('detrade_bot', 'start', '买币/卖币', 'Buy&Sell', '💳', 'web_app', 'https://detrade.com/app/buysell', 0, 2, 3, 1),

('detrade_bot', 'start', '专属 Bot', 'Exclusive Bot', '🤖', 'url', 'https://t.me/detrade_exclusive_bot', 1, 0, 4, 1),
('detrade_bot', 'start', '群组管理', 'Group Management', '🎯', 'url', 'https://t.me/detrade_group', 1, 1, 5, 1),
('detrade_bot', 'start', '工具', 'Tools', '🔧', 'callback', 'show_tools_menu', 1, 2, 6, 1),

('detrade_bot', 'start', 'Bot 设置', 'Bot Settings', '️', 'callback', 'show_settings_menu', 2, 0, 7, 1),
('detrade_bot', 'start', '新功能', 'New Functions', '💡', 'callback', 'show_new_features', 2, 1, 8, 1),
('detrade_bot', 'start', '下载 App', 'Download App', '🔗', 'url', 'https://detrade.com/download', 2, 2, 9, 1);
```

### 3.3 菜单配置快照表（用于版本回滚）

```sql
CREATE TABLE tg_bot_menu_snapshot (
    id            BIGINT PRIMARY KEY AUTO_INCREMENT,
    bot_id        VARCHAR(64) NOT NULL,
    menu_group    VARCHAR(32) NOT NULL,
    config_json   JSON NOT NULL COMMENT '完整菜单配置快照',
    version       INT NOT NULL COMMENT '版本号',
    created_by    VARCHAR(64) COMMENT '操作人',
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_bot_version (bot_id, menu_group, version)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='TG Bot 菜单配置快照表';
```

---

## 四、后台管理系统设计

### 4.1 菜单管理页面 UI

```
┌────────────────────────────────────────────────────────────────┐
│  TG Bot 菜单管理                                    [保存] [预览] │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  Bot 选择：[detrade_bot ▼]    菜单分组：[start ▼]              │
│                                                                │
│  ┌──────────────────────────────────────────────────────────┐ │
│  │ 第 1 行                                                   │ │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐                  │ │
│  │ │🏦 我的钱包│ │🔄 兑换    │ │💳 买币卖币│                  │ │
│  │ │web_app   │ │web_app   │ │web_app   │                  │ │
│  │ │[编辑][删除]│ │[编辑][删除]│ │[编辑][删除]│                  │ │
│  │ └────────── └──────────┘ └──────────                  │ │
│  │                              [+ 添加按钮]                 │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ 第 2 行                                                   │ │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐                  │ │
│  │ │🤖 专属Bot │ │ 群组管理│ │🔧 工具    │                  │ │
│  │ │[编辑][删除]│ │[编辑][删除]│ │[编辑][删除]│                  │ │
│  │ └──────────┘ └──────────┘ └──────────┘                  │ │
│  │                              [+ 添加按钮]                 │ │
│  ├──────────────────────────────────────────────────────────┤ │
│  │ 第 3 行                                                   │ │
│  │ ┌──────────┐ ┌──────────┐ ┌──────────┐                  │ │
│  │ │⚙️ Bot设置 │ │💡 新功能  │ │🔗 下载App │                  │ │
│  │ │[编辑][删除]│ │[编辑][删除]│ │[编辑][删除]│                  │ │
│  │ └──────────┘ └──────────┘ └──────────┘                  │ │
│  │                              [+ 添加按钮]                 │ │
│  ──────────────────────────────────────────────────────────┘ │
│                                                                │
│  [+ 添加新行]                                                  │
│                                                                │
────────────────────────────────────────────────────────────────┤
│  实时预览（右侧/下方）                                          │
│  ┌────────────────────────────────────────┐                   │
│  │  🏦 My Wallet │ 🔄 Exchange │ 💳 Buy&Sell│                   │
│  │  🤖 Exclusive │ 🎯 Group    │ 🔧 Tools  │                   │
│  │  ⚙️ Settings  │ 💡 New Func │ 🔗 Download│                  │
│  └────────────────────────────────────────┘                   │
└────────────────────────────────────────────────────────────────┘
```

### 4.2 按钮编辑弹窗

```
┌─────────────────────────────────────┐
│  编辑按钮                      [X]   │
├─────────────────────────────────────┤
│                                     │
│  图标：[🏦] (emoji 选择器)           │
│                                     │
│  中文标签：[我的钱包__________]      │
│  英文标签：[My Wallet__________]    │
│                                     │
│  按钮类型：[web_app ▼]              │
│    ├── url          → 跳转外部链接   │
│    ├── web_app      → 打开 Mini App │
│    ├── callback     → 触发 Bot 回调 │
│    └── switch_inline→ 切换内联模式  │
│                                     │
│  动作值：[https://detrade.com/app/wallet] │
│                                     │
│  所在行：[1 ▼]    所在列：[1 ▼]     │
│                                     │
│  启用状态：[● 启用 ○ 禁用]           │
│                                     │
│            [取消]    [保存]          │
└─────────────────────────────────────┘
```

### 4.3 按钮类型说明

| 类型 | 说明 | action_value 示例 | TG API 对应 |
|------|------|-------------------|-------------|
| **url** | 点击跳转外部链接 | `https://t.me/detrade_group` | `InlineKeyboardButton(url=...)` |
| **web_app** | 点击打开 Mini App | `https://detrade.com/app/wallet` | `InlineKeyboardButton(web_app=WebAppInfo(url=...))` |
| **callback** | 点击触发 Bot 回调 | `show_tools_menu` | `InlineKeyboardButton(callback_data=...)` |
| **switch_inline** | 切换到内联模式 | `search ` | `InlineKeyboardButton(switch_inline_query=...)` |

---

## 五、API 接口设计

### 5.1 后台管理 API

#### 获取菜单配置

```
GET /api/admin/tg-bot/menu?bot_id=detrade_bot&menu_group=start

Response:
{
  "code": 0,
  "data": {
    "bot_id": "detrade_bot",
    "menu_group": "start",
    "rows": [
      {
        "row_index": 0,
        "buttons": [
          {
            "id": 1,
            "label": "我的钱包",
            "label_en": "My Wallet",
            "icon": "🏦",
            "button_type": "web_app",
            "action_value": "https://detrade.com/app/wallet",
            "col_index": 0,
            "is_enabled": 1
          },
          {
            "id": 2,
            "label": "兑换",
            "label_en": "Exchange",
            "icon": "🔄",
            "button_type": "web_app",
            "action_value": "https://detrade.com/app/exchange",
            "col_index": 1,
            "is_enabled": 1
          },
          {
            "id": 3,
            "label": "买币/卖币",
            "label_en": "Buy&Sell",
            "icon": "💳",
            "button_type": "web_app",
            "action_value": "https://detrade.com/app/buysell",
            "col_index": 2,
            "is_enabled": 1
          }
        ]
      },
      {
        "row_index": 1,
        "buttons": [
          {
            "id": 4,
            "label": "专属 Bot",
            "label_en": "Exclusive Bot",
            "icon": "🤖",
            "button_type": "url",
            "action_value": "https://t.me/detrade_exclusive_bot",
            "col_index": 0,
            "is_enabled": 1
          },
          {
            "id": 5,
            "label": "群组管理",
            "label_en": "Group Management",
            "icon": "🎯",
            "button_type": "url",
            "action_value": "https://t.me/detrade_group",
            "col_index": 1,
            "is_enabled": 1
          },
          {
            "id": 6,
            "label": "工具",
            "label_en": "Tools",
            "icon": "🔧",
            "button_type": "callback",
            "action_value": "show_tools_menu",
            "col_index": 2,
            "is_enabled": 1
          }
        ]
      },
      {
        "row_index": 2,
        "buttons": [
          {
            "id": 7,
            "label": "Bot 设置",
            "label_en": "Bot Settings",
            "icon": "⚙️",
            "button_type": "callback",
            "action_value": "show_settings_menu",
            "col_index": 0,
            "is_enabled": 1
          },
          {
            "id": 8,
            "label": "新功能",
            "label_en": "New Functions",
            "icon": "💡",
            "button_type": "callback",
            "action_value": "show_new_features",
            "col_index": 1,
            "is_enabled": 1
          },
          {
            "id": 9,
            "label": "下载 App",
            "label_en": "Download App",
            "icon": "🔗",
            "button_type": "url",
            "action_value": "https://detrade.com/download",
            "col_index": 2,
            "is_enabled": 1
          }
        ]
      }
    ]
  }
}
```

#### 保存菜单配置

```
POST /api/admin/tg-bot/menu

Request Body:
{
  "bot_id": "detrade_bot",
  "menu_group": "start",
  "rows": [
    {
      "row_index": 0,
      "buttons": [
        {
          "id": 1,
          "label": "我的钱包",
          "label_en": "My Wallet",
          "icon": "🏦",
          "button_type": "web_app",
          "action_value": "https://detrade.com/app/wallet",
          "col_index": 0,
          "is_enabled": 1
        },
        ...
      ]
    },
    ...
  ]
}

Response:
{
  "code": 0,
  "message": "保存成功，已同步到 TG Bot",
  "data": {
    "sync_status": "success",
    "sync_time": "2026-07-01 12:00:00"
  }
}
```

#### 单个按钮 CRUD

```
POST   /api/admin/tg-bot/menu/button          # 新增按钮
PUT    /api/admin/tg-bot/menu/button/:id      # 编辑按钮
DELETE /api/admin/tg-bot/menu/button/:id      # 删除按钮
PATCH  /api/admin/tg-bot/menu/button/:id/toggle  # 启用/禁用
```

#### 同步到 TG Bot

```
POST /api/admin/tg-bot/menu/sync

Request Body:
{
  "bot_id": "detrade_bot",
  "menu_group": "start"
}

Response:
{
  "code": 0,
  "message": "同步成功",
  "data": {
    "sync_time": "2026-07-01 12:00:00",
    "button_count": 9
  }
}
```

### 5.2 Bot 服务 API（内部）

#### 获取最新菜单（Bot 服务调用）

```
GET /api/internal/tg-bot/menu?bot_id=detrade_bot&menu_group=start&lang=zh

Response: 同后台管理 API 格式
```

#### 刷新菜单缓存（后台保存后调用）

```
POST /api/internal/tg-bot/menu/refresh

Request Body:
{
  "bot_id": "detrade_bot",
  "menu_group": "start"
}

Response:
{
  "code": 0,
  "message": "缓存已刷新"
}
```

---

## 六、Bot 服务实现

### 6.1 菜单渲染逻辑（Node.js + Telegraf 示例）

```javascript
const { Telegraf, Markup } = require('telegraf');
const axios = require('axios');

const bot = new Telegraf(process.env.BOT_TOKEN);

// 菜单缓存
let menuCache = {};
const CACHE_TTL = 60000; // 1 分钟缓存
let lastFetchTime = {};

/**
 * 从后台 API 获取最新菜单配置
 */
async function fetchMenu(botId, menuGroup, lang = 'zh') {
  const cacheKey = `${botId}:${menuGroup}:${lang}`;
  const now = Date.now();
  
  // 检查缓存是否有效
  if (menuCache[cacheKey] && (now - lastFetchTime[cacheKey]) < CACHE_TTL) {
    return menuCache[cacheKey];
  }
  
  try {
    const res = await axios.get(
      `${process.env.ADMIN_API_URL}/api/internal/tg-bot/menu`,
      { params: { bot_id: botId, menu_group: menuGroup, lang } }
    );
    
    if (res.data.code === 0) {
      menuCache[cacheKey] = res.data.data;
      lastFetchTime[cacheKey] = now;
      return res.data.data;
    }
  } catch (err) {
    console.error('Failed to fetch menu:', err.message);
  }
  
  // 降级：使用缓存或默认菜单
  return menuCache[cacheKey] || getDefaultMenu();
}

/**
 * 将数据库配置转换为 Telegram Inline Keyboard
 */
function buildInlineKeyboard(menuData) {
  const keyboard = menuData.rows
    .filter(row => row.buttons.some(btn => btn.is_enabled))
    .map(row => {
      return row.buttons
        .filter(btn => btn.is_enabled)
        .map(btn => {
          const label = btn.icon ? `${btn.icon} ${btn.label}` : btn.label;
          
          switch (btn.button_type) {
            case 'url':
              return Markup.button.url(label, btn.action_value);
            
            case 'web_app':
              return Markup.button.webApp(label, { url: btn.action_value });
            
            case 'callback':
              return Markup.button.callback(label, btn.action_value);
            
            case 'switch_inline':
              return Markup.button.switchToChat(label, btn.action_value);
            
            default:
              return Markup.button.callback(label, 'unknown');
          }
        });
    });
  
  return Markup.inlineKeyboard(keyboard);
}

/**
 * /start 命令处理
 */
bot.start(async (ctx) => {
  const menuData = await fetchMenu('detrade_bot', 'start');
  const keyboard = buildInlineKeyboard(menuData);
  
  await ctx.reply(
    'Welcome to Detrade — the 0 gas gateway to Web3 trading',
    {
      ...keyboard,
      reply_markup: {
        ...keyboard.reply_markup,
        resize_keyboard: true,
      }
    }
  );
});

/**
 * 回调按钮处理
 */
bot.action('show_tools_menu', async (ctx) => {
  // 从数据库获取工具子菜单
  const menuData = await fetchMenu('detrade_bot', 'tools');
  const keyboard = buildInlineKeyboard(menuData);
  
  await ctx.editMessageText('🔧 工具菜单', keyboard);
});

bot.action('show_settings_menu', async (ctx) => {
  const menuData = await fetchMenu('detrade_bot', 'settings');
  const keyboard = buildInlineKeyboard(menuData);
  
  await ctx.editMessageText('⚙️ Bot 设置', keyboard);
});

bot.action('show_new_features', async (ctx) => {
  await ctx.editMessageText('💡 新功能即将上线，敬请期待！');
});

/**
 * 接收后台刷新通知（Webhook 方式）
 */
bot.telegram.setWebhook(process.env.WEBHOOK_URL);

// 内部 API 路由：接收后台刷新通知
app.post('/api/internal/tg-bot/menu/refresh', (req, res) => {
  const { bot_id, menu_group } = req.body;
  const cacheKey = `${bot_id}:${menu_group}`;
  
  // 清除缓存，下次请求时重新拉取
  delete menuCache[cacheKey];
  delete lastFetchTime[cacheKey];
  
  // 清除所有语言的缓存
  Object.keys(menuCache).forEach(key => {
    if (key.startsWith(`${bot_id}:${menu_group}:`)) {
      delete menuCache[key];
      delete lastFetchTime[key];
    }
  });
  
  res.json({ code: 0, message: '缓存已刷新' });
});

bot.launch();
```

### 6.2 默认菜单降级

```javascript
function getDefaultMenu() {
  return {
    rows: [
      {
        row_index: 0,
        buttons: [
          { id: 0, label: 'Open in app', label_en: 'Open in app', icon: '', button_type: 'web_app', action_value: 'https://detrade.com', col_index: 0, is_enabled: 1 },
        ]
      },
      {
        row_index: 1,
        buttons: [
          { id: 0, label: 'Detrade Group', label_en: 'Detrade Group', icon: '', button_type: 'url', action_value: 'https://t.me/detrade_group', col_index: 0, is_enabled: 1 },
          { id: 0, label: 'Detrade X', label_en: 'Detrade X (Twitter)', icon: '', button_type: 'url', action_value: 'https://twitter.com/detrade', col_index: 1, is_enabled: 1 },
        ]
      },
      {
        row_index: 2,
        buttons: [
          { id: 0, label: 'Deposit', label_en: 'Deposit', icon: '', button_type: 'web_app', action_value: 'https://detrade.com/app/deposit', col_index: 0, is_enabled: 1 },
          { id: 0, label: 'Withdraw', label_en: 'Withdraw', icon: '', button_type: 'web_app', action_value: 'https://detrade.com/app/withdraw', col_index: 1, is_enabled: 1 },
        ]
      },
      {
        row_index: 3,
        buttons: [
          { id: 0, label: 'Refer', label_en: 'Refer', icon: '', button_type: 'web_app', action_value: 'https://detrade.com/app/refer', col_index: 0, is_enabled: 1 },
          { id: 0, label: 'Redemption', label_en: 'Redemption', icon: '', button_type: 'web_app', action_value: 'https://detrade.com/app/redemption', col_index: 1, is_enabled: 1 },
        ]
      }
    ]
  };
}
```

---

## 七、实时同步实现方案

### 7.1 方案对比

| 方案 | 延迟 | 复杂度 | 可靠性 | 推荐场景 |
|------|------|--------|--------|---------|
| **Webhook 推送** | < 1s | 低 | 高 | Bot 服务有独立 API（推荐） |
| **定时拉取** | 1-5 min | 最低 | 最高 | 共享数据库，无 API |
| **Redis Pub/Sub** | < 100ms | 中 | 高 | 已有 Redis，追求极致 |

### 7.2 推荐方案：Webhook 推送 + 缓存降级

```
后台保存菜单
  │
  ├── 1. 写入数据库（MySQL/PG）
  ├── 2. 写入快照表（版本回滚）
  ├── 3. HTTP POST 通知 Bot 服务刷新缓存
  │     └── 成功 → 返回 "已同步"
  │     └── 失败 → 记录日志，Bot 服务下次请求时自动拉取
  └── 4. 返回前端 "保存成功"
```

**关键设计**：
- Bot 服务有 1 分钟缓存，减少 API 调用
- 后台保存后立即通知 Bot 服务清除缓存
- 即使通知失败，Bot 服务 1 分钟后也会自动拉取最新配置
- 双重保障：推送 + 定时兜底

### 7.3 后台保存逻辑（伪代码）

```javascript
// 后台管理系统 - 保存菜单
async function saveMenu(req, res) {
  const { bot_id, menu_group, rows } = req.body;
  
  // 1. 开启事务
  const transaction = await db.beginTransaction();
  
  try {
    // 2. 保存快照（版本回滚）
    const currentConfig = await db.query(
      'SELECT * FROM tg_bot_menu WHERE bot_id = ? AND menu_group = ?',
      [bot_id, menu_group]
    );
    await db.query(
      'INSERT INTO tg_bot_menu_snapshot (bot_id, menu_group, config_json, version, created_by) VALUES (?, ?, ?, ?, ?)',
      [bot_id, menu_group, JSON.stringify(currentConfig), getNextVersion(bot_id, menu_group), req.user.name]
    );
    
    // 3. 删除旧配置
    await db.query(
      'DELETE FROM tg_bot_menu WHERE bot_id = ? AND menu_group = ?',
      [bot_id, menu_group]
    );
    
    // 4. 插入新配置
    for (const row of rows) {
      for (const btn of row.buttons) {
        await db.query(
          `INSERT INTO tg_bot_menu (bot_id, menu_group, label, label_en, icon, button_type, action_value, row_index, col_index, sort_order, is_enabled) 
           VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
          [bot_id, menu_group, btn.label, btn.label_en, btn.icon, btn.button_type, btn.action_value, row.row_index, btn.col_index, btn.id || 0, btn.is_enabled]
        );
      }
    }
    
    await transaction.commit();
    
    // 5. 通知 Bot 服务刷新缓存
    try {
      await axios.post(
        `${process.env.BOT_SERVICE_URL}/api/internal/tg-bot/menu/refresh`,
        { bot_id, menu_group }
      );
    } catch (err) {
      console.warn('Bot 服务通知失败，将在下次请求时自动刷新:', err.message);
      // 不阻塞保存流程
    }
    
    res.json({ code: 0, message: '保存成功，已同步到 TG Bot' });
    
  } catch (err) {
    await transaction.rollback();
    res.json({ code: -1, message: '保存失败' });
  }
}
```

---

## 八、后台管理前端实现

### 8.1 技术选型

| 层面 | 推荐方案 | 说明 |
|------|---------|------|
| **前端框架** | React / Vue（与现有后台一致） | 复用现有组件库 |
| **拖拽排序** | `@dnd-kit/core` 或 `vuedraggable` | 按钮行列拖拽调整 |
| **Emoji 选择器** | `emoji-picker-react` 或 `emoji-mart` | 图标选择 |
| **实时预览** | 组件化渲染 | 左侧编辑，右侧预览 |

### 8.2 核心组件结构

```
TgBotMenuManager/
├── MenuManager.jsx          # 主页面
├── MenuRow.jsx              # 行组件（可拖拽）
├── MenuButton.jsx           # 按钮组件（可拖拽、编辑、删除）
├── ButtonEditor.jsx         # 按钮编辑弹窗
├── EmojiPicker.jsx          # Emoji 选择器
├── MenuPreview.jsx          # 实时预览组件
└── api.js                   # API 调用
```

### 8.3 拖拽排序实现

```javascript
// 使用 @dnd-kit/core 实现行列拖拽
import { DndContext, closestCenter } from '@dnd-kit/core';
import { SortableContext, arrayMove, useSortable } from '@dnd-kit/sortable';

function MenuManager() {
  const [rows, setRows] = useState([]);
  
  // 行拖拽
  const handleRowDragEnd = (event) => {
    const { active, over } = event;
    if (active.id !== over.id) {
      setRows(rows => {
        const oldIndex = rows.findIndex(r => r.row_index === active.id);
        const newIndex = rows.findIndex(r => r.row_index === over.id);
        const newRows = arrayMove(rows, oldIndex, newIndex);
        // 重新计算 row_index
        return newRows.map((row, idx) => ({ ...row, row_index: idx }));
      });
    }
  };
  
  // 按钮拖拽（跨行）
  const handleButtonDragEnd = (event) => {
    const { active, over } = event;
    // 处理按钮在行内和跨行移动
  };
  
  return (
    <DndContext collisionDetection={closestCenter} onDragEnd={handleRowDragEnd}>
      <SortableContext items={rows.map(r => r.row_index)}>
        {rows.map(row => (
          <MenuRow key={row.row_index} row={row} onButtonDragEnd={handleButtonDragEnd} />
        ))}
      </SortableContext>
    </DndContext>
  );
}
```

---

## 九、多语言支持

### 9.1 方案

菜单配置表中已有 `label`（中文）和 `label_en`（英文）字段。

Bot 服务根据用户语言返回对应标签：

```javascript
bot.start(async (ctx) => {
  const userLang = ctx.from.language_code || 'zh';
  const lang = userLang.startsWith('zh') ? 'zh' : 'en';
  
  const menuData = await fetchMenu('detrade_bot', 'start', lang);
  const keyboard = buildInlineKeyboard(menuData, lang);
  
  const welcomeText = lang === 'zh' 
    ? '欢迎来到 Detrade — 0 Gas 的 Web3 交易入口'
    : 'Welcome to Detrade — the 0 gas gateway to Web3 trading';
  
  await ctx.reply(welcomeText, keyboard);
});
```

### 9.2 后台多语言编辑

后台编辑按钮时，同时填写中文和英文标签：

```
中文标签：[我的钱包__________]
英文标签：[My Wallet__________]
```

预览时支持切换语言查看效果。

---

## 十、安全与权限

### 10.1 后台权限控制

| 权限 | 说明 |
|------|------|
| **查看菜单** | 运营人员可查看 |
| **编辑菜单** | 运营主管可编辑 |
| **发布菜单** | 需二次确认，记录操作日志 |
| **回滚菜单** | 可从快照表恢复到历史版本 |

### 10.2 Bot 服务安全

```javascript
// 内部 API 需要鉴权
const internalAuth = (req, res, next) => {
  const token = req.headers['x-internal-token'];
  if (token !== process.env.INTERNAL_API_TOKEN) {
    return res.status(403).json({ code: -1, message: 'Forbidden' });
  }
  next();
};

app.post('/api/internal/tg-bot/menu/refresh', internalAuth, (req, res) => {
  // ...
});
```

### 10.3 操作审计

```sql
CREATE TABLE tg_bot_menu_audit_log (
    id            BIGINT PRIMARY KEY AUTO_INCREMENT,
    bot_id        VARCHAR(64) NOT NULL,
    menu_group    VARCHAR(32) NOT NULL,
    action        VARCHAR(16) NOT NULL COMMENT 'create/update/delete/rollback',
    operator      VARCHAR(64) NOT NULL COMMENT '操作人',
    old_config    JSON COMMENT '变更前配置',
    new_config    JSON COMMENT '变更后配置',
    created_at    DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='TG Bot 菜单操作审计日志';
```

---

## 十一、实施步骤

### Phase 1：数据库 + API（1-2 天）

- [ ] 创建 `tg_bot_menu` 表
- [ ] 创建 `tg_bot_menu_snapshot` 表
- [ ] 创建 `tg_bot_menu_audit_log` 表
- [ ] 实现菜单 CRUD API
- [ ] 实现菜单同步 API
- [ ] 导入现有菜单数据（从图 1 的配置迁移）

### Phase 2：后台管理前端（2-3 天）

- [ ] 菜单管理页面（列表 + 编辑）
- [ ] 按钮编辑弹窗
- [ ] Emoji 选择器
- [ ] 实时预览组件
- [ ] 拖拽排序
- [ ] 多语言编辑

### Phase 3：Bot 服务改造（1-2 天）

- [ ] 从数据库/ API 拉取菜单配置
- [ ] Inline Keyboard 渲染逻辑
- [ ] 缓存机制（1 分钟 TTL）
- [ ] 刷新缓存接口
- [ ] 回调按钮处理
- [ ] 降级默认菜单

### Phase 4：联调测试（1 天）

- [ ] 后台保存 → Bot 实时生效
- [ ] 多语言切换
- [ ] 按钮类型全覆盖测试
- [ ] 异常降级测试
- [ ] 操作审计日志验证

### 总工期：5-8 天

---

## 十二、现有菜单迁移方案

### 12.1 图 1 现有菜单 → 数据库

```sql
-- 将现有 ReplyKeyboard 菜单迁移为 Inline Keyboard 配置
INSERT INTO tg_bot_menu (bot_id, menu_group, label, label_en, icon, button_type, action_value, row_index, col_index, sort_order, is_enabled) VALUES
('detrade_bot', 'start', '打开应用', 'Open in app', '📱', 'web_app', 'https://detrade.com', 0, 0, 1, 1),

('detrade_bot', 'start', 'Detrade 群组', 'Detrade Group', '💬', 'url', 'https://t.me/detrade_group', 1, 0, 2, 1),
('detrade_bot', 'start', 'Detrade X', 'Detrade X (Twitter)', '🐦', 'url', 'https://twitter.com/detrade', 1, 1, 3, 1),

('detrade_bot', 'start', '充值', 'Deposit', '', 'web_app', 'https://detrade.com/app/deposit', 2, 0, 4, 1),
('detrade_bot', 'start', '提现', 'Withdraw', '', 'web_app', 'https://detrade.com/app/withdraw', 2, 1, 5, 1),

('detrade_bot', 'start', '推荐', 'Refer', '🎁', 'web_app', 'https://detrade.com/app/refer', 3, 0, 6, 1),
('detrade_bot', 'start', '兑换', 'Redemption', '🔄', 'web_app', 'https://detrade.com/app/redemption', 3, 1, 7, 1);
```

### 12.2 扩展为目标菜单（图 2）

在现有 7 个按钮基础上，后台新增：
- 🏦 我的钱包（web_app → /app/wallet）
- 🔄 兑换（web_app → /app/exchange）
- 💳 买币/卖币（web_app → /app/buysell）
-  专属 Bot（url → 另一个 Bot）
- 🎯 群组管理（url → 群组链接）
- 🔧 工具（callback → 子菜单）
- ️ Bot 设置（callback → 子菜单）
- 💡 新功能（callback → 提示消息）
- 🔗 下载 App（url → 下载页）

---

## 十三、子菜单设计

### 13.1 工具子菜单（callback: show_tools_menu）

```
┌────────────────────────────────────────┐
│  🔧 工具                                │
│                                        │
│  📊 交易信号  │  🤖 AI 分析  │  📈 排行榜│
│  🔔 价格提醒  │  📋 持仓管理  │  📖 教程 │
│  [← 返回主菜单]                        │
└────────────────────────────────────────
```

### 13.2 设置子菜单（callback: show_settings_menu）

```
┌────────────────────────────────────────┐
│  ⚙️ Bot 设置                            │
│                                        │
│  🌐 语言设置   │  🔔 通知设置           │
│  🎨 主题切换   │  🔒 安全设置           │
│  [← 返回主菜单]                        │
└────────────────────────────────────────
```

### 13.3 子菜单数据配置

```sql
-- 工具子菜单
INSERT INTO tg_bot_menu (bot_id, menu_group, label, label_en, icon, button_type, action_value, row_index, col_index, sort_order, is_enabled) VALUES
('detrade_bot', 'tools', '交易信号', 'Trading Signals', '📊', 'web_app', 'https://detrade.com/app/signals', 0, 0, 1, 1),
('detrade_bot', 'tools', 'AI 分析', 'AI Analysis', '🤖', 'web_app', 'https://detrade.com/app/ai', 0, 1, 2, 1),
('detrade_bot', 'tools', '排行榜', 'Leaderboard', '📈', 'web_app', 'https://detrade.com/app/leaderboard', 0, 2, 3, 1),
('detrade_bot', 'tools', '价格提醒', 'Price Alerts', '🔔', 'callback', 'set_price_alert', 1, 0, 4, 1),
('detrade_bot', 'tools', '持仓管理', 'Positions', '📋', 'web_app', 'https://detrade.com/app/positions', 1, 1, 5, 1),
('detrade_bot', 'tools', '教程', 'Tutorial', '📖', 'url', 'https://detrade.com/docs', 1, 2, 6, 1),
('detrade_bot', 'tools', '← 返回', '← Back', '', 'callback', 'show_start_menu', 2, 0, 7, 1);

-- 设置子菜单
INSERT INTO tg_bot_menu (bot_id, menu_group, label, label_en, icon, button_type, action_value, row_index, col_index, sort_order, is_enabled) VALUES
('detrade_bot', 'settings', '语言设置', 'Language', '🌐', 'callback', 'set_language', 0, 0, 1, 1),
('detrade_bot', 'settings', '通知设置', 'Notifications', '🔔', 'callback', 'set_notifications', 0, 1, 2, 1),
('detrade_bot', 'settings', '主题切换', 'Theme', '🎨', 'callback', 'set_theme', 1, 0, 3, 1),
('detrade_bot', 'settings', '安全设置', 'Security', '🔒', 'callback', 'set_security', 1, 1, 4, 1),
('detrade_bot', 'settings', '← 返回', '← Back', '', 'callback', 'show_start_menu', 2, 0, 5, 1);
```

---

## 十四、核心业务菜单规划

基于 Detrade 两大核心业务模块，建议菜单结构：

### 14.1 完整菜单架构

```
/start 主菜单
├── 第 1 行： 我的钱包 | 🔄 兑换 | 💳 买币/卖币
├── 第 2 行：📈 合约交易 | 🎯 预测交易 | 📊 交易信号
├── 第 3 行：🎁 推荐返佣 | 💰 充值 | 💸 提现
└── 第 4 行：⚙️ 设置 | 💬 群组 | 🔗 下载 App

合约交易子菜单（futures/updown/highlow/spread/taptrading）
├── 第 1 行：⚡ 合约 Futures | 📈 涨跌 UpDown | 📉 高低价 HighLow
├── 第 2 行： 差价 Spread | 👆 点击交易 TapTrading
└── 第 3 行：← 返回

预测交易子菜单（Polymarket 事件）
├── 第 1 行：⚽ 足球预测 | 🏀 篮球预测 |  橄榄球预测
├── 第 2 行：🏆 世界杯专题 | 📊 热门赛事 |  最新上线
└── 第 3 行：← 返回
```

### 14.2 菜单配置数据

```sql
-- 主菜单（按 Detrade 业务重新规划）
INSERT INTO tg_bot_menu (bot_id, menu_group, label, label_en, icon, button_type, action_value, row_index, col_index, sort_order, is_enabled) VALUES
-- 第 1 行：资产
('detrade_bot', 'start', '我的钱包', 'My Wallet', '🏦', 'web_app', 'https://detrade.com/app/wallet', 0, 0, 1, 1),
('detrade_bot', 'start', '兑换', 'Exchange', '🔄', 'web_app', 'https://detrade.com/app/exchange', 0, 1, 2, 1),
('detrade_bot', 'start', '买币/卖币', 'Buy & Sell', '', 'web_app', 'https://detrade.com/app/buysell', 0, 2, 3, 1),

-- 第 2 行：核心交易
('detrade_bot', 'start', '合约交易', 'Futures Trading', '📈', 'callback', 'show_futures_menu', 1, 0, 4, 1),
('detrade_bot', 'start', '预测交易', 'Prediction Trading', '🎯', 'callback', 'show_prediction_menu', 1, 1, 5, 1),
('detrade_bot', 'start', '交易信号', 'Trading Signals', '📊', 'web_app', 'https://detrade.com/app/signals', 1, 2, 6, 1),

-- 第 3 行：资产操作
('detrade_bot', 'start', '推荐返佣', 'Refer & Earn', '🎁', 'web_app', 'https://detrade.com/app/refer', 2, 0, 7, 1),
('detrade_bot', 'start', '充值', 'Deposit', '💰', 'web_app', 'https://detrade.com/app/deposit', 2, 1, 8, 1),
('detrade_bot', 'start', '提现', 'Withdraw', '💸', 'web_app', 'https://detrade.com/app/withdraw', 2, 2, 9, 1),

-- 第 4 行：其他
('detrade_bot', 'start', '设置', 'Settings', '⚙️', 'callback', 'show_settings_menu', 3, 0, 10, 1),
('detrade_bot', 'start', '社群', 'Community', '', 'url', 'https://t.me/detrade_group', 3, 1, 11, 1),
('detrade_bot', 'start', '下载 App', 'Download App', '', 'url', 'https://detrade.com/download', 3, 2, 12, 1);

-- 合约交易子菜单
INSERT INTO tg_bot_menu (bot_id, menu_group, label, label_en, icon, button_type, action_value, row_index, col_index, sort_order, is_enabled) VALUES
('detrade_bot', 'futures', '合约', 'Futures', '⚡', 'web_app', 'https://detrade.com/app/trade/futures', 0, 0, 1, 1),
('detrade_bot', 'futures', '涨跌', 'UpDown', '📈', 'web_app', 'https://detrade.com/app/trade/updown', 0, 1, 2, 1),
('detrade_bot', 'futures', '高低价', 'HighLow', '📉', 'web_app', 'https://detrade.com/app/trade/highlow', 0, 2, 3, 1),
('detrade_bot', 'futures', '差价', 'Spread', '📊', 'web_app', 'https://detrade.com/app/trade/spread', 1, 0, 4, 1),
('detrade_bot', 'futures', '点击交易', 'TapTrading', '👆', 'web_app', 'https://detrade.com/app/trade/taptrading', 1, 1, 5, 1),
('detrade_bot', 'futures', '← 返回', '← Back', '', 'callback', 'show_start_menu', 2, 0, 6, 1);

-- 预测交易子菜单
INSERT INTO tg_bot_menu (bot_id, menu_group, label, label_en, icon, button_type, action_value, row_index, col_index, sort_order, is_enabled) VALUES
('detrade_bot', 'prediction', '足球', 'Football', '⚽', 'web_app', 'https://detrade.com/app/predict/football', 0, 0, 1, 1),
('detrade_bot', 'prediction', '篮球', 'Basketball', '', 'web_app', 'https://detrade.com/app/predict/basketball', 0, 1, 2, 1),
('detrade_bot', 'prediction', '橄榄球', 'Football(US)', '🏈', 'web_app', 'https://detrade.com/app/predict/football-us', 0, 2, 3, 1),
('detrade_bot', 'prediction', '世界杯', 'World Cup', '', 'web_app', 'https://detrade.com/app/predict/worldcup', 1, 0, 4, 1),
('detrade_bot', 'prediction', '热门赛事', 'Hot Events', '🔥', 'web_app', 'https://detrade.com/app/predict/hot', 1, 1, 5, 1),
('detrade_bot', 'prediction', '← 返回', '← Back', '', 'callback', 'show_start_menu', 2, 0, 6, 1);
```

---

## 十五、总结

### 核心实现路径

```
1. 数据库建表（tg_bot_menu + 快照表 + 审计表）
2. 后台 API（CRUD + 同步通知）
3. 后台前端（可视化编辑 + 拖拽 + 预览）
4. Bot 服务改造（从 API 拉取菜单 + Inline Keyboard 渲染 + 缓存）
5. 联调测试（保存 → 实时生效验证）
```

### 关键技术点

| 技术点 | 方案 |
|--------|------|
| **菜单存储** | MySQL 表，支持行列布局 |
| **实时同步** | Webhook 推送 + 1 分钟缓存兜底 |
| **按钮渲染** | Inline Keyboard（非 Reply Keyboard） |
| **多语言** | label + label_en 双字段 |
| **版本回滚** | 快照表 + 一键恢复 |
| **操作审计** | 审计日志表记录所有变更 |

### 预期效果

- 运营人员在后台拖拽调整菜单布局，点击保存
- **1 秒内** TG Bot 对话页面菜单自动更新
- 无需改代码、无需重新部署
- 支持多语言、多 Bot、版本回滚

---

**文档结束**
