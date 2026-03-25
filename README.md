# openclaw-skills

A collection of community skills for [OpenClaw](https://openclaw.ai) — the personal AI agent platform.

These skills solve real problems encountered running multi-agent systems in production. Each is self-contained, generalized for public use, and tested in live deployments.

## Skills

| Skill | Description |
|-------|-------------|
| [openclaw-memory-resilience](./openclaw-memory-resilience/) | Configure agent memory to survive compaction and session restarts |
| [openclaw-cron-manager](./openclaw-cron-manager/) | Make cron jobs survive gateway restarts and updates |
| [caddy-cloudflare-https](./caddy-cloudflare-https/) | Trusted HTTPS for local/private services — no public port 80/443 required |

## Coming Soon

- `openclaw-memory-stack-qdrant` — Production multi-agent memory with Qdrant + Mem0 + local embedding model. For teams running 3+ concurrent agents who need agent-scoped memory isolation, cross-session recall, and concurrent write support.

## Installation

Skills are installed via the [ClawHub CLI](https://clawhub.com):

```bash
clawhub install openclaw-memory-resilience
clawhub install openclaw-cron-manager
clawhub install caddy-cloudflare-https
```

Or manually: copy the skill directory into your OpenClaw workspace `skills/` folder.

## Contributing

PRs welcome. Each skill should include:
- `SKILL.md` — the skill itself (loaded by the agent)
- `references/` — supporting docs referenced from SKILL.md (optional)
- `README.md` — public-facing documentation

## License

MIT
