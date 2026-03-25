---
name: openclaw-cron-manager
description: >
  Manage OpenClaw cron jobs durably. Add, list, restore, and protect cron jobs from being lost
  after gateway restarts or updates. Use when adding new cron jobs, checking existing schedules,
  restoring lost crons after an update, or setting up a new scheduled job that must survive reboots.
  Covers the full resilience pattern: cron-definitions.json as source of truth,
  LaunchAgent auto-restore on boot, and self-healing backup integration.
---

# OpenClaw Cron Manager

## The Problem

OpenClaw cron jobs live in gateway runtime state. A `gateway restart` or `openclaw update` wipes them. They must be re-registered after every restart — unless you use the resilience pattern below.

## The Resilience Pattern

Three layers protect crons from being lost:

1. **`cron-definitions.json`** — canonical source of truth in the workspace, backed up nightly
2. **`ai.openclaw.cron-restore` LaunchAgent** — fires 30s after every login, restores any missing jobs
3. **Backup script self-heal** — nightly backup checks cron count, restores if anything is missing

## Key File Locations

| File | Purpose |
|------|---------|
| `<your-workspace>/scripts/cron-definitions.json` | **Source of truth** — edit this to add/change crons |
| `<your-workspace>/scripts/restore-crons.sh` | Restore script — safe to run any time, skips existing |
| `~/Library/LaunchAgents/ai.openclaw.cron-restore.plist` | LaunchAgent — auto-runs restore on login |

## Adding a New Cron Job

### Step 1 — Add to cron-definitions.json

```json
{
  "name": "my-job-name",
  "description": "What this job does",
  "cron": "0 9 * * 1",
  "tz": "America/Los_Angeles",
  "session": "isolated",
  "announce": true,
  "timeout_seconds": 120,
  "message": "Your agent task message here. Tell it exactly what to do using available tools."
}
```

**Cron field reference:** `minute hour day month weekday` (5-field). Examples:
- `0 2 * * *` — daily at 2:00 AM
- `0 9 * * 1` — every Monday at 9:00 AM
- `0 */6 * * *` — every 6 hours
- `30 8 * * 1-5` — weekdays at 8:30 AM

**Session targets:**
- `isolated` — fresh session, no history (best for scheduled tasks)
- `main` — runs in the main session (use for heartbeat-style tasks)

### Step 2 — Register it immediately

```bash
openclaw cron add \
  --name "my-job-name" \
  --description "What this job does" \
  --cron "0 9 * * 1" \
  --tz "America/Los_Angeles" \
  --session isolated \
  --message "Your task message" \
  --announce \
  --timeout-seconds 120
```

### Step 3 — Verify

```bash
openclaw cron list
```

That's it. The LaunchAgent and backup script will keep it alive from here.

## Removing a Cron Job

1. Remove its entry from `cron-definitions.json`
2. Get the ID: `openclaw cron list`
3. Remove it: `openclaw cron rm <id>`

## Checking Cron Status

```bash
# List all jobs with next run time
openclaw cron list

# Show run history
openclaw cron runs

# Run a job immediately (for testing)
openclaw cron run <id>
```

## Manual Restore

If crons are ever missing (e.g., after a gateway update):

```bash
<your-workspace>/scripts/restore-crons.sh
```

Safe to run any time — skips jobs that already exist by name.

## The Restore Script

Create `<your-workspace>/scripts/restore-crons.sh`:

```bash
#!/bin/bash
# restore-crons.sh — idempotent cron restore from cron-definitions.json
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
DEFINITIONS="$SCRIPT_DIR/cron-definitions.json"

if [ ! -f "$DEFINITIONS" ]; then
  echo "No cron-definitions.json found at $DEFINITIONS"
  exit 1
fi

# Get currently registered job names
EXISTING=$(openclaw cron list --json 2>/dev/null | python3 -c "
import sys, json
data = json.load(sys.stdin)
print('\n'.join(j.get('name','') for j in data))
" 2>/dev/null || echo "")

# Register any missing jobs
python3 - "$DEFINITIONS" "$EXISTING" <<'EOF'
import sys, json, subprocess

with open(sys.argv[1]) as f:
    jobs = json.load(f)

existing = sys.argv[2].strip().split('\n') if sys.argv[2].strip() else []

for job in jobs:
    name = job['name']
    if name in existing:
        print(f"  SKIP {name} (already registered)")
        continue

    cmd = [
        "openclaw", "cron", "add",
        "--name", name,
        "--description", job.get("description", ""),
        "--cron", job["cron"],
        "--tz", job.get("tz", "UTC"),
        "--session", job.get("session", "isolated"),
        "--message", job["message"],
        "--timeout-seconds", str(job.get("timeout_seconds", 120))
    ]
    if job.get("announce"):
        cmd.append("--announce")

    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode == 0:
        print(f"  RESTORED {name}")
    else:
        print(f"  FAILED {name}: {result.stderr.strip()}")
EOF
```

## The LaunchAgent

Create `~/Library/LaunchAgents/ai.openclaw.cron-restore.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>ai.openclaw.cron-restore</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/path/to/your-workspace/scripts/restore-crons.sh</string>
    </array>
    <key>StartInterval</key>
    <integer>30</integer>
    <key>RunAtLoad</key>
    <true/>
    <key>StandardOutPath</key>
    <string>/tmp/openclaw-cron-restore.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/openclaw-cron-restore.log</string>
</dict>
</plist>
```

Load it:
```bash
launchctl load ~/Library/LaunchAgents/ai.openclaw.cron-restore.plist
```

## Gateway Update Procedure

When updating OpenClaw manually:
```bash
npm install -g openclaw@latest
openclaw gateway restart
# Crons auto-restore within 30s via LaunchAgent
```

Verify after: `openclaw cron list` — should show all jobs within 30–60 seconds.
