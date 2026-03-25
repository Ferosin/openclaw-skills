# openclaw-skills

A collection of community skills for [OpenClaw](https://openclaw.ai) — the personal AI agent platform.

These skills solve real problems encountered running multi-agent systems in production. Each is self-contained, generalized for public use, and tested in live deployments.

## Skills

| Skill | Description |
|-------|-------------|
| [openclaw-memory-resilience](./openclaw-memory-resilience/) | Configure agent memory to survive compaction and session restarts |

## Coming Soon

- `openclaw-cron-manager` — Durable cron job management that survives gateway restarts and updates
- `caddy-cloudflare-https` — Trusted HTTPS for local/private services via Caddy + Cloudflare DNS-01

## Installation

Skills are installed via the [ClawHub CLI](https://clawhub.com):

```bash
clawhub install openclaw-memory-resilience
```

Or manually: copy the skill directory into your OpenClaw workspace `skills/` folder and reference it in your agent config.

## Contributing

PRs welcome. Each skill should include:
- `SKILL.md` — the skill itself (loaded by the agent)
- `references/` — supporting docs referenced from SKILL.md
- `README.md` — public-facing documentation

## License

MIT
