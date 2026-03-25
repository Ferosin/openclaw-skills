# openclaw-cron-manager

An OpenClaw skill for making cron jobs survive gateway restarts and updates.

## The Problem

OpenClaw cron jobs live in gateway runtime state. A `gateway restart` or `openclaw update` wipes them. This is the most common "why did my scheduled task stop running?" cause for OpenClaw users.

This skill implements a three-layer resilience pattern that keeps your crons alive through restarts, updates, and reboots.

## How It Works

```
cron-definitions.json          ← source of truth, edit this
       │
       ├── LaunchAgent (30s after login) → restore-crons.sh → openclaw cron add
       │
       └── Nightly backup script → cron count check → restore-crons.sh if missing
```

1. **`cron-definitions.json`** — every cron job defined as JSON. This is the record. If a job isn't here, it doesn't survive a restart.
2. **LaunchAgent** — fires 30 seconds after every login, runs the restore script. Idempotent — skips already-registered jobs.
3. **Self-heal in backup** — nightly backup checks cron count and restores if anything is missing. Belt and suspenders.

## Quick Start

1. Create `<your-workspace>/scripts/cron-definitions.json` with your jobs
2. Create `<your-workspace>/scripts/restore-crons.sh` (see SKILL.md)
3. Install the LaunchAgent (see SKILL.md)
4. Register jobs once with `openclaw cron add`

From that point, restarts and updates handle themselves.

## Adding a Job

Add to `cron-definitions.json`:

```json
[
  {
    "name": "daily-summary",
    "description": "Post a daily status summary",
    "cron": "0 8 * * *",
    "tz": "America/Los_Angeles",
    "session": "isolated",
    "announce": true,
    "timeout_seconds": 120,
    "message": "Generate a brief status summary of active projects and post it to the team channel."
  }
]
```

Then register immediately:
```bash
openclaw cron add --name "daily-summary" --cron "0 8 * * *" \
  --session isolated --message "..." --announce --timeout-seconds 120
```

The LaunchAgent takes it from there.

## Common Operations

```bash
# List all jobs with next run time
openclaw cron list

# Run a job immediately (testing)
openclaw cron run <id>

# Manual restore (after update or if jobs go missing)
<your-workspace>/scripts/restore-crons.sh

# Remove a job (also remove from cron-definitions.json)
openclaw cron rm <id>
```

## Cron Expression Reference

| Expression | Meaning |
|-----------|---------|
| `0 2 * * *` | Daily at 2:00 AM |
| `0 9 * * 1` | Every Monday at 9:00 AM |
| `0 */6 * * *` | Every 6 hours |
| `30 8 * * 1-5` | Weekdays at 8:30 AM |
| `*/15 * * * *` | Every 15 minutes |

## Installation

```bash
clawhub install openclaw-cron-manager
```

Or manually copy this directory into `<your-workspace>/skills/`.

## License

MIT
