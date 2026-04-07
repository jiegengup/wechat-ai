# 微聊 AI 助手 — 变更记录

## 2026-04-04

### docs: PRD 文档审查与重组

- 评审了 `/Volumes/外置1T/wechat-ai/docs/` 下 6 份文档
- 发现 PRD.md / PRD-wechat-bot.md / PRD-cps.md / PRD-cps-v2.md 内容大量重复
- 确认 PRD-wechat-bot.md 是给 coding agent 的技术交接（代码路径合理）
- 开始按照 tokencps 标准格式重组文档

### fix: Gateway 报错排查

- 官方 Anthropic API 超时（90s）+ xinxu 中转站 key 被禁用 + selao 502 三路全挂
- 确认 LLM timeout 默认 90 秒（无显式配置）

## 2026-04-04 之前

### feat: 核心功能全部完成

- 后端全部 API 路由（auth/bot/admin/plugins/credits/personas/storage/platform-credentials）
- iLink 客户端：QR 登录、消息长轮询、发消息、媒体上传下载
- Bot 管理器：多 Bot 实例管理、自动加载、优雅退出
- 消息路由：按插件优先级链式分发
- CPS 引擎：万单联盟签名+调用、订单侠淘宝/京东/拼多多转链+搜索
- 插件系统：22 个内置插件 + 社区插件沙箱
- 前端：7 个页面（微信绿色主题）
- 部署：腾讯云 + Nginx + Let's Encrypt SSL
- 桔梗已实际扫码成功，Bot 上线并开始消息轮询

## 待完成

- AI API Key 配置
- CPS 实际转链测试
- 京东/拼多多转链验证
- 订单追踪和佣金结算表
- 媒体上传 sendImage
- 定时推送
- 非微信适配器完善
