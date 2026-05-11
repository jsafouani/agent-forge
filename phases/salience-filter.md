# Salience Filter (Phase: post-steward re-ranking)

## What this phase does

Reads `~/.claude/inbox.md` and **re-ranks every section by salience**, so the most actionable items surface to the top. Triggered automatically after every steward run (when invoked via triggers) or manually via `/forge inbox --reprioritize`.

Without this phase, the inbox grows in append-order — five stewards × N findings each = an unsorted blob. The salience filter is what makes the inbox **usable** as a daily attention queue, not an audit log.

## Context sections needed

(This phase reads `~/.claude/inbox.md` directly — no `forge-context.md` dependency.)

## Recommended model

haiku — scoring + sorting. The scoring formula is deterministic. No model judgment.

## Hard rules

- **Pure re-ranking.** Never edit a steward's section content, never merge sections, never delete findings. The salience filter only changes order and adds `[TIER]` prefixes.
- **Idempotent.** Re-running on an already-prioritized inbox produces the same output (tie-breaking is deterministic).
- **Time-box at 10 seconds.** Inbox is small; this should be near-instant.
- **Status:** `DONE` | `DONE_NO_OP` (inbox is empty or has only the `## ... clean` sections from stewards) | `BLOCKED` (inbox file unreadable).

## Step 1 — Read and parse the inbox

```bash
INBOX="$HOME/.claude/inbox.md"
test -f "$INBOX" || { echo "DONE_NO_OP — no inbox to filter"; exit 0; }
```

Split the inbox into sections. Each section starts with `## ` and continues until the next `## ` or end-of-file. The `# Forge Inbox` H1 header is preserved verbatim.

## Step 2 — Score each section

For each section, compute `salience = (severity_weight × confidence) ÷ (1 + days_since_observed)`.

**Severity weights** (extracted from each section's `severity:` line, defaulting to `low` if absent):

| Severity | Weight |
|---|---|
| high | 100 |
| med | 30 |
| low | 5 |

**Confidence** — extracted from the section if present (as a `confidence:` line or implicit from the steward's threshold rules). Default: 80 (mid-high; most steward findings are deterministic).

**Days since observed** — parse the section's most recent `evidence:` timestamp. If no timestamp, use 0 (treat as today).

Special cases:

- Sections from ChangeGate with `verdict: block` → force `salience = 999` (always top). Blocks are non-negotiable.
- Sections from ChangeGate with `verdict: gate-manual` → force `salience = max(salience, 200)` (always above any non-ChangeGate item).
- Sections matching `^## .*: clean$` → force `salience = -1` (always at bottom; informational only).
- Stale Core integrity findings → multiply salience by 3 (load-bearing files; treat as high priority regardless of declared severity).

## Step 3 — Assign tiers

Map salience to a printable tier label:

| Salience range | Tier label |
|---|---|
| ≥ 100 | `[HIGH]` |
| 20 – 99 | `[MED]` |
| 1 – 19 | `[LOW]` |
| ≤ 0 | (no label — informational) |

## Step 4 — Re-emit the inbox

```markdown
# Forge Inbox (re-prioritized <YYYY-MM-DD HH:MM:SS>)

> Sorted by salience = (severity_weight × confidence) ÷ (1 + days_since_observed).
> Special rules: ChangeGate `block` always top; `gate-manual` always above non-gate; Stable Core integrity × 3.

## [HIGH] <original-section-heading>
... (section body unchanged)

## [HIGH] <original-section-heading>
...

## [MED] <original-section-heading>
...

## [LOW] <original-section-heading>
...

## <clean section heading>
...
```

Write to a temp file first, then atomically replace `~/.claude/inbox.md`:

```bash
mv -f /tmp/inbox-reranked.md ~/.claude/inbox.md
```

## Step 5 — Status report

```
DONE — Salience filter complete. <N> sections re-ranked. <H> HIGH, <M> MED, <L> LOW, <C> informational.
```

If the inbox had only clean / no-op sections:

```
DONE_NO_OP — Inbox has no actionable items (all stewards reported clean).
```

## Integration

### Auto-invocation after every trigger

The cron-daily, git-post-commit, and file-watcher triggers all end with:

```
claude --prompt "/forge inbox --reprioritize" --no-interactive
```

This is a new arg form (added in v3.0 alongside this phase) that dispatches the salience-filter phase against the inbox and exits.

### Manual invocation

```
/forge inbox --reprioritize
```

The orchestrator recognizes the `--reprioritize` flag in the arg-form parser (Step 0 in SKILL.md) and dispatches this phase before printing the inbox to stdout.

## Why this is a phase (and not a steward)

Stewards write to the inbox. The salience filter reads the inbox and changes its presentation. That's a fundamentally different role — it doesn't generate findings, it organizes them. Putting it in `phases/` rather than `stewards/` reflects that:

- **Stewards** observe + recommend.
- **Phases** transform state.

The salience filter transforms the inbox state from append-order to priority-order.

## What if it gets the priority wrong?

The formula is deliberately simple. If users want different weights, edit this phase prompt directly. The simplicity also means the user can mentally re-rank if they disagree with the filter's choice for any specific item. Both Sentinel and ConfigAuditor's high-severity findings should always pop to the top; if they're not, that's a bug in the rule, not in the user's judgment.
