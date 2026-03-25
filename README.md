# openclaw-skills

A collection of community skills for [OpenClaw](https://openclaw.ai) — the personal AI agent platform.

These skills solve real problems encountered running multi-agent systems in production. Each is self-contained, generalized for public use, and tested in live deployments.

## Skills

| Skill | Description | Complexity |
|-------|-------------|------------|
| [openclaw-memory-resilience](./openclaw-memory-resilience/) | Configure agent memory to survive compaction and session restarts | Low — config only |
| [openclaw-cron-manager](./openclaw-cron-manager/) | Make cron jobs survive gateway restarts and updates | Low — scripts + LaunchAgent |
| [caddy-cloudflare-https](./caddy-cloudflare-https/) | Trusted HTTPS for local/private services — no public port 80/443 required | Medium — custom binary + DNS |
| [openclaw-memory-stack-qdrant](./openclaw-memory-stack-qdrant/) | Production multi-agent memory: Qdrant + Mem0 + local embeddings | High — 3 services + plugin |

## Skill Selection Guide

**"My agent keeps forgetting things"** → [openclaw-memory-resilience](./openclaw-memory-resilience/)

**"My cron jobs disappear after gateway restarts"** → [openclaw-cron-manager](./openclaw-cron-manager/)

**"I want real HTTPS certs for my local services"** → [caddy-cloudflare-https](./caddy-cloudflare-https/)

**"I'm running multiple agents and need persistent, agent-scoped memory"** → [openclaw-memory-stack-qdrant](./openclaw-memory-stack-qdrant/)

## Installation

Skills are installed via the [ClawHub CLI](https://clawhub.com):

```bash
clawhub install openclaw-memory-resilience
clawhub install openclaw-cron-manager
clawhub install caddy-cloudflare-https
clawhub install openclaw-memory-stack-qdrant
```

Or manually: copy the skill directory into your OpenClaw workspace `skills/` folder.

## Contributing

PRs welcome. Each skill should include:
- `SKILL.md` — the skill itself (loaded by the agent)
- `references/` — supporting docs referenced from SKILL.md (optional)
- `README.md` — public-facing documentation

## License

MIT
