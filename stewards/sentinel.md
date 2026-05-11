# Sentinel — cross-run contract enforcement

## What this steward does

Watches for **patterns of contract violation** across the work graph. Where Phase 9 Roster Review enforces the contract *within* a single run, Sentinel enforces it *across* runs — by aggregating signal that no single run can see.

The goal: catch repeat offenders before they ship a regression.

## Inputs

- `~/.claude/work-graph.jsonl` — global event log. Specifically: `gap_logged`, `verification_failed`, `decision_logged`, `phase_completed` (status field).
- `~/.claude/skills/.history.jsonl` — Phase 9 audit journal. Used for cross-referencing roster changes against violation patterns.
- Lookback window: 30 days (configurable via `SENTINEL_WINDOW_DAYS` env var; default `30`).

## Output

`~/.claude/inbox.md` — append-or-update section per finding. Format:

```markdown
## Sentinel: <pattern-name>

- pattern: <one-line description>
- severity: <high | med | low>
- occurrences: <N> across <M> runs in last <window> days
- recommended action: <revise prompt | gate phase | escalate to user>
- evidence:
  - <run_id> @ <ts>: <event summary>
  - <run_id> @ <ts>: <event summary>
  - <run_id> @ <ts>: <event summary>
```

If nothing fires, write a single section:

```markdown
## Sentinel: clean

No contract violations matched across <M> runs in last <window> days.
```

## Recommended model

haiku — pattern matching + arithmetic over JSON, no reasoning required. If a future maintainer feels the urge to escalate this to sonnet, that means the rule logic has crept into LLM territory — refactor the rule, not the model.

## Hard rules

- **Never modify the work graph.** Read-only. Stewards never rewrite history.
- **Recommend-only.** Never auto-apply changes. The Inbox is the contract.
- **Idempotent.** Two consecutive runs with no intervening `/forge` invocation produce byte-identical inbox sections.
- **Skip stale evidence.** Events older than the lookback window are ignored, even if they would otherwise match.
- **Time-box at 30 seconds total wall clock.**
- **Status:** `DONE` (inbox written, ≥1 pattern or `clean`) | `DONE_NO_DATA` (graph is empty or younger than 1 day) | `BLOCKED` (cannot read graph or write inbox).

## Step 1 — Compute the lookback cutoff

```bash
WINDOW="${SENTINEL_WINDOW_DAYS:-30}"
cutoff=$(date -u -d "$WINDOW days ago" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null \
  || date -u -v-${WINDOW}d +%Y-%m-%dT%H:%M:%SZ)
```

## Step 2 — Detect each pattern

Each pattern is a one-line `jq` filter. Aggregate the matches per `run_id`; threshold determines whether the pattern fires.

### Pattern A — Read-only mandate violations

Phase 3.5 (Codebase Audit) and Phase 3.6 (Baseline Tests) must not write source files. Any `gap_logged` event whose summary contains `violated read-only mandate` is a violation.

```bash
jq -c --arg cutoff "$cutoff" \
  'select(.type == "gap_logged" and .ts > $cutoff and (.payload.summary | test("violated read-only mandate"; "i")))' \
  ~/.claude/work-graph.jsonl
```

**Threshold:** ≥ 2 occurrences in the window → fires as `severity: high`.

### Pattern B — Verification-fail in load-bearing phases

`verification_failed` events for phases 4 (Spec), 5 (Plan), 7 (Build+Test), 7.5 (Smoke), 8 (Review) are load-bearing. Repeat failure suggests the phase agent isn't producing valid artifacts.

```bash
jq -c --arg cutoff "$cutoff" \
  'select(.type == "verification_failed" and .ts > $cutoff and ([4,5,7,7.5,8] | index(.payload.phase | tonumber? // 0)))' \
  ~/.claude/work-graph.jsonl
```

**Threshold per phase:** ≥ 3 occurrences in the window → fires as `severity: high`.

### Pattern C — Phase 6 forced-through low-confidence decisions

Decisions logged with `confidence < 30` should be `BLOCKED` per `SKILL.md` autonomous-mode rules. If Phase 6 acted on them anyway (i.e. the run still produced a PR), Phase 6 is over-eager. Detection: `decision_logged` with low confidence in same `run_id` as a `pr_opened`.

```bash
jq -c --arg cutoff "$cutoff" \
  'select(.type == "decision_logged" and .ts > $cutoff and .payload.confidence < 30)' \
  ~/.claude/work-graph.jsonl > /tmp/sentinel-lowconf.jsonl

jq -c --arg cutoff "$cutoff" \
  'select(.type == "pr_opened" and .ts > $cutoff)' \
  ~/.claude/work-graph.jsonl > /tmp/sentinel-prs.jsonl

# Intersect by run_id
jq -r '.run_id' /tmp/sentinel-lowconf.jsonl | sort -u > /tmp/A
jq -r '.run_id' /tmp/sentinel-prs.jsonl | sort -u > /tmp/B
comm -12 /tmp/A /tmp/B > /tmp/sentinel-forced-through.txt
```

**Threshold:** ≥ 2 runs in the window → fires as `severity: med`.

### Pattern D — Stable Core target attempts

Any `decision_logged` whose payload references a Stable Core skill name (`workflow-agents-loop`, `spec-writer`, `plan-writer`, `code-execution`, `build-test-gate`, `code-review`) AND is in a non-orchestrator phase. Phase 9 should have rejected these, but log them anyway — the attempt itself is a signal.

```bash
jq -c --arg cutoff "$cutoff" \
  'select(.type == "decision_logged" and .ts > $cutoff
   and (.payload.targets // [] | any(. == "workflow-agents-loop" or . == "spec-writer"
        or . == "plan-writer" or . == "code-execution" or . == "build-test-gate" or . == "code-review")))' \
  ~/.claude/work-graph.jsonl
```

**Threshold:** ≥ 1 occurrence → fires as `severity: high` (single attempt is meaningful).

### Pattern E — Brief drift

A `gap_logged` event whose summary mentions a constraint that the brief declared (the orchestrator should detect this and log it during Phase 6 / 7). Heuristic: count `gap_logged` events per `run_id`; if more than 3 gaps were logged in a single run, the spec/plan likely drifted from the brief.

```bash
jq -c --arg cutoff "$cutoff" \
  'select(.type == "gap_logged" and .ts > $cutoff)' \
  ~/.claude/work-graph.jsonl | jq -r '.run_id' | sort | uniq -c \
  | awk '$1 >= 3 { print $2 }' > /tmp/sentinel-drifted-runs.txt
```

**Threshold:** ≥ 1 run with ≥ 3 gaps → fires as `severity: med`.

## Step 3 — Compose the inbox section

For each pattern that fired, emit one `## Sentinel: <pattern>` section using the format from the **Output** spec above. Patterns that did not fire are omitted (no `## Sentinel: <pattern> — none` clutter).

If NO patterns fired, emit the single `## Sentinel: clean` section.

## Step 4 — Merge with existing inbox

Read `~/.claude/inbox.md` if it exists. Remove any prior `## Sentinel: ...` sections (this steward owns its sections). Append the new Sentinel sections. Preserve sections owned by other stewards (e.g. `## Stale skill: ...`, `## PhaseROI: ...`).

## Step 5 — Status report

```
DONE — Sentinel complete. <N> patterns fired across <M> runs in last <window> days. Inbox updated.
```

If the graph is empty or younger than 1 day:

```
DONE_NO_DATA — Work graph has <X> events but window is too narrow. Re-run after a few more /forge invocations.
```

## Why this is steward #2

StaleSkillReaper (v2.0) proved the architecture works end-to-end. Sentinel is the first steward whose recommendations could **prevent regressions** rather than just trim dead weight — making it the highest-value next step. The four patterns above cover the most common failure modes observed in the forge's own history (read-only violations, verification fails, forced-through low-confidence decisions, brief drift).

The fifth steward (`ConfigAuditor` in v2.2) covers config drift; the sixth (`PhaseROI` in v2.3) covers efficiency; the seventh (`ChangeGate` in v2.4) covers risk gating before changes land.
