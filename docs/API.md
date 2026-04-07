# 微聊 AI 助手 — API 接口文档

> 更新: 2026-04-06
> Base URL: `https://wechat.aiwuyi.top`（生产）/ `http://localhost:3000`（本地）
> 统一响应: REST JSON（无统一 wrapper，按 route 而定）

## 认证

JWT (jsonwebtoken, 7 天有效期) + bcryptjs 密码哈希。

### POST /api/auth/register
注册新用户。
```json
// Request
{ "phone": "string", "password": "string", "username?": "string", "inviteCode?": "string" }
// Response
{ "token": "jwt", "user": {...} }
```

### POST /api/auth/login
```json
{ "phone": "string", "password": "string" }
// Response: 同 register
```

### GET /api/auth/me
获取当前用户信息（需 Authorization: Bearer <token>）。

### POST /api/auth/change-password
修改密码。

---

## Bot 管理

### POST /api/bots
创建 Bot 实例。自动安装所有内置插件。

### GET /api/bots
获取当前用户的 Bot 列表。

### GET /api/bots/:id
获取 Bot 详情（含状态、插件列表）。

### POST /api/bots/:id/qr-start
启动扫码绑定流程。调用 iLink 获取二维码。
```json
// Response
{ "qrCode": "https://api.qrserver.com/v1/create-qr-code/?data=..." }
```

### GET /api/bots/:id/qr-status
轮询扫码状态（5 秒超时）。
```json
// Response
{ "status": "waiting|scaned|confirmed", "botToken?: "xxx" }
```

### DELETE /api/bots/:id
删除 Bot（停止轮询 + 删除 DB 记录）。

---

## CPS 接口

> CPS 主要通过微信消息触发，API 仅供测试/调试。

### POST /api/cps/test
测试 CPS 转链。
```json
{ "text": "外卖", "botId": 1 }
// Response
{ "reply": "🔥 美团外卖红包\n👉 点击领取：https://..." }
```

---

## 积分系统

### POST /api/credits/checkin
每日签到获取积分。

### GET /api/credits/history
积分流水记录。

---

## 插件管理

### GET /api/plugins
获取所有可用插件列表。

### POST /api/bots/:botId/plugins/:pluginId/toggle
切换插件启用/禁用状态。

### PATCH /api/bots/:botId/plugins/:pluginId/priority
修改插件优先级。

---

## AI 人设

### GET /api/personas
获取人设列表（10 个官方预置人设）。

### POST /api/personas
创建自定义人设。

---

## 管理后台（需 admin 角色）

### GET /api/admin/dashboard
运营数据。

### GET /api/admin/users
用户列表。

### POST /api/admin/broadcast
群发消息。

---

## iLink 内部 API（后端调用，非暴露接口）

### 获取登录二维码
`GET https://ilinkai.weixin.qq.com/ilink/bot/get_bot_qrcode?bot_type=3`

### 查询扫码状态
`GET https://ilinkai.weixin.qq.com/ilink/bot/get_qrcode_status?qrcode=xxx`

### 长轮询接收消息
`POST https://ilinkai.weixin.qq.com/ilink/bot/getupdates`
Body: `{ "get_updates_buf": "<游标>" }`

### 发送消息
`POST https://ilinkai.weixin.qq.com/ilink/bot/sendmessage`

### 发送正在输入
`POST https://ilinkai.weixin.qq.com/ilink/bot/sendtyping`

### 获取配置
`POST https://ilinkai.weixin.qq.com/ilink/bot/getconfig`

### 获取媒体上传 URL
`POST https://ilinkai.weixin.qq.com/ilink/bot/getuploadurl`

认证头:
```
AuthorizationType: ilink_bot_token
Authorization: Bearer <bot_token>
X-WECHAT-UIN: <随机 base64>
```

---

## CPS 平台内部 API

### 万单联盟

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 获取活动列表 | GET | `/activities` | 带 HMAC-SHA256 签名，10 分钟缓存 |

Base URL: `http://open.wandanlianmeng.com/api/v1/open-api`

### 订单侠

| 接口 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 淘宝转链 | POST | `tbk/tbk_wn_convert` | 淘口令转推广链接 |
| 优惠券搜索 | GET | `tbk/tbk_super_search_material` | 关键词搜索 |
| 京东转链 | GET | `jd/jd_by_unionid_promotion_common_get` | 京东链接转链 |
| 拼多多转链 | GET | `pdd/pdd_ddk_goods_promotion_url_generate` | 拼多多链接转链 |

Base URL: `http://api.tbk.dingdanxia.com`
认证: URL 参数 `apikey=xxx`
