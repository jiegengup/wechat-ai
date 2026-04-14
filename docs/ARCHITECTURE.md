# 微信 Bot + 外卖 CPS 技术架构

> 更新: 2026-04-06

## 定位

类 BotFox.ai 的微信 Bot SaaS 平台，核心商业逻辑：AI 聊天为钩子吸引用户 → CPS 返利为盈利引擎。

## 技术栈

| 层 | 技术 |
|---|------|
| 语言 | Node.js (ES Module) |
| 框架 | Express 4.21 |
| 数据库 | SQLite (better-sqlite3 v11.7) |
| 微信接口 | iLink Bot HTTP API（官方接口，非灰色协议） |
| AI 对话 | OpenAI 兼容 /chat/completions（当前未配置） |
| CPS | 万单联盟（外卖/本地生活）+ 订单侠（电商聚合转链） |
| 前端 | 纯 HTML + Tailwind CSS（零构建，CDN 被墙用本地文件） |
| 日志 | Pino + pino-pretty |
| 加密 | AES-128-ECB（iLink 媒体）+ JWT（认证）+ bcryptjs（密码） |
| 部署 | PM2 + Nginx + Let's Encrypt SSL |

## 整体架构图

```
用户微信（公众号/个人号 Bot）
    │
    │ iLink Bot API（微信官方）
    │ 长轮询 Long Polling（35s 超时）
    ▼
Node.js + Express 后端
    │
    ├── API 路由层（REST API）
    │   ├── auth        注册/登录
    │   ├── bot         Bot CRUD + 扫码绑定
    │   ├── admin       管理后台
    │   ├── plugins     插件管理
    │   ├── credits     积分系统
    │   ├── personas    AI 人设
    │   └── platform-credentials  平台凭证
    │
    ├── 消息引擎
    │   └── 消息路由器（按 priority 排序）
    │       ├── CPS 插件 (priority=10)
    │       │   ├── 万单联盟（外卖/闪购/酒店）
    │       │   └── 订单侠（淘宝/京东/拼多多）
    │       ├── AI 聊天 (fallback)
    │       └── 其他插件 (priority=50)
    │           ├── 游戏(5个)
    │           ├── 工具(13个)
    │           └── 飞书(6个)
    │
    ├── Bot 管理器
    │   └── iLink 客户端（QR登录/消息轮询/发消息）
    │
    ├── 定时任务
    │   ├── push.js: 每天 10:30/16:30 推送外卖红包
    │   └── report.js: 省钱报告
    │
    └── SQLite 数据库
        └── users/bots/messages/plugins/bot_plugins/credit_transactions/personas

Web 控制台（纯 HTML + Tailwind，微信绿色主题 #07C160）
    └── 首页/登录/注册/控制台/微信绑定/插件市场/管理后台
```

## 消息处理时序

1. 用户微信发消息 → iLink 长轮询接收
2. Bot 管理器收到消息 → 写入 messages 表 (direction=in)
3. 消息路由器按 priority 排序加载已安装插件
4. 逐个尝试插件：
   - CPS 插件 (priority=10) → 关键词/正则匹配 → 匹配则返回回复
   - 匹配失败 → 下一个插件
   - 全部未匹配 → fallback 到 AI 聊天
5. 返回回复 → 通过 iLink 发送消息
6. 写入 messages 表 (direction=out, plugin_name=处理插件)

## CPS 链路

```
用户发消息
  ↓
handleCpsMessage(text, msg)          <- src/plugins/cps/index.js
  │
  ├─ 万单联盟关键词？
  │    关键词: 外卖/美团/闪购/酒店/饿了么
  │    API: GET /activities (HMAC-SHA256 签名, 10分钟缓存)
  │    流程: 签名 → 请求活动列表 → 按关键词匹配 → 返回推广链接
  │
  └─ 订单侠模式？
       ├─ 淘口令 → /tbk/wn_convert
       ├─ 京东链接 → /jd/by_unionid_promotion
       ├─ 拼多多链接 → /pdd/convert
       └─ "查券 xxx" → /tbk/super_search_material
```

## 模块划分

| 模块 | 路径 | 职责 |
|------|------|------|
| 入口 | src/index.js | 初始化 DB → 启动 HTTP → 加载 Bot → 启动定时任务 |
| 配置 | src/config/index.js | 统一配置（环境变量 + .keys 文件） |
| 数据库 | src/db/index.js + schema.sql | SQLite 初始化、Schema、Seed |
| API | src/api/server.js + routes/ | 8 个路由模块 |
| Bot 管理器 | src/bot/manager.js | 多 Bot 实例管理 |
| iLink 客户端 | src/bot/ilink-client.js | QR登录/消息轮询/发消息 |
| 消息路由 | src/bot/message-router.js | 按插件优先级链式分发 |
| 插件系统 | src/plugins/ | 注册表 + 基类 + 沙箱 + 存储 |
| CPS 引擎 | src/plugins/cps/ | 万单联盟 + 订单侠 |
| 定时任务 | src/scheduler/ | 推送 + 报告 |
| 前端 | web/ | 7 个页面 |

## 数据库 (7 张核心表)

| 表 | 说明 | 关键字段 |
|---|------|----------|
| users | 用户 | phone, password_hash, role(user/admin), points, invite_code, invited_by |
| bots | Bot 实例 | user_id, platform(wechat), bot_token, ilink_bot_id, status(online/offline/expired) |
| messages | 消息记录 | bot_id, from_user, direction(in/out), content, plugin_name |
| plugins | 插件 | name, slug, category(tool/game/ai/cps/feishu), is_builtin, version |
| bot_plugins | Bot-插件关联 | bot_id, plugin_id, is_active, priority, config(JSON) |
| credit_transactions | 积分流水 | user_id, amount(+/-), type(register/invite/consume/checkin) |
| personas | AI 人设 | name, system_prompt, icon |

## iLink 微信对接要点

- 接口: https://ilinkai.weixin.qq.com/ilink/bot（微信官方 Bot API）
- 认证: AuthorizationType: ilink_bot_token + Authorization: Bearer <token> + X-WECHAT-UIN: <随机>
- 消息接收: POST /getupdates 长轮询，35s 超时，需持久化 get_updates_buf 游标
- 会话过期: ret=-14 或 HTTP 401/403 → 标记 Bot expired → 需重新扫码
- 认证流程: GET /get_bot_qrcode → 前端轮询状态 → 扫码确认 → 启动消息轮询

## 文件路径速查

```
/Volumes/外置1T/wechat-ai/          <- 本地开发目录
├── src/
│   ├── index.js                    <- 主入口
│   ├── config/index.js             <- 配置管理
│   ├── db/
│   │   ├── index.js                <- SQLite 初始化
│   │   └── schema.sql              <- 表结构
│   ├── api/
│   │   ├── server.js               <- Express 应用
│   │   └── routes/                 <- 8 个路由文件
│   ├── bot/
│   │   ├── ilink-client.js         <- iLink 核心客户端
│   │   ├── manager.js              <- Bot 多实例管理
│   │   └── message-router.js       <- 消息路由
│   ├── plugins/
│   │   ├── cps/
│   │   │   ├── index.js            <- CPS 统一入口
│   │   │   ├── wandanlianmeng.js   <- 万单联盟
│   │   │   └── dingdanxia.js       <- 订单侠
│   │   ├── builtin/                <- 22 个内置插件
│   │   ├── registry.js             <- 插件注册表
│   │   └── sandbox.js              <- 社区插件沙箱
│   ├── scheduler/                  <- 定时推送 + 报告
│   └── utils/logger.js             <- Pino 日志
├── web/                            <- 7 个前端页面
├── data/wechat-ai.db               <- SQLite 数据库
├── .keys/cps-credentials.json      <- CPS 密钥
└── package.json
```

## 决策记录

- 不用好单库（需招商淘客资质）、不用折淘客（打不开）→ 选订单侠
- 订单侠用自己 PID，不抽佣，只收接口服务费
- 去掉 @wechatbot/wechatbot SDK，直接调 iLink HTTP API
- 前端不上 React，纯 HTML + Tailwind，保持零构建
- CPS 插件 priority=10（最高优先级），不可卸载
- SQLite 而非 MySQL：单机够用，运维简单
