# ConfigAuditor — model-tier + Stable Core drift detection

## What this steward does

Audits the gap between **declared configuration** (what phase prompts say) and **actual runtime behavior** (what the work graph recorded). Three drift categories:

1. **Model-tier drift** — a phase declares `## Recommended model: sonnet` but runs `opus` most of the time. The default is wrong, OR per-task hints are escalating without justification.
2. **Stable Core integrity drift** — one of the 6 immutable skill files is missing, has changed name in its frontmatter, or is shorter than the last known length (suggesting truncation / accidental edit).
3. **Phase verification drift** — a phase declares a `verification` command in `SKILL.md` but the orchestrator hasn't been running it (no corresponding `verification_failed` events ever logged, even on demonstrably broken runs).

The goal: catch silent config rot before it produces a behavioral regression.

## Inputs

- `~/.claude/work-graph.jsonl` — for `phase_completed.payload.model` and `verification_failed` events.
- Phase prompt files: `<skill-base>/phases/*.md` — for declared `## Recommended model:` values.
- Stable Core files: `<skill-base>/SKILL.md`, `<skill-base>/phases/spec-writer.md`, `<skill-base>/phases/plan-writer.md`, `<skill-base>/phases/code-execution.md`, `<skill-base>/phases/build-test-gate.md`, `<skill-base>/phases/code-review.md`.
- `<skill-base>/STABLE_CORE.md` — for the authoritative list.
- Lookback window: 30 days (configurable via `AUDITOR_WINDOW_DAYS` env var).

## Output

`~/.claude/inbox.md` — append-or-update section per finding. Three section types:

```markdown
## ConfigAuditor: model-tier drift — <phase-name>

- declared: <model>
- actual (last <N> runs): <distribution>
- recommendation: <promote default | demote default | leave + fix per-task hints>
- evidence: see work-graph events for run_ids: <list>

## ConfigAuditor: Stable Core integrity — <file>

- expected: <last-known SHA or skill name>
- actual: <missing | name changed | byte count delta>
- severity: <high>
- action required: investigate immediately; this file is load-bearing

## ConfigAuditor: verification dormant — <phase>

- declared verification: <command from SKILL.md>
- runs in last <window>: <N>
- verification_failed events: 0
- likely cause: orchestrator skipping the check, or every run is perfect (unlikely)
- recommendation: instrument the check; add a forced-fail test
```

If everything is clean, emit a single section:

```markdown
## ConfigAuditor: clean

No model-tier drift, Stable Core integrity verified, verification commands appear active.
```

## Recommended model

haiku — file reads, frontmatter parsing, basic counting. No reasoning.

## Hard rules

- **Read-only on phase prompts.** Never edits a phase prompt file. If a recommendation needs to land, the user (or a future steward with a contract clause) applies it.
- **Stable Core files are read-only AND read-very-carefully.** Compute a hash, log it, but never modify.
- **Recommend-only.** Same contract as every other steward.
- **Idempotent.**
- **Time-box at 45 seconds total wall clock.** File reads + jq aggregation; this one can be slightly slower than Sentinel.
- **Status:** `DONE` | `DONE_NO_DATA` (no `phase_completed` events with model field present) | `BLOCKED` (cannot read skill base directory).

## Step 1 — Resolve skill base directory

```bash
# Caller (orchestrator dispatching this steward) sets SKILL_BASE_DIR.
# Fallback: look for the user-level install.
SKILL_BASE="${SKILL_BASE_DIR:-$HOME/.claude/skills/workflow-agents-loop}"
test -d "$SKILL_BASE" || { echo "BLOCKED — skill base $SKILL_BASE not found"; exit 1; }
```

## Step 2 — Build declared-model map from phase prompts

```bash
for phase in "$SKILL_BASE"/phases/*.md; do
  name=$(basename "$phase" .md)
  declared=$(grep -A1 "^## Recommended model" "$phase" | tail -1 | awk '{print $1}' | tr -d '`')
  echo "$name|$declared"
done > /tmp/auditor-declared.txt
```

## Step 3 — Build actual-model distribution from work graph

```bash
WINDOW="${AUDITOR_WINDOW_DAYS:-30}"
cutoff=$(date -u -d "$WINDOW days ago" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null \
  || date -u -v-${WINDOW}d +%Y-%m-%dT%H:%M:%SZ)

jq -r --arg cutoff "$cutoff" \
  'select(.type == "phase_completed" and .ts > $cutoff and .payload.model)
   | "\(.payload.name)|\(.payload.model)"' \
  ~/.claude/work-graph.jsonl \
  | sort | uniq -c \
  | awk '{print $2 "|" $1}' > /tmp/auditor-actual.txt
```

## Step 4 — Detect model-tier drift

For each phase:

- If declared = `opus` and actual median = `opus` → fine.
- If declared = `sonnet` and actual = `opus` in ≥ 60% of runs → **drift** (`recommendation: promote default to opus`).
- If declared = `opus` and actual = `sonnet` or `haiku` in ≥ 60% of runs → **drift** (`recommendation: demote default; per-task hints are doing the work`).
- If declared = `haiku` and actual ≠ `haiku` in any run → **drift** (`recommendation: investigate why haiku was overridden — likely a missing per-task hint`).

Threshold: minimum 5 runs of the phase in the window for the rule to apply (avoid noise on rarely-used phases).

## Step 5 — Stable Core integrity check

Read `<skill-base>/STABLE_CORE.md` for the authoritative list. For each Stable Core skill name:

```bash
for skill in workflow-agents-loop spec-writer plan-writer code-execution build-test-gate code-review; do
  if [ "$skill" = "workflow-agents-loop" ]; then
    target="$SKILL_BASE/SKILL.md"
  else
    target="$SKILL_BASE/phases/$skill.md"
  fi

  if [ ! -f "$target" ]; then
    echo "$skill|MISSING|"
    continue
  fi

  # Frontmatter name field (only meaningful for SKILL.md)
  fm_name=$(awk '/^name:/{print $2; exit}' "$target")
  size=$(wc -c < "$target" | tr -d ' ')

  echo "$skill|OK|name=$fm_name|bytes=$size"
done > /tmp/auditor-core.txt
```

Anything with `MISSING` → fires as `severity: high`. Anything with `name=` that doesn't match the directory/file convention → fires.

For byte-count tracking, store a baseline on first run at `~/.claude/auditor-baseline.txt`. On subsequent runs, alert if any Stable Core file has lost > 25% of its bytes (suggests truncation / accidental edit). Increases are fine.

## Step 6 — Verification dormancy check

Read `<skill-base>/SKILL.md` and extract the verification-command table (the block under "Evidence-based phase verification"). For each phase with a declared verification command:

```bash
# Number of runs that should have produced verification events
runs=$(jq -c --arg cutoff "$cutoff" --arg p "$phase" \
  'select(.type == "phase_completed" and .ts > $cutoff and .payload.name == $p)' \
  ~/.claude/work-graph.jsonl | wc -l)

# Number of actual verification_failed events for that phase
fails=$(jq -c --arg cutoff "$cutoff" --arg p "$phase" \
  'select(.type == "verification_failed" and .ts > $cutoff and (.payload.phase | tostring) == $p)' \
  ~/.claude/work-graph.jsonl | wc -l)

# If runs ≥ 20 and fails == 0, the verification is suspiciously quiet.
# (Adjust threshold per phase — Build+Test Gate genuinely should mostly pass.)
```

Fires only for: spec-writer, plan-writer, code-execution, code-review. Build+Test Gate and Smoke Test legitimately pass most of the time; flagging them produces too much noise.

## Step 7 — Compose inbox sections

For each drift / integrity / dormancy finding, emit a `## ConfigAuditor: <type>` section per the **Output** spec. Strip prior `## ConfigAuditor: ...` sections from the existing inbox before appending — this steward owns its sections.

If nothing fired, emit the single `## ConfigAuditor: clean` section.

## Step 8 — Status report

```
DONE — ConfigAuditor complete. <N> drifts, <M> integrity findings, <K> dormancy findings.
```

If the graph has no `phase_completed` events with a `model` field (Phase 11 Observe doesn't emit it yet, or all runs predate v2.0):

```
DONE_NO_DATA — Work graph has phase events but no model field on any of them. Re-run after a few v2.0+ /forge invocations.
```

## Why this is steward #3

Sentinel (v2.1) watches behavioral patterns. ConfigAuditor watches **configuration**. Together they cover both axes of drift — *what the system does* vs. *how the system is set up to do it*. The next steward (PhaseROI in v2.3) closes the third axis: *whether each piece is worth its cost*.
