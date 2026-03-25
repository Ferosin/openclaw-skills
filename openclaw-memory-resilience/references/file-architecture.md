# File Architecture for Memory Durability

## Bootstrap Files (always in context)

These are injected at every session start from disk. They survive compaction because they're reloaded from disk, not from conversation history.

Standard set for an OpenClaw agent workspace:

```
workspace/
├── SOUL.md       — Identity, tone, behavior rules
├── AGENTS.md     — Session rules, memory rules, task execution
├── USER.md       — Who the human is, preferences, context
├── TOOLS.md      — Environment: SSH hosts, services, credentials map
├── IDENTITY.md   — Name, email, handles (optional, keep tiny)
├── MEMORY.md     — Curated long-term facts (not raw logs)
├── HEARTBEAT.md  — What to check on each heartbeat poll
└── memory/
    ├── YYYY-MM-DD.md   — Daily raw logs (one per day)
    └── archive/        — Older daily notes (still searchable)
```

## MEMORY.md vs Daily Notes

**MEMORY.md** — curated, compact, durable facts. Credentials, infrastructure, decisions, preferences. Updated manually when something important should persist. Keep under 5KB.

**memory/YYYY-MM-DD.md** — raw session logs. Append-only. The pre-compaction flush writes here. Archive after ~2 weeks.

**Never** put session logs in MEMORY.md. They bloat bootstrap and get stale fast.

## The Archival Pattern

When daily notes accumulate (>10-15 files), move older ones to `memory/archive/`:

```bash
mkdir -p workspace/memory/archive
mv workspace/memory/2026-01-*.md workspace/memory/archive/
mv workspace/memory/2026-02-*.md workspace/memory/archive/
```

Archived files remain fully searchable via `memory_search` — they just don't bloat bootstrap context.

Keep the most recent 5-7 daily notes in `memory/` directly. Archive the rest.

## Bootstrap Size Limits

Check with `/context list` in any session:

- `bootstrapMaxChars` — per-file limit (default 20,000 chars). Files over this get TRUNCATED.
- `bootstrapTotalMaxChars` — aggregate cap (default 150,000 chars ~50K tokens).

If files are being truncated, either reduce the file or raise the limit:

```json
{
  "agents": {
    "defaults": {
      "bootstrapMaxChars": 30000,
      "bootstrapTotalMaxChars": 200000
    }
  }
}
```

## Retrieval Backend Options

The file architecture works with any OpenClaw retrieval backend. Two tiers:

### Level 1: QMD (built-in, zero infrastructure)

QMD is OpenClaw's built-in hybrid BM25 + vector search. Single embedded binary, no external services required. Good for single-agent setups.

```json
{
  "memory": {
    "backend": "qmd",
    "qmd": {
      "includeDefaultMemory": true,
      "update": {
        "interval": "5m",
        "onBoot": true
      }
    }
  }
}
```

Force a re-index after adding/moving files:
```bash
qmd update && qmd embed
```

### Level 2: Qdrant + Mem0 (production multi-agent)

For multi-agent deployments needing agent-scoped memory, concurrent writes, and cross-session recall. Requires external services: Qdrant vector store + Mem0 OSS + embedding model + small reasoning model.

See the [openclaw-memory-stack-qdrant](../openclaw-memory-stack-qdrant/) skill for the full setup guide.

**When to upgrade from QMD to Qdrant+Mem0:**
- Running 3+ agents concurrently (QMD has no concurrent write support)
- Need agent-scoped memory isolation (Mem0 handles this natively)
- QMD's embedded binary breaks after OS or Node.js version changes
- Want cross-session fact extraction and relationship tracking

## Manual Memory Discipline

The automated flush is a safety net, not a guarantee. Complement it with manual saves:

- Before switching tasks: tell the agent to save current context to memory
- After important decisions: explicitly ask for a memory write
- Use `/compact` proactively (before context fills) rather than reactively

The timing trick: save context → `/compact` → give new instructions. New instructions land in fresh context and have maximum lifespan before the next compaction.
