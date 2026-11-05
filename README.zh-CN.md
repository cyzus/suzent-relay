# Suzent Relay

[English](README.md) | 简体中文

**[Suzent](https://github.com/cyzus/suzent)** 的自部署中继：在任何地方用手机上的 Suzent App 连到你电脑上的 Suzent，也让 GitHub 这类服务能调用你的 webhook。不用端口转发，不用 VPN，也不经过任何不是你自己运行的服务器。

```
电脑上的 Suzent ──出站 WSS──▶  你的中继  ◀──HTTPS/WSS── 手机上的 Suzent
                                    ▲
                                    └──── HTTPS POST /hooks/… ── GitHub、脚本
```

电脑**主动连出去**并保持连接。手机和 webhook 调用方把请求发给中继，中继再顺着这条连接转给电脑，支持 HTTP、流式（SSE）和 WebSocket。手机流量在 App 和电脑之间端到端加密，中继只能看到密文。中继不往磁盘写任何东西。

Suzent **不提供官方中继**。中继由你自己部署，流量只经过你控制的服务器。

## 部署

三选一，每种都只要几分钟。

### 方式 A：Cloudflare Workers（免费，不需要服务器）

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/cyzus/suzent-relay)

或者从源码部署：

```bash
git clone https://github.com/cyzus/suzent-relay && cd suzent-relay
npm install
npx wrangler deploy        # 输出 https://suzent-relay.<你>.workers.dev
npx wrangler secret put RELAY_REGISTRATION_KEY   # 建议设置，见下文
```

运行在 Workers **免费套餐**上，使用基于 SQLite 的 Durable Objects 和 WebSocket 休眠，空闲连接几乎不产生费用。自带 TLS 和 `*.workers.dev` 域名。要用自己的域名，取消 `wrangler.jsonc` 里 `routes` 的注释。

### 方式 B：Docker + 自动 HTTPS（任意 VPS）

需要一台开放 80 和 443 端口、并有域名解析到它的服务器。

```bash
git clone https://github.com/cyzus/suzent-relay && cd suzent-relay
cp .env.example .env       # 设置 RELAY_DOMAIN 和 RELAY_REGISTRATION_KEY
docker compose up -d
```

[Caddy](https://caddyserver.com) 会自动申请和续期 Let's Encrypt 证书。中继地址是 `https://relay.example.com`。

### 方式 C：放在你自己的反向代理后面

```bash
docker run -d --restart unless-stopped -p 127.0.0.1:8787:8787 ghcr.io/cyzus/suzent-relay
# 或者不用 Docker（Node ≥ 20）：
npx github:cyzus/suzent-relay
```

然后把 HTTPS 域名代理到 `127.0.0.1:8787`。必须允许 WebSocket，并关闭响应缓冲（nginx 和 Caddy 配置见[英文 README](README.md#option-c-behind-your-own-reverse-proxy)）。

用 `curl https://relay.example.com/healthz` 检查是否正常。

## 连接 Suzent

1. 在电脑上打开 Suzent 的 **设置 → 设备 → 中继**。
2. 填入中继的 `https://` 地址；如果设置了注册密钥，也一起填上。Suzent 连上后会显示**已连接**。
3. 照常扫码配对手机。二维码里会带上中继地址，手机连不上电脑时就自动走中继。

之后 **设置 → 自动化** 里的 webhook 任务也会显示一个中继地址，可以填给 GitHub 等公网服务。

Suzent 只在运行时保持中继连接。想在关掉窗口后手机仍能连上，请开启**后台服务**。

## 配置

所有配置都通过环境变量：Docker 写在 `.env` 里；Cloudflare 用 `wrangler secret put NAME`（密钥）或 `wrangler.jsonc` 的 `vars`（普通设置）。全部可选，完整列表见[英文 README](README.md#configuration)。最常用的两个：

| 变量 | 含义 |
| --- | --- |
| `RELAY_REGISTRATION_KEY` | 电脑连接前必须出示的共享密钥，在 Suzent 里填为注册密钥。建议设置。 |
| `RELAY_TOKEN_SECRET` | 多人共用一个中继时，改用每人一个的签名 token，可以单独吊销。优先级高于 `RELAY_REGISTRATION_KEY`。 |

中继里还有从上游继承的推送通知（APNs/FCM）和 TURN 支持，Suzent 暂时不用，相关变量留空即可。

## 安全

- **手机流量端到端加密。** App 和电脑在隧道里用配对二维码里的电脑公钥做一次带认证的密钥交换，之后的内容全部用 AES-256-GCM 加密。运行中继的人既读不到也改不了，并且有前向保密。
- **Webhook 不是端到端加密的。** GitHub 这类服务只能用普通 HTTPS 发给中继，所以中继能看到 webhook 内容。Suzent 为 webhook 单独用一个频道，这个频道只接受 `/hooks/` 请求。
- **没有可偷的东西。** 中继只按 `sha256(频道 token)` 路由，不把 token 和流量写进磁盘或日志。
- **一定要用 HTTPS。** 频道 token 在 URL 里传输。
- **吊销。** 在 Suzent 设置里重置中继频道，所有经它连接的手机和 webhook 地址都会失效。单台手机照常在 **设置 → 设备** 里撤销。

## 致谢

Suzent Relay fork 自 yetone 的 [yetone/alma-relay](https://github.com/yetone/alma-relay)，遵循 MIT 协议。wire 协议没有改动，上游的修复可以直接合并过来。

## 许可证

[MIT](LICENSE)
