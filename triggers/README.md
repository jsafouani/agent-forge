# triggers/ — always-on steward activation

This directory contains **trigger configurations** that turn the v2 stewards from manual (`/forge steward X`) into ambient (runs automatically on cron / file change / git event).

## Why this exists

The v2 stewards (StaleSkillReaper, Sentinel, ConfigAuditor, PhaseROI, ChangeGate) are useful only when run. A user who forgets to run them gets no value. v3.0 makes the stewards **ambient** — they run on real-world signals (time, file changes, git events) and contribute to a single prioritized inbox.

Markdown-only constraint: the trigger configs in this directory document the shell commands and OS integration steps. The actual installation is one-time setup the user performs (copy the cron line, install the git hook, etc.). The forge does not run background daemons on the user's behalf — explicit setup keeps trust.

## Files

| File | Trigger source | What it runs | Frequency |
|---|---|---|---|
| `cron-daily.md` | cron / Task Scheduler | All 5 stewards in sequence | Once / day at 06:00 local |
| `git-post-commit.md` | git post-commit hook | ConfigAuditor + Sentinel (lightweight subset) | After every commit |
| `file-watcher.md` | fswatch (macOS) / inotify (Linux) / Watchman (Windows) | StaleSkillReaper | When `~/.claude/skills/` directory changes |

## How activation works at runtime

Each trigger ultimately invokes:

```
claude --prompt "/forge steward <name>" --no-interactive
```

…which dispatches the appropriate steward via the existing v2.0 `/forge steward <name>` arg form. The output goes to `~/.claude/inbox.md` as usual.

After the steward runs, the **salience filter** (Phase: salience-filter, see `<skill-base>/phases/salience-filter.md`) re-ranks the combined inbox so the most urgent items surface to the top.

## Opt-out

Triggers are user-installed. Remove the cron line / uninstall the git hook / kill the file watcher → triggers stop. No global kill switch needed.

To suppress ambient runs temporarily without uninstalling, create `~/.claude/triggers.pause` — every trigger config below checks for this sentinel before invoking a steward.
