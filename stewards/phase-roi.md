# PhaseROI — cost-vs-value calibration

## What this steward does

Calibrates each phase against its actual contribution to outcomes. Where Sentinel watches behavior and ConfigAuditor watches configuration, PhaseROI watches **economics** — which phases are worth their token cost and which are dead weight.

Three calibration axes:

1. **Skip-without-regret detector** — phases that completed as `DONE_SKIPPED` in N of last M runs without any downstream `gap_logged` event tied to the skip. If a phase can be skipped repeatedly without consequence, the phase is a deprecation candidate.
2. **Low-confidence-then-trust pattern** — phases whose decisions consistently log `confidence < 50%` and downstream phases nonetheless act on them. Two readings: (a) the phase's model tier is too low (escalate), or (b) the phase is asking the wrong questions and should be redesigned.
3. **No-downstream-reference detector** — phases that produce output (write to `forge-context.md`) but whose output is never referenced by any later phase's `## Context sections needed`. Generates work nobody reads.

The goal: trim the loop. Every phase that stays should earn its place.

## Inputs

- `~/.claude/work-graph.jsonl` — `phase_completed`, `decision_logged`, `gap_logged` events.
- Phase prompt files: `<skill-base>/phases/*.md` — for `## Context sections needed` declarations.
- `<skill-base>/SKILL.md` — for the phase map and verification command list.
- Lookback window: 60 days (configurable via `ROI_WINDOW_DAYS`; longer than other stewards because ROI signal accumulates slowly).
- Minimum sample: 15 runs in the window for any phase to be evaluated (suppresses noise on rarely-run phases).

## Output

`~/.claude/inbox.md` — append-or-update section per finding. Three section types:

```markdown
## PhaseROI: skip-without-regret — <phase-name>

- skipped in <N> of last <M> runs (<percent>%)
- downstream gaps tied to skip: <N> (threshold for retain: ≥ 3)
- recommendation: deprecate in <next-version> | demote to opt-in
- evidence:
  - <run_id> @ <ts>: SKIPPED, downstream complete
  - <run_id> @ <ts>: SKIPPED, downstream complete
  - <run_id> @ <ts>: SKIPPED, downstream complete

## PhaseROI: low-confidence pattern — <phase-name>

- decisions logged: <N>
- median confidence: <pct>%
- downstream PRs that acted on these decisions: <N>
- reading: <model tier too low | phase scope too broad | rules need updating>
- recommendation: <escalate from sonnet to opus | narrow phase scope | revise prompt rules>

## PhaseROI: orphan output — <phase-name>

- produces section: <section name>
- referenced by downstream phases: 0
- recommendation: remove output OR add downstream consumer
- detection method: grep'd every <skill-base>/phases/*.md for the section name; no matches
```

If nothing fires:

```markdown
## PhaseROI: clean

All phases meet ROI thresholds in last <window> days.
```

## Recommended model

haiku — counting + grep, no judgment. The recommendations themselves are deterministic outputs of the rules.

## Hard rules

- **Read-only.** Never modifies phase prompts or SKILL.md.
- **Recommend-only.** Never auto-deprecates a phase. Removal is a user decision, period.
- **60-day window** — shorter windows produce too much noise; ROI signal is slow.
- **Min 15 runs per phase** before evaluation fires.
- **Idempotent.**
- **Time-box at 60 seconds total wall clock.**
- **Status:** `DONE` | `DONE_NO_DATA` (graph has < 15 runs of any phase) | `BLOCKED` (cannot read skill base or work graph).

## Step 1 — Cutoff + min-sample setup

```bash
WINDOW="${ROI_WINDOW_DAYS:-60}"
MIN_RUNS=15
cutoff=$(date -u -d "$WINDOW days ago" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null \
  || date -u -v-${WINDOW}d +%Y-%m-%dT%H:%M:%SZ)
SKILL_BASE="${SKILL_BASE_DIR:-$HOME/.claude/skills/workflow-agents-loop}"
```

## Step 2 — Per-phase run counts and status distribution

```bash
jq -c --arg cutoff "$cutoff" \
  'select(.type == "phase_completed" and .ts > $cutoff)
   | "\(.payload.name)|\(.payload.status)"' \
  ~/.claude/work-graph.jsonl \
  | sort | uniq -c \
  | awk '{print $2 "|" $1}' > /tmp/roi-status-dist.txt
```

For each phase: total runs, `DONE` count, `DONE_SKIPPED` count, `BLOCKED` count.

## Step 3 — Skip-without-regret detection

For each phase with `DONE_SKIPPED >= 50% of runs AND total runs >= MIN_RUNS`:

```bash
# Find runs where this phase was skipped
jq -c --arg cutoff "$cutoff" --arg p "$phase" \
  'select(.type == "phase_completed" and .ts > $cutoff
   and .payload.name == $p and .payload.status == "DONE_SKIPPED")
   | .run_id' \
  ~/.claude/work-graph.jsonl > /tmp/roi-skipped-runs.txt

# For each skipped run, check if any gap_logged event was tied to the skip
# Heuristic: gap_logged event in the same run_id, within next 30 minutes after the skip event
while read run_id; do
  jq -c --arg r "$run_id" \
    'select(.type == "gap_logged" and .run_id == $r)' \
    ~/.claude/work-graph.jsonl
done < /tmp/roi-skipped-runs.txt | wc -l > /tmp/roi-tied-gaps.txt
```

**Threshold:** Skipped ≥ 50% AND tied gaps ≤ 3 → fires as `recommendation: deprecate in next-major` (severity: low — non-urgent cleanup).

## Step 4 — Low-confidence pattern detection

For each phase:

```bash
# Median confidence of decisions logged in this phase
jq -r --arg cutoff "$cutoff" --arg p "$phase" \
  'select(.type == "decision_logged" and .ts > $cutoff and .payload.phase == $p)
   | .payload.confidence' \
  ~/.claude/work-graph.jsonl | sort -n > /tmp/roi-conf.txt

# Sample size check
n=$(wc -l < /tmp/roi-conf.txt)
[ "$n" -lt 5 ] && continue  # need ≥5 decisions to evaluate

median=$(awk 'NR==int((n+1)/2){print; exit}' n=$n /tmp/roi-conf.txt)

# Cross-reference: how many of these runs ended with pr_opened?
# (Decisions acted on by downstream phases)
acted_count=$(jq -c --arg p "$phase" --arg cutoff "$cutoff" \
  'select(.type == "decision_logged" and .ts > $cutoff and .payload.phase == $p)
   | .run_id' \
  ~/.claude/work-graph.jsonl | sort -u | while read r; do
    jq -c --arg r "$r" 'select(.type == "pr_opened" and .run_id == $r)' \
      ~/.claude/work-graph.jsonl
  done | wc -l)
```

**Threshold:** median confidence < 50% AND acted_count ≥ 5 → fires as `severity: med`.

**Reading rule:**
- Phase declared `haiku` or `sonnet` AND median confidence < 40% → reading: **model tier too low**; recommendation: **escalate model**.
- Phase declared `opus` AND median confidence < 40% → reading: **phase scope too broad**; recommendation: **narrow phase scope** or **revise prompt rules**.
- Phase declared anything AND 40% ≤ median < 50% → reading: **rules need updating**; recommendation: **revise prompt rules**.

## Step 5 — Orphan output detection

For each phase: parse the phase prompt for its declared output section (typically a heading like `## <Phase Name> Results` or whatever the agent writes back to forge-context.md).

Then grep all phase prompts for `## Context sections needed` blocks and see if the output section is referenced anywhere.

```bash
for phase in "$SKILL_BASE"/phases/*.md; do
  name=$(basename "$phase" .md)
  # Heuristic: find the most likely output section by looking for backticks
  # around "## XYZ Results" or "## XYZ Path" patterns
  outputs=$(grep -oE '`## [A-Za-z][A-Za-z .]+`' "$phase" | sort -u | tr -d '`')

  for output_section in $outputs; do
    # Skip generic sections shared across phases
    case "$output_section" in
      "## Brief"|"## Brief.Mode"|"## Phase Log"|"## Decision Log"|"## Run Stats") continue ;;
    esac

    # Count references in other phase prompts' "Context sections needed" blocks
    refs=$(grep -l "^- $output_section$" "$SKILL_BASE"/phases/*.md \
      | grep -v "$phase" | wc -l)

    if [ "$refs" -eq 0 ]; then
      echo "$name|$output_section|orphan"
    fi
  done
done > /tmp/roi-orphans.txt
```

**Threshold:** any orphan → fires as `severity: low` (non-urgent, but worth a CHANGELOG note).

## Step 6 — Compose inbox sections

Strip prior `## PhaseROI: ...` sections from the existing inbox. For each finding, emit one section per the **Output** spec. If nothing fired, emit `## PhaseROI: clean`.

## Step 7 — Status report

```
DONE — PhaseROI complete. <N> skip-without-regret, <M> low-confidence patterns, <K> orphan outputs.
```

If no phase has ≥ MIN_RUNS in the window:

```
DONE_NO_DATA — Largest phase has only <X> runs in <window>-day window (need <MIN_RUNS>). Re-run after more forge invocations.
```

## Why this is steward #4

Sentinel watches behavior, ConfigAuditor watches configuration, PhaseROI watches **cost**. Together the three cover the full audit surface of the crowd: what it does, how it's set up, and whether each piece earns its place.

ChangeGate (v2.4) is the fourth and final pre-v3 steward — it gates risky changes BEFORE they auto-apply, closing the loop on Phase 9's auto-apply default.
