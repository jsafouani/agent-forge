# Trigger: daily cron — full steward sweep

## What this does

Runs all five v2 stewards once per day at 06:00 local time, then runs the salience filter to re-rank the combined inbox. Most users want this — it gives you a fresh, prioritized inbox to triage with morning coffee, without remembering to invoke anything.

## What it costs

Each steward run is haiku-tier or cheap-sonnet; total per day is ~$0.05–0.20 depending on graph size. Negligible relative to a /forge run.

## Install (Unix — Linux / macOS)

Add to your crontab (`crontab -e`):

```cron
# agent-forge — daily steward sweep at 06:00 local time
0 6 * * * /bin/sh -c 'test -f $HOME/.claude/triggers.pause || ( \
  claude --prompt "/forge steward stale-skill-reaper" --no-interactive ; \
  claude --prompt "/forge steward sentinel"             --no-interactive ; \
  claude --prompt "/forge steward config-auditor"       --no-interactive ; \
  claude --prompt "/forge steward phase-roi"            --no-interactive ; \
  claude --prompt "/forge steward change-gate"          --no-interactive ; \
  claude --prompt "/forge inbox --reprioritize"         --no-interactive \
)' >> $HOME/.claude/triggers.log 2>&1
```

Output goes to `~/.claude/triggers.log` — rotate this file periodically (it grows). Add `logrotate` config if you care.

## Install (Windows — Task Scheduler)

Create `%USERPROFILE%\.claude\triggers\cron-daily.ps1`:

```powershell
if (Test-Path "$env:USERPROFILE\.claude\triggers.pause") { exit 0 }

$stewards = @('stale-skill-reaper', 'sentinel', 'config-auditor', 'phase-roi', 'change-gate')
foreach ($s in $stewards) {
  & claude --prompt "/forge steward $s" --no-interactive *>> "$env:USERPROFILE\.claude\triggers.log"
}
& claude --prompt "/forge inbox --reprioritize" --no-interactive *>> "$env:USERPROFILE\.claude\triggers.log"
```

Register the scheduled task:

```powershell
$action = New-ScheduledTaskAction -Execute "pwsh" -Argument "-File $env:USERPROFILE\.claude\triggers\cron-daily.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At 6am
Register-ScheduledTask -TaskName "agent-forge-daily-stewards" -Action $action -Trigger $trigger -Description "Runs agent-forge v2 stewards daily"
```

## What you see the next morning

`~/.claude/inbox.md` will have all five stewards' findings, re-ranked by the salience filter:

```markdown
# Forge Inbox (re-prioritized 2026-05-12 06:02:14)

> Sorted by salience = (severity_weight × confidence) ÷ (1 + days_since_observed).

## [HIGH] ChangeGate: pending approval — forge-csv-cli-20260512
- gate-manual: 612 LOC change touched /Auth/

## [HIGH] ConfigAuditor: Stable Core integrity — spec-writer.md
- byte count dropped 28% since baseline (8341 → 6003)

## [MED] Sentinel: read-only mandate violations
- pattern fired 3× in last 30 days

## [LOW] PhaseROI: skip-without-regret — phase 2A
- 12 of 15 runs skipped, 0 tied gaps

## [LOW] Stale skill: csv-importer
- unused 67 days
```

## Disable temporarily

```bash
touch ~/.claude/triggers.pause
```

The cron line will still fire, but the trigger script checks for the sentinel and exits cleanly without running anything.

To re-enable:

```bash
rm ~/.claude/triggers.pause
```

## Disable permanently

Remove the cron line / unregister the scheduled task.

## Why 06:00?

Most engineers want the inbox ready when they start work, not at midnight when builds and stewards might collide. Adjust to taste.
