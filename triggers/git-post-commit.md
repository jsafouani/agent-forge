# Trigger: git post-commit — lightweight contract check

## What this does

Runs **ConfigAuditor + Sentinel** (the two stewards that benefit most from fresh signal) after every git commit in any repo the user opts in. This catches config drift and contract violations the moment they're introduced — before they accumulate.

## What it costs

Two haiku-tier steward runs per commit. ~$0.01 per commit. If you commit 50 times a day, that's $0.50/day — a coffee.

## Install per-repo (recommended)

Drop into `<repo>/.git/hooks/post-commit` (make it executable):

```sh
#!/bin/sh
# agent-forge — lightweight post-commit steward run
# Skip if user paused triggers
test -f "$HOME/.claude/triggers.pause" && exit 0

# Skip if repo is opted out (sentinel file in repo root)
test -f "$(git rev-parse --show-toplevel)/.forge-no-post-commit" && exit 0

# Run the two cheap-and-useful stewards in the background; don't block the commit
(
  claude --prompt "/forge steward config-auditor" --no-interactive >> "$HOME/.claude/triggers.log" 2>&1
  claude --prompt "/forge steward sentinel"       --no-interactive >> "$HOME/.claude/triggers.log" 2>&1
) &

exit 0
```

The `( ... ) &` background invocation means the hook does not block `git commit` from returning. Steward output appears in `~/.claude/inbox.md` a few seconds later.

## Install globally (apply to every repo)

Create `~/.git-templates/hooks/post-commit` with the same content, then:

```sh
git config --global init.templateDir '~/.git-templates'
```

Existing repos need:

```sh
git init   # re-runs init in the current repo and copies the global template hooks
```

## Per-repo opt-out

Create `<repo>/.forge-no-post-commit` in any repo where you don't want the trigger to fire. The hook checks for this sentinel.

## Why these two stewards specifically

- **ConfigAuditor** — most likely to detect harm from a fresh commit (model-tier drift, Stable Core integrity).
- **Sentinel** — pattern detector that benefits from fresh signal; if you just merged a PR that violated read-only mandate, you want to know within seconds, not the next morning.

The other three (StaleSkillReaper, PhaseROI, ChangeGate-standalone) don't benefit from being run on every commit:
- StaleSkillReaper needs days/weeks of signal to be meaningful.
- PhaseROI needs 60-day windows and 15+ runs per phase.
- ChangeGate runs inline during /forge runs already (its hook mode); standalone mode is weekly summary.

So per-commit triggers focus on the two stewards where fresh signal matters.

## Quieting the inbox

If post-commit triggers flood your inbox, set `SENTINEL_WINDOW_DAYS=7` and `AUDITOR_WINDOW_DAYS=7` in your shell env — short windows produce fewer findings and avoid duplicate alerts across consecutive commits.

## Uninstall

Delete `<repo>/.git/hooks/post-commit` (or the global template).
