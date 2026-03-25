# caddy-cloudflare-https

Trusted HTTPS for internal/private services using Caddy reverse proxy + Let's Encrypt via Cloudflare DNS-01 challenge. No public port 80/443 required.

## The Problem

Running Home Assistant, Unifi, NAS, or any self-hosted service on your local network and want real HTTPS (not self-signed cert warnings)? Standard ACME requires a public HTTP port for challenge verification. Most home lab setups can't or won't expose that.

DNS-01 challenge is the solution: Let's Encrypt verifies domain ownership by checking a DNS TXT record you control. Cloudflare's API handles the TXT record automatically. No inbound ports, no firewall rules, works on air-gapped private networks.

## What You Get

- Real Let's Encrypt certificates for `*.yourdomain.com` internal services
- Auto-renewal (Caddy handles this)
- Single reverse proxy for all internal services on non-root ports (8443+)
- macOS LaunchAgent for auto-start on login

## Requirements

- Domain managed by Cloudflare (free plan works)
- Cloudflare API token (`Zone → DNS → Edit`)
- macOS (LaunchAgent) or Linux (adapt to systemd)

## Quick Start

1. Build or download Caddy with the `caddy-dns/cloudflare` plugin
2. Store your Cloudflare API token at `~/.config/caddy/.cf_token` (chmod 600)
3. Write your Caddyfile (see SKILL.md for templates)
4. Install the LaunchAgent

## Key Patterns

**Standard HTTP backend:**
```caddy
myservice.yourdomain.com:8443 {
    reverse_proxy localhost:3000
    tls { dns cloudflare {env.CF_API_TOKEN} }
}
```

**WebSocket service (OpenClaw, etc.):**
```caddy
openclaw.yourdomain.com:8443 {
    reverse_proxy localhost:18789 {
        transport http { versions 1.1 }
    }
    tls {
        dns cloudflare {env.CF_API_TOKEN}
        alpn http/1.1
    }
}
```

**Self-signed backend (router, NAS, NVR):**
```caddy
router.yourdomain.com:8444 {
    reverse_proxy https://192.168.1.1 {
        transport http { tls_insecure_skip_verify }
    }
    tls { dns cloudflare {env.CF_API_TOKEN} }
}
```

## Common Operations

```bash
# Reload config (zero downtime)
CF_API_TOKEN=$(cat ~/.config/caddy/.cf_token) ~/caddy-cloudflare reload --config ~/Caddyfile

# Check cert issuer
openssl s_client -connect service.yourdomain.com:8443 -servername service.yourdomain.com 2>/dev/null \
  | openssl x509 -noout -issuer -dates

# Check Caddy logs
tail -f /tmp/caddy.log
```

See [SKILL.md](./SKILL.md) for the full reference including LaunchAgent config, troubleshooting, and all config patterns.

## Installation

```bash
clawhub install caddy-cloudflare-https
```

Or manually copy this directory into `<your-workspace>/skills/`.

## License

MIT
