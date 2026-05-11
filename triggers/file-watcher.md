# Trigger: file-watcher — react to skill roster changes

## What this does

Runs **StaleSkillReaper** whenever the `~/.claude/skills/` directory changes (skill added, removed, or its file structure modified). The reaper's findings are most accurate immediately after a roster mutation; this trigger captures that moment.

## What it costs

One haiku-tier run per skill-directory change. Bounded by debounce — see below. Roster doesn't change often, so this trigger is rarely active.

## Install (macOS — fswatch)

```sh
brew install fswatch
```

Add to your shell profile (`.bashrc` / `.zshrc`):

```sh
# agent-forge — react to skill roster changes
fswatch -o ~/.claude/skills 2>/dev/null | while read -r _; do
  test -f "$HOME/.claude/triggers.pause" && continue
  claude --prompt "/forge steward stale-skill-reaper" --no-interactive \
    >> "$HOME/.claude/triggers.log" 2>&1
  sleep 30  # debounce — coalesce rapid changes
done &
```

Or run as a launchd agent — more durable across reboots. See [fswatch docs](https://github.com/emcrisostomo/fswatch).

## Install (Linux — inotifywait)

```sh
sudo apt install inotify-tools  # Debian/Ubuntu
# or
sudo dnf install inotify-tools  # Fedora
```

Background watcher:

```sh
while inotifywait -e create,delete,modify -r ~/.claude/skills 2>/dev/null; do
  test -f "$HOME/.claude/triggers.pause" && continue
  claude --prompt "/forge steward stale-skill-reaper" --no-interactive \
    >> "$HOME/.claude/triggers.log" 2>&1
  sleep 30
done &
```

For durability, install as a systemd user service. Template (`~/.config/systemd/user/agent-forge-watcher.service`):

```ini
[Unit]
Description=agent-forge skill-roster file watcher
After=default.target

[Service]
ExecStart=/bin/bash -c 'while inotifywait -e create,delete,modify -r %h/.claude/skills; do test -f %h/.claude/triggers.pause && continue; claude --prompt "/forge steward stale-skill-reaper" --no-interactive >> %h/.claude/triggers.log 2>&1; sleep 30; done'
Restart=always

[Install]
WantedBy=default.target
```

```sh
systemctl --user enable --now agent-forge-watcher
```

## Install (Windows — Watchman)

```powershell
# Install Watchman via Chocolatey or scoop
scoop install watchman
```

Configure a Watchman trigger:

```powershell
watchman watch "$env:USERPROFILE\.claude\skills"
watchman -- trigger "$env:USERPROFILE\.claude\skills" `
  agent-forge-reaper -- pwsh -NoProfile -Command "if (-not (Test-Path '$env:USERPROFILE\.claude\triggers.pause')) { claude --prompt '/forge steward stale-skill-reaper' --no-interactive *>> '$env:USERPROFILE\.claude\triggers.log' }"
```

## Debounce

Roster mutations often come in bursts (e.g., installing 5 skills via `git clone` runs into a directory). The 30-second `sleep` after each fire absorbs the burst. If you want tighter / looser debounce, edit the `sleep 30` line.

## When this trigger is most useful

Two scenarios where having immediate StaleSkillReaper output matters:

1. **Audit after adding a batch of skills.** Did the new skills overlap with existing ones? Are any of the old ones now unused?
2. **Audit after Phase 9 Roster Review fired/hired.** Did the change leave any orphans? Did it produce the intended roster?

If you don't regularly mutate `~/.claude/skills/`, this trigger is low-value — stick to the daily cron.

## Uninstall

Kill the background loop / disable the systemd unit / remove the Watchman trigger.
