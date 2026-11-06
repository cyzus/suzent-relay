# Suzent Relay

English | [简体中文](README.zh-CN.md)

Your own relay for **[Suzent](https://github.com/cyzus/suzent)**: reach the
Suzent on your computer from the Suzent app on your phone, and let services such
as GitHub call your webhooks, from anywhere. No port forwarding, no VPN, and no
server you don't run yourself.

```
desktop Suzent ──outbound WSS──▶  your relay  ◀──HTTPS/WSS── Suzent on your phone
                                       ▲
                                       └──── HTTPS POST /hooks/… ── GitHub, scripts
```

Your computer dials **out** to the relay and keeps the connection open. The
phone and webhook callers send their requests to the relay, which passes them
down that connection: HTTP, streaming (SSE) and WebSockets. Phone traffic is
end-to-end encrypted between the app and your computer, so the relay only ever
sees ciphertext. The relay stores nothing on disk.

Suzent has **no hosted relay**. You deploy this one, so the traffic stays on
infrastructure you control.

## Deploy

Pick one. Each takes a few minutes.

### Option A: Cloudflare Workers (free, no server)

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/cyzus/suzent-relay)

Or from a clone:

```bash
git clone https://github.com/cyzus/suzent-relay && cd suzent-relay
npm install
npx wrangler deploy        # prints https://suzent-relay.<you>.workers.dev
npx wrangler secret put RELAY_REGISTRATION_KEY   # recommended, see below
```

This runs on the Workers **Free plan**, using SQLite-backed Durable Objects with
WebSocket hibernation, so an idle connection costs next to nothing. TLS and a
`*.workers.dev` hostname come with it. To use your own domain, uncomment
`routes` in `wrangler.jsonc`.

### Option B: Docker with automatic HTTPS (any VPS)

You need a server with ports 80 and 443 open and a DNS record pointing at it.

```bash
git clone https://github.com/cyzus/suzent-relay && cd suzent-relay
cp .env.example .env       # set RELAY_DOMAIN and RELAY_REGISTRATION_KEY
docker compose up -d
```

[Caddy](https://caddyserver.com) obtains and renews the Let's Encrypt
certificate on its own. Your relay is at `https://relay.example.com`.

### Option C: Behind your own reverse proxy

```bash
docker run -d --restart unless-stopped -p 127.0.0.1:8787:8787 ghcr.io/cyzus/suzent-relay
# or, without Docker (Node ≥ 20):
npx github:cyzus/suzent-relay
```

Then proxy your HTTPS hostname to `127.0.0.1:8787`. WebSockets must be allowed
and responses must not be buffered:

<details>
<summary>nginx</summary>

```nginx
location / {
    proxy_pass http://127.0.0.1:8787;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_buffering off;
    proxy_read_timeout 1h;
    client_max_body_size 128m;
}
```
</details>

<details>
<summary>Caddy</summary>

```
relay.example.com {
    reverse_proxy 127.0.0.1:8787 {
        flush_interval -1
    }
}
```
</details>

Check it with `curl https://relay.example.com/healthz`.

## Connect Suzent

1. On your computer, open **Settings → Devices → Relay** in Suzent.
2. Enter your relay's `https://` address and, if you set one, the registration
   key. Suzent connects and shows **Connected**.
3. Pair your phone by scanning the QR code as usual. The QR code now carries
   the relay address, so the phone uses it whenever it can't reach your
   computer directly.

Webhook tasks in **Settings → Automation** then also show a relay URL you can
give to GitHub and other services on the internet.

Suzent only keeps the relay connection while it is running, so turn on the
**Background Service** if you want your phone to reach it with the window
closed.

## Configuration

Set everything through environment variables: in `.env` for Docker, or with
`wrangler secret put NAME` (secrets) or `vars` in `wrangler.jsonc` (plain
settings) on Cloudflare. All of them are optional.

| Variable | Meaning |
| --- | --- |
| `RELAY_REGISTRATION_KEY` | A shared key computers must present before they can connect. Paste it into Suzent as the registration key. Recommended. |
| `RELAY_TOKEN_SECRET` | Use per-user signed tokens instead of a shared key, for one relay shared by several people. Takes precedence over `RELAY_REGISTRATION_KEY`. |
| `RELAY_TOKEN_MESSAGE` | What a computer without a valid token is told (default: `A valid relay token is required`). |
| `PORT` / `HOST` | Listen address (Node: `8787` / `0.0.0.0`). |
| `MAX_BODY_MB` | Request-body cap in MB (Node: `128`; Cloudflare: `100`). |
| `MAX_BUFFERED_MB` | Node: cap on request bodies held in memory at once, in MB (`512`). |
| `MAX_AGENTS` | Node: most computers connected at once (`10000`). |
| `MAX_REQUESTS_PER_AGENT` / `MAX_SOCKETS_PER_AGENT` | Node: concurrent HTTP requests / WebSockets per computer (`256` each). |
| `MAX_CONNECTIONS` | Node: TCP connection cap for the process (unset: no cap). |
| `STALL_TIMEOUT_S` | Cut off a client that reads nothing for this many seconds (`60`). |
| `RELAY_PROTOCOL` | Set to `1` to stop offering [protocol 2](docs/PROTOCOL.md#protocol-2). Default `2`. |
| `LOCATION_HINT` | Cloudflare only: where tunnels live, for example `apac`, `enam` or `weur`. |

The relay also contains push-notification (APNs/FCM) and TURN support inherited
from upstream. Suzent doesn't use them yet; leave those variables unset.

### Who can use your relay

With no key set, anyone who knows your relay's address can open tunnels on it.
They still can't reach *your* computer: that takes its 256-bit channel token,
and phone traffic additionally needs your computer's key from the pairing QR
code. Set `RELAY_REGISTRATION_KEY` anyway, to keep strangers from using your
server at all.

To share one relay between several people and be able to cut off individuals,
set `RELAY_TOKEN_SECRET` and mint a token for each person:

```bash
RELAY_TOKEN_SECRET=… npm run mint-token -- alice 365     # user id, days valid
# Docker: docker compose exec relay node scripts/mint-token.mjs alice
```

## Security

- **End-to-end encryption for phones.** The app and your computer run an
  authenticated key exchange over the tunnel, using your computer's public key
  from the pairing QR code, and encrypt everything after it with AES-256-GCM.
  Whoever runs the relay can't read or alter the traffic, and sessions have
  forward secrecy.
- **Webhooks are not end-to-end encrypted.** GitHub and similar services can only
  send plain HTTPS to the relay, so the relay sees webhook payloads. Suzent uses
  a separate channel for webhooks and accepts only `/hooks/` requests on it.
- **Nothing to steal.** The relay routes by `sha256(channel token)` and never
  writes tokens or traffic to disk or logs.
- **Always serve it over HTTPS.** Channel tokens travel in the URL.
- **Revoking.** Reset the relay channel in Suzent's settings to cut off every
  phone and webhook URL that used it. Revoke a single phone in **Settings →
  Mobile** as usual.

## Development

```bash
npm install
npm start                    # Node edition on :8787
npx wrangler dev             # Cloudflare edition, locally

npm test                     # end-to-end suite against the Node edition
npm run test:cloudflare      # the same suite against `wrangler dev`
node test/e2e.mjs https://relay.example.com   # smoke-test a deployment
```

| File | |
| --- | --- |
| `index.mjs` | Node edition (the `ws` package is its only dependency) |
| `cloudflare/worker.mjs` | Cloudflare Workers + Durable Objects edition |
| `shared.mjs` | Code both editions use |
| `docs/PROTOCOL.md` | The wire protocol |

## Credits

Suzent Relay is a fork of [yetone/alma-relay](https://github.com/yetone/alma-relay)
by yetone, used under the MIT license. The wire protocol is unchanged, so
fixes from upstream can be merged as they are.

## License

[MIT](LICENSE)
