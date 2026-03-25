---
name: caddy-cloudflare-https
description: Set up and manage trusted HTTPS for local/private services using Caddy reverse proxy with Let's Encrypt certificates via Cloudflare DNS-01 challenge. Use when configuring HTTPS for internal services (no public port 80/443 required), adding new virtual hosts, rotating certs, troubleshooting Caddy, or managing the LaunchAgent on macOS.
---

# Caddy + Cloudflare HTTPS

Non-root Caddy on macOS proxying internal services with real Let's Encrypt certs via DNS-01 challenge. No public port 80/443 required — works entirely on private networks.

## Why This Approach

Standard HTTPS setup requires a public port 80 or 443 for ACME HTTP-01 challenge. If your services are internal-only (home lab, self-hosted, private network), that's not an option.

DNS-01 challenge solves this: Let's Encrypt verifies domain ownership via a DNS TXT record, not an HTTP request. Combined with Cloudflare DNS API, cert issuance is fully automated — no inbound ports, no firewall rules.

## Prerequisites

- A domain managed by Cloudflare (any plan including free)
- A Cloudflare API token with `Zone → DNS → Edit` permission for your domain
- macOS host (for LaunchAgent setup — adapt plist to systemd for Linux)

## Key Files

| File | Purpose |
|------|---------|
| `~/caddy-cloudflare` | Custom Caddy binary with `caddy-dns/cloudflare` plugin |
| `~/Caddyfile` | Main Caddy config |
| `~/.config/caddy/.cf_token` | Cloudflare API token (chmod 600) |
| `~/Library/LaunchAgents/com.caddy.server.plist` | macOS LaunchAgent (auto-start on login) |

## Getting the Custom Binary

The stock Caddy binary doesn't include the Cloudflare DNS plugin. Build or download a custom binary:

**Option A: xcaddy (build from source)**
```bash
go install github.com/caddyserver/xcaddy/cmd/xcaddy@latest
xcaddy build --with github.com/caddy-dns/cloudflare
mv caddy ~/caddy-cloudflare
```

**Option B: Download from Caddy download page**
https://caddyserver.com/download — select `caddy-dns/cloudflare` plugin before downloading.

## Cloudflare API Token

Create at https://dash.cloudflare.com/profile/api-tokens with:
- Template: "Edit zone DNS"
- Zone Resources: Include → Specific zone → `<your-domain>`

Store it:
```bash
mkdir -p ~/.config/caddy
echo "your-token-here" > ~/.config/caddy/.cf_token
chmod 600 ~/.config/caddy/.cf_token
```

## Caddyfile

```caddy
{
    acme_dns cloudflare {env.CF_API_TOKEN}
    http_port  8080
    https_port 8443
}

# Standard service (HTTP backend)
service1.yourdomain.com:8443 {
    reverse_proxy localhost:3000
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1
    }
}

# WebSocket-compatible service (e.g. OpenClaw)
openclaw.yourdomain.com:8443 {
    reverse_proxy localhost:18789 {
        transport http {
            versions 1.1        # WebSocket requires HTTP/1.1
        }
    }
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1
        alpn http/1.1           # Prevents H2 negotiation (breaks WebSocket)
    }
}

# Self-signed backend (e.g. router, NAS, NVR with self-signed cert)
router.yourdomain.com:8444 {
    reverse_proxy https://192.168.1.1 {
        transport http { tls_insecure_skip_verify }
    }
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        resolvers 1.1.1.1
    }
}
```

## Critical Config Notes

- **WebSocket services:** Require both `versions 1.1` in transport AND `alpn http/1.1` in TLS block. HTTP/2 breaks WebSocket upgrade. Both are required — one alone is not enough.
- **Self-signed backends:** Use `tls_insecure_skip_verify` in the transport block. Common for routers, NAS devices, NVRs that ship with self-signed certs.
- **Non-root ports (8443+):** Avoids needing root or `setcap`. Use any port >1024.
- **`http_port 8080` in global block:** Prevents conflict with port 80. If another service is on 8080, change to `8079` or any free port.
- **`auto_https off` not needed:** Custom ports prevent Caddy from trying port 80 for HTTP-01 redirect.

## LaunchAgent (macOS)

Create `~/Library/LaunchAgents/com.caddy.server.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.caddy.server</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/sh</string>
        <string>-c</string>
        <string>CF_API_TOKEN=$(cat /Users/youruser/.config/caddy/.cf_token) /Users/youruser/caddy-cloudflare run --config /Users/youruser/Caddyfile</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/caddy.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/caddy.log</string>
</dict>
</plist>
```

Load:
```bash
launchctl load ~/Library/LaunchAgents/com.caddy.server.plist
```

## Common Operations

### Check cert validity
```bash
openssl s_client -connect service1.yourdomain.com:8443 -servername service1.yourdomain.com 2>/dev/null \
  | openssl x509 -noout -issuer -dates
# issuer should show: O=Let's Encrypt
```

### Restart Caddy
```bash
pkill -f caddy-cloudflare
CF_API_TOKEN=$(cat ~/.config/caddy/.cf_token) ~/caddy-cloudflare run --config ~/Caddyfile &
```

### Reload config without restart (zero downtime)
```bash
CF_API_TOKEN=$(cat ~/.config/caddy/.cf_token) ~/caddy-cloudflare reload --config ~/Caddyfile
```

### Add a new virtual host
1. Add DNS A record in Cloudflare (DNS-only, gray cloud, pointing to your host's LAN IP)
2. Add new block in `~/Caddyfile` following the pattern above
3. Reload Caddy

### Unload/reload LaunchAgent
```bash
launchctl unload ~/Library/LaunchAgents/com.caddy.server.plist
launchctl load ~/Library/LaunchAgents/com.caddy.server.plist
```

## Troubleshooting

**Cert not issuing:**
- Check Cloudflare token has DNS Edit permission for the correct zone
- Verify DNS record exists in Cloudflare (even pointing to a private IP is fine for DNS-01)
- Check Caddy logs: `tail -f /tmp/caddy.log`

**WebSocket not connecting:**
- Confirm both `versions 1.1` (transport) and `alpn http/1.1` (TLS) are set
- HTTP/2 must be disabled at both levels

**Port conflict:**
- Check what's on 8080: `lsof -i :8080`
- Change `http_port` in global block if needed
