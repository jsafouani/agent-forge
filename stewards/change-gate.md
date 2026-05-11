# ChangeGate — generalized risk gating before auto-apply

## What this steward does

Gates risky changes **before** Phase 9 Roster Review auto-applies them. The default in agent-forge is auto-apply (with all the safety guards: Stable Core untouchable, snapshots, audit journal, contract pinning, champion-challenger trials, validation gate, independent reviewer). ChangeGate generalizes that into a **standing rule set** the user controls.

Where Phase 8 (Code Review) and Phase 9 (Roster Review) gate things per-run, ChangeGate provides **always-on, contract-defined risk policy**. A user can declare:

> *"Auto-apply PRs under 200 LOC that pass review; pause anything touching `/Auth/`; require manual approval for any Stable Core change."*

…and ChangeGate enforces it across every run.

## Inputs

- **Per-run inputs (when invoked from Phase 9 hook):**
  - Proposed change diff from Phase 6 (file paths + LOC counts).
  - Current Stable Core file list from `<skill-base>/STABLE_CORE.md`.
  - The run's `## Decision Log` for cross-checking risky decisions.
  - The run's `## Review Status` from Phase 8.

- **Per-run inputs (when invoked standalone via `/forge steward change-gate`):**
  - The work graph for the last 7 days — to summarize what was auto-applied vs. gated recently.

- **Policy config (read by both modes):**
  - `~/.claude/change-gate.policy.md` — the user's risk policy as a markdown file with structured rules. If missing, a sensible default policy is used (documented below).

## Output

### Mode 1: Phase 9 hook (called inline during a /forge run)

Writes a single line to `forge-context.md ## Roster Changes Proposed` for each proposed change:

```
- [change-gate] <change>: <verdict> (<rule fired>)
```

Where `<verdict>` is one of:
- `auto-apply` — passes all gate rules
- `gate-manual` — fails a gate rule; written to inbox for manual approval
- `block` — hits a hard-stop rule (e.g. Stable Core target attempt)

Returns the verdict for the orchestrator to act on.

### Mode 2: Standalone steward (`/forge steward change-gate`)

Writes a summary section to `~/.claude/inbox.md`:

```markdown
## ChangeGate: weekly summary

Last 7 days: <N> auto-applied, <M> gated for manual approval, <K> blocked.

### Pending manual approval (<M>)
1. <run_id> — <change>: <rule that gated it>
2. ...

### Blocked (<K>)
1. <run_id> — <change>: <hard-stop reason>
2. ...

### Recent auto-applied (<N> — sample of 5 most recent)
1. <run_id> — <change>
2. ...
```

If nothing pending: `## ChangeGate: clean — no changes gated in last 7 days.`

## Default policy (when `~/.claude/change-gate.policy.md` is absent)

```markdown
# ChangeGate default policy

## Rules (evaluated in order; first match wins)

1. **Block:** any change targeting a Stable Core file (`workflow-agents-loop`, `spec-writer`, `plan-writer`, `code-execution`, `build-test-gate`, `code-review`, or the work-graph event schema).
2. **Gate-manual:** any change touching paths matching `/(auth|security|identity|session|token|password|credential)/i`.
3. **Gate-manual:** any change touching paths matching `/(TenantId|tenant_id|multi.?tenant)/i`.
4. **Gate-manual:** any change touching files matching `(\\.env|appsettings.*\\.json|secrets\\.[A-Za-z]+|\\.pem|\\.key)$`.
5. **Gate-manual:** any change > 500 LOC across all touched files (counting `+` lines in unified diff).
6. **Gate-manual:** any change whose run has ≥ 1 decision logged with `confidence < 30%` AND no `verification_failed` for that confidence in the same run.
7. **Auto-apply:** everything else (default permissive).
```

The user edits this file to tighten or loosen the policy. ChangeGate parses the markdown structure and applies the rules in declared order.

## Recommended model

haiku — pattern matching against file paths + line-count arithmetic. The judgment is in the rules file, not in the model.

## Hard rules

- **Never modify the proposed change.** ChangeGate either approves the change to auto-apply, gates it for manual review, or blocks it. It never edits the diff.
- **Block ≠ delete.** A blocked change is recorded in the inbox; the user can still apply it manually after the run.
- **Idempotent across modes.** Running ChangeGate standalone after the same week of runs produces the same summary.
- **Time-box at 20 seconds per invocation in hook mode.** The Phase 9 hook is on the critical path; if ChangeGate takes longer than 20s, the orchestrator times it out and defaults to `gate-manual` for safety.
- **Status:** `DONE` | `BLOCKED` (cannot read policy file or skill base).

## Step 1 — Load policy

```bash
POLICY="${HOME}/.claude/change-gate.policy.md"
if [ ! -f "$POLICY" ]; then
  # Use the embedded default — see "Default policy" section above
  cat > /tmp/cg-policy.md <<'POLICY_EOF'
[ ... default policy rules ... ]
POLICY_EOF
  POLICY="/tmp/cg-policy.md"
fi
```

Parse the policy file into an ordered list of `(verdict, regex_or_predicate)` tuples by reading each `## Rules` line.

## Step 2 — Evaluate proposed change (hook mode)

The orchestrator passes the diff via `forge-context.md` (Phase 9 section). Extract:

- File paths touched (from `git diff --name-only` against the worktree's base branch).
- LOC count (from `git diff --shortstat`).
- Decision Log entries (already in `forge-context.md`).

For each rule in policy order:

```bash
# Stable Core rule (rule 1)
if echo "$touched_files" | grep -qE '(SKILL\.md|spec-writer\.md|plan-writer\.md|code-execution\.md|build-test-gate\.md|code-review\.md)'; then
  echo "block: Stable Core target attempt"
  exit
fi

# Auth/security paths (rule 2)
if echo "$touched_files" | grep -qiE '/(auth|security|identity|session|token|password|credential)/'; then
  echo "gate-manual: auth/security path touched"
  exit
fi

# … and so on for each rule
```

First match wins. If no rule fires, the verdict is `auto-apply`.

## Step 3 — Record verdict (hook mode)

Append to `forge-context.md ## Roster Changes Proposed`:

```
- [change-gate] <change-summary>: <verdict> (<rule-fired>)
```

If verdict is `gate-manual` or `block`, also append a section to `~/.claude/inbox.md`:

```markdown
## ChangeGate: pending approval — <run_id>

- run_id: <run_id>
- change: <change-summary>
- verdict: <gate-manual | block>
- rule fired: <rule description>
- files touched: <list>
- LOC delta: <+N -M>
- manual approval steps:
  1. Review the worktree at <worktree_path>
  2. If approved: `cd <worktree>; git push origin <branch>; gh pr create ...`
  3. If rejected: `cd <worktree>; git worktree remove .` (destroys the work)
```

## Step 4 — Weekly summary (standalone mode)

```bash
WINDOW=7
cutoff=$(date -u -d "$WINDOW days ago" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null \
  || date -u -v-${WINDOW}d +%Y-%m-%dT%H:%M:%SZ)

# Auto-applied: pr_opened events
auto=$(jq -c --arg cutoff "$cutoff" \
  'select(.type == "pr_opened" and .ts > $cutoff)' \
  ~/.claude/work-graph.jsonl | wc -l)

# Gated for manual: parse inbox for "## ChangeGate: pending approval" sections
gated=$(grep -c "^## ChangeGate: pending approval" ~/.claude/inbox.md 2>/dev/null || echo 0)

# Blocked: same as gated but with verdict: block
# (Need to parse each pending-approval section; left as future work in v2.4 — for now, count both together)
```

Compose the `## ChangeGate: weekly summary` section per the **Output** spec.

## Step 5 — Status report

```
DONE — ChangeGate complete. (<auto>/<gated>/<blocked> in last 7 days)
```

## Phase 9 integration (SKILL.md hook)

Phase 9 Roster Review must consult ChangeGate before its auto-apply step. The hook is a one-line check:

```
Before any roster change with auto_apply=true is committed, dispatch the change-gate steward
in hook mode with the change details. If verdict is "block" or "gate-manual", downgrade the
change to "propose-to-user" and record the verdict in the audit journal. If verdict is
"auto-apply", proceed with normal auto-apply path.
```

The user's `CONTRACT.md` already permits this read — ChangeGate uses the same shared inbox.

## Why this is steward #5 (final pre-v3.0)

The four stewards together form the **observability fleet**:

- Sentinel — watches behavior (what the system does)
- ConfigAuditor — watches configuration (how it's set up)
- PhaseROI — watches economics (whether it earns its cost)
- ChangeGate — watches **proposed changes** (whether the next change is safe)

With ChangeGate in place, the v2 fleet is complete. v3.0 lifts the framework from "manual stewards triggered by `/forge steward X`" to "always-on stewards triggered by cron / file-watcher / git-hook," with a salience filter that ranks the combined output for the user's attention.
