# Inverted Tasking (Phase: attention queue)

## What this phase does

Inverts the agent–user relationship. The default forge model: *user files a brief → forge produces a PR.* Inverted Tasking flips it: **the forge produces an attention queue → user acts on it**.

This is the forge as **presence** — proactive about your time. Where v2.0–v4.0 made the forge observant, ambient, and anticipatory, **v5.0 makes the forge an attention director**.

Output: `~/.claude/attention.md` — a ranked queue of things YOU should act on, with evidence and a recommended next action for each. The forge tells you *"Sir, you should…"* — not *"Sir, what should I build?"*.

## Context sections needed

This phase reads from **outside** the per-run `forge-context.md`. Its inputs are global:

- `~/.claude/work-graph.jsonl` — full event history.
- `~/.claude/inbox.md` — outstanding steward recommendations.
- `~/.claude/skills/.history.jsonl` — Phase 9 audit journal.
- The user's git history (last 30 days across repos in `~/projects/` and `~/work/` — configurable via `INVERTED_TASKING_REPO_ROOTS` env var).
- The user's GitHub PRs / reviews (via `gh pr list --author "@me"` and `gh pr list --review-requested "@me"`).

## Recommended model

sonnet — composing the ranked queue requires synthesizing across multiple sources and producing natural prose. Haiku is too narrow. Opus is overkill.

## Hard rules

- **Never auto-act.** Inverted tasking surfaces; it does not execute. Even if a queue item says *"PR #42 has no merge conflicts and 2 LGTM reviews — merge it"*, the agent does not merge.
- **Always cite evidence.** Every queue item has a `> evidence:` line pointing to the graph events / PR URLs / file paths that justify its rank.
- **Time-box items.** Each queue item includes an estimated effort label: `<5min`, `5-30min`, `>30min`. User decides where to spend.
- **Idempotent within a 1-hour window.** Re-running within an hour produces near-identical output (signals don't change that fast).
- **Time-box at 90 seconds total wall clock.** Multi-source aggregation can be slow; cap at 90s and use whatever sources resolved.
- **Status:** `DONE` (queue written) | `DONE_NO_SIGNAL` (no items meet threshold; queue is empty) | `BLOCKED` (cannot read graph or write attention file).

## Categories of attention item

The phase scans for items in these categories, in this priority order:

### Category 1 — Stalled own PRs

PRs the user authored that have been open > 3 days without their own activity.

```bash
gh pr list --author "@me" --json number,title,url,updatedAt,state \
  --jq '.[] | select(.state == "OPEN" and (now - (.updatedAt | fromdateiso8601)) > 259200)'
```

Recommended action: *"Reply to last review comment or close as stale."* Effort: `5-30min`.

### Category 2 — Reviews requested of you

```bash
gh pr list --review-requested "@me" --json number,title,url,createdAt,repository
```

Recommended action: *"Review PR — author is blocked."* Effort: by PR size; haiku-estimate from `additions + deletions` count.

### Category 3 — Forced-through low-confidence decisions

From the work graph: `decision_logged` events with `confidence < 30` whose `run_id` also has a `pr_opened` event. The forge proceeded on weak signal AND shipped. The user should review whether the shipped code reflects the right call.

```bash
jq -c 'select(.type == "decision_logged" and .payload.confidence < 30)' \
  ~/.claude/work-graph.jsonl | head -10
```

Recommended action: *"Review the PR for this run — Phase 6 acted on a 22%-confidence decision. Verify the outcome matches intent."* Effort: `5-30min`.

### Category 4 — Stable Core integrity findings

Any `## ConfigAuditor: Stable Core integrity` section in the inbox is, by definition, urgent — the forge's load-bearing files may have drifted.

Recommended action: *"Inspect the named Stable Core file immediately. Diff against the v1.0.0 tagged version."* Effort: `<5min` (just look).

### Category 5 — Pending ChangeGate approvals

Any `## ChangeGate: pending approval` section in the inbox needs the user to approve or reject.

Recommended action: *"Approve or reject the gated change. Worktree at [path]."* Effort: by LOC.

### Category 6 — Skills with stale Roster Review decisions

From `~/.claude/skills/.history.jsonl`: roster proposals that landed in trial mode > 7 days ago and haven't been promoted or fired.

Recommended action: *"Promote or fire trial skill [name] — 7 days in trial is stale."* Effort: `<5min`.

### Category 7 — Patterns Sentinel has flagged repeatedly

From the inbox: `## Sentinel: <pattern>` sections that have appeared in ≥ 2 consecutive daily-cron runs (compare timestamps in the prior `attention.md` if it exists).

Recommended action: *"Sentinel has flagged [pattern] for [N] consecutive days. Either fix the underlying cause or suppress the rule."* Effort: depends.

## Step 1 — Gather signals from all 7 categories in parallel

Dispatch 7 sub-agents (one per category), each with the specific data-gathering query above. Each gets 15 s budget. Aggregate at 90 s wall clock.

## Step 2 — Compute attention score per item

```
attention_score = (category_weight × urgency) ÷ (1 + cost_to_act_hours)
```

| Category | Weight |
|---|---|
| 4. Stable Core integrity | 100 |
| 5. ChangeGate pending approvals | 80 |
| 2. Reviews requested of you | 60 |
| 1. Stalled own PRs | 40 |
| 3. Forced-through low-confidence | 30 |
| 6. Stale roster trials | 20 |
| 7. Repeated Sentinel patterns | 10 |

Urgency = 1 + (days_since_first_observed / 7), capped at 5. Items become more urgent over time.

Cost to act = effort label converted to hours (`<5min` = 0.08, `5-30min` = 0.25, `>30min` = 0.5+).

## Step 3 — Emit ranked queue to `~/.claude/attention.md`

```markdown
# Attention queue (generated <YYYY-MM-DD HH:MM:SS>)

> The forge's recommendation for what you should act on next. Ranked by attention_score.
> Re-generate with: `/forge attend`

## 1. <Title> [<effort>]

<one-sentence why this matters>

- recommended action: <verb-first instruction>
- evidence:
  - <link or path>
  - <link or path>
- score: <number>

## 2. <Title> [<effort>]

...
```

Cap at top 10 items. If fewer than 10 meet a minimum score threshold (≥ 5), emit only those that qualify. If none qualify:

```markdown
# Attention queue (generated <YYYY-MM-DD HH:MM:SS>)

## Inbox zero

Nothing in any of the 7 attention categories meets the threshold today. Build something.
```

## Step 4 — Status report

```
DONE — Inverted tasking complete. <N> items in queue (top score: <score>). Written to ~/.claude/attention.md.
```

If no items met threshold:

```
DONE_NO_SIGNAL — Inbox zero. Nothing urgent in any of the 7 attention categories.
```

## New arg form: `/forge attend`

The orchestrator's Step 0 arg-form parser (v3.0+) gains a fourth arg form in v5.0:

```
/forge attend → dispatch the inverted-tasking phase, then print ~/.claude/attention.md
```

Daily cron trigger (`triggers/cron-daily.md`) is updated in v5.0 to also invoke `/forge attend` after the salience filter — so your morning inbox is paired with a morning attention queue.

## Why this is the final layer

| Layer | Quality |
|---|---|
| Layer 2 (v2.0) | The forge remembers across runs |
| Observability fleet (v2.0–v2.4) | The forge audits itself |
| Layer 1 (v3.0) | The forge reacts ambient |
| Layer 3 (v4.0) | The forge anticipates failure |
| **Inverted tasking (v5.0)** | **The forge directs your attention** |

Together they constitute the forge as a **presence**. The forge is no longer a tool you invoke — it knows what you've done, what you're doing, what's likely to go wrong, and what you should do next. The hand-off from "agent for engineering tasks" to "ambient engineering partner" is complete.

## What this is NOT

- Not an auto-actor. The forge surfaces; you decide.
- Not a replacement for your judgment. Items are ranked but you re-rank by reading.
- Not perfect. The forge is wrong sometimes — about which PR matters most, about which Sentinel pattern is real signal vs. noise. Treat the queue as a high-quality first draft, not a verdict.
- Not the end. v6.0+ explorations (multi-machine sync, cross-user graph aggregation, pre-mortem simulation with quantitative cost predictions) are still ahead — but v5.0 closes the core forge arc.
