# openclaw-memory-resilience

An OpenClaw skill for configuring agent memory to survive compaction and session restarts.

## The Problem

OpenClaw agents lose context when the conversation compacts. This is expected behavior -- but without the right configuration, compaction is **destructive**:

- Instructions you gave mid-session? Gone.
- Preferences the agent learned? Gone.
- The thread of a complex task? Summarized into mush.

Most users discover this when their agent starts behaving oddly. By then the damage is done.

This skill covers the two highest-impact layers of memory durability: **gateway compaction config** and **file architecture**.

## Three Failure Modes

| Mode | What Happens | Symptom |
|------|-------------|---------|
| **A** | Instruction only existed in conversation, never written to a file | Agent forgot a preference you gave it |
| **B** | Compaction summarized it away — lossy, nuance lost | Agent forgot an entire thread or context |
| **C** | Session pruning trimmed old tool results | Agent can't recall what a tool returned (temporary, lossless) |

Quick diagnostic: *Forgot a preference?* → Failure A. *Forgot what a tool returned?* → Failure C. *Forgot the whole thread?* → Failure B.

## Two Compaction Paths

**Good path (maintenance compaction):**
1. Context nears threshold
2. Pre-compaction memory flush fires silently
3. Agent saves important context to disk
4. Compaction summarizes older history
5. Agent continues with fresh context + everything saved to disk

**Bad path (overflow recovery):**
1. Context fills completely, API rejects the request
2. OpenClaw compresses everything at once to recover
3. No memory flush, no saving — maximum context loss

This skill keeps you on the good path.

## Layer 1: Gateway Compaction Config

Apply via `openclaw gateway config.patch`. This is a global default — applies to all agents:

```json
{
  "agents": {
    "defaults": {
      "compaction": {
        "mode": "safeguard",
        "reserveTokensFloor": 40000,
        "memoryFlush": {
          "enabled": true,
          "softThresholdTokens": 4000,
          "systemPrompt": "Session nearing compaction. Store durable memories now.",
          "prompt": "Write any lasting notes to memory/YYYY-MM-DD.md; reply with NO_REPLY if nothing to store."
        }
      },
      "contextPruning": {
        "mode": "cache-ttl",
        "ttl": "1h"
      }
    }
  }
}
```

See [`references/config-explained.md`](./references/config-explained.md) for why each value is set the way it is.

## Layer 2: Context Footer

Add this to every agent's `SOUL.md`. It gives you real-time visibility into context fill and compaction count — the fastest way to know when to act before you hit the bad compaction path.

```markdown
## Context Management (Auto-Applied)
**Every response:** fetch live status via `session_status`, append footer: `🧠 [used]/[total] ([%]) | 🧹 [compactions]`
- Auto-clear: **85% context** OR **6 compactions**
- Warn: **70% context** OR **4 compactions**
- Before clearing: file critical info to memory, then reset
```

The agent calls `session_status` before every reply and appends:

```
🧠 142k/200k (71%) | 🧹 2
```

- `🧠 used/total (%)` — current context fill
- `🧹 N` — number of compactions this session

Each compaction is lossy. One: minor nuance loss. Three: significant degradation. Six: the summary is a summary of summaries. The counter tells you this at a glance.

See [`references/context-footer.md`](./references/context-footer.md) for threshold tuning guidance.

## Layer 3: File Architecture

```
workspace/
├── SOUL.md         — Identity, tone, behavior rules
├── AGENTS.md       — Session rules, memory rules, task execution
├── USER.md         — Who the human is, preferences, context
├── TOOLS.md        — Environment: services, credentials map
├── MEMORY.md       — Curated long-term facts (not raw logs)
├── HEARTBEAT.md    — What to check on each heartbeat poll
└── memory/
    ├── YYYY-MM-DD.md   — Daily raw logs (one per day)
    └── archive/        — Older daily notes (still searchable)
```

**Key rules:**
- `MEMORY.md` = curated, compact, durable facts only. Keep under 5KB.
- `memory/YYYY-MM-DD.md` = raw session logs. Append-only. The pre-compaction flush writes here.
- Never put session logs in `MEMORY.md`.
- Archive daily notes older than ~2 weeks to `memory/archive/` — they stay searchable but don't bloat bootstrap context.

See [`references/file-architecture.md`](./references/file-architecture.md) for bootstrap size limits and memory search setup.

## Quick Start

1. Apply the gateway config via `openclaw gateway config.patch`
2. Add the context footer block to your agent's `SOUL.md`
3. Create the file structure in your workspace
4. Run `/context list` in any session to verify bootstrap files are loading correctly

## Diagnosing Problems

Run `/context list` in any OpenClaw session to see:
- Which bootstrap files loaded and at what size
- Whether any files are TRUNCATED (over per-file limit)
- Total injected chars vs raw chars

If a file shows TRUNCATED: reduce it or raise `bootstrapMaxChars` in config.
If a file is missing entirely: check the workspace path in agent config.

## References

- [`references/config-explained.md`](./references/config-explained.md) — Compaction config field reference
- [`references/context-footer.md`](./references/context-footer.md) — Why the footer matters and how to tune thresholds
- [`references/file-architecture.md`](./references/file-architecture.md) — Bootstrap file patterns, size limits, archival

## Installation

```bash
clawhub install openclaw-memory-resilience
```

Or manually copy this directory into `<your-workspace>/skills/`.

## License

MIT
