# 微聊 AI 助手 — 部署指南

> 更新: 2026-04-06

## 服务器信息

| 项目 | 值 |
|------|---|
| 云服务商 | 腾讯云轻量应用服务器 |
| IP | 58.87.69.241 |
| OS | Ubuntu |
| SSH | `ssh root@58.87.69.241`（密钥认证） |
| 面板 | 宝塔面板 |

## 端口分配

| 端口 | 项目 |
|------|------|
| 3000 | **微聊 AI 助手** |
| 3100 | tokencps |
| 5432 | PostgreSQL (Docker, multica 用) |

## Nginx 配置

| 域名 | 指向 | SSL |
|------|------|-----|
| `wechat.aiwuyi.top` | `127.0.0.1:3000` | Let's Encrypt，2026-06-29 到期 |

## 部署流程

```bash
# 本地开发
cd /Volumes/外置1T/wechat-ai
npm install
npm run dev
# 访问 http://localhost:3000

# 部署到腾讯云
ssh root@58.87.69.241
cd /root/wechat-ai
pm2 restart wechat-ai
```

## 环境变量

| 变量名 | 默认值 | 说明 |
|--------|--------|------|
| `PORT` | `3000` | HTTP 端口 |
| `JWT_SECRET` | `dev-secret-change-me` | JWT 密钥（生产必须改） |
| `ADMIN_TOKEN` | `wechat-ai-admin-2024` | 管理后台 token（生产必须改） |
| `DB_PATH` | `data/wechat-ai.db` | SQLite 数据库路径 |
| `WDLM_APP_KEY` | 从 .keys 读 | 万单联盟 appKey |
| `WDLM_APP_SECRET` | 从 .keys 读 | 万单联盟 appSecret |
| `DDX_API_KEY` | 从 .keys 读 | 订单侠 apiKey |
| `AI_BASE_URL` | `https://api.openai.com/v1` | AI 接口地址 |
| `AI_API_KEY` | 空 | AI 模型 API Key（未配置则返回 fallback） |
| `AI_MODEL` | `gpt-3.5-turbo` | AI 模型名 |
| `ILINK_API_BASE` | `https://ilinkai.weixin.qq.com/ilink/bot` | iLink API |

## PM2 管理

```bash
pm2 list
pm2 restart wechat-ai
pm2 logs wechat-ai
pm2 save
pm2 startup
```

## 数据库

```bash
sqlite3 data/wechat-ai.db
.tables
SELECT * FROM users;
SELECT * FROM bots;
SELECT COUNT(*) FROM messages;
SELECT * FROM plugins WHERE is_builtin=1;
```

## SSL 证书

- 签发: Let's Encrypt
- 到期: 2026-06-29
- 自动续期: certbot renew（需确认 cron 配置）
- ⚠️ 宝塔面板 nftables 防火墙可能拦截 SSH，`nft flush ruleset` 清除

## 测试账号

- 手机号: `16676666791` / 密码: `jiegeng666@`（admin 角色）
- 邀请码: `ac856aba`
