# BRIEF-v2.md — agent-forge v2.0 (Observe + StaleSkillReaper)

> The forge enhances itself. Feed this file to `/forge BRIEF-v2.md` from inside the `agent-forge` repo. The forge creates a worktree off `origin/main`, runs the 16-phase loop in enhance mode, and opens a PR against `agent-forge` itself for v2.0. The README already notes this kind of self-enhancement has happened before (the Stable Core pattern came out of one such run).

---

## Mode

enhance

## Target Repo

<absolute path to your local agent-forge clone>

> Replace this placeholder with the absolute path to your local `agent-forge` clone (e.g. `C:\Users\<you>\projects\agent-forge` on Windows or `~/projects/agent-forge` on Unix). The forge needs a writable local path because it creates a worktree off `origin/main` for the run.

## Idea

Add a persistent work graph and a steward framework to `agent-forge`. v2.0 ships exactly two pieces: (1) **Phase 11: Observe** appends structured JSONL events to `~/.claude/work-graph.jsonl` at the end of every `/forge` run; (2) **StaleSkillReaper** — a manually-triggered steward that mines the graph plus `~/.claude/skills/.history.jsonl` to identify skills unused for 60+ days, writing recommendations to `~/.claude/inbox.md`. Two new arg forms — `/forge inbox` (prints the inbox and exits) and `/forge steward <name>` (runs one steward and exits) — are handled inside `SKILL.md` before the BRIEF.md / inline-ask flow. This is the smallest shippable scope for the v2 architecture sketched in the README; the remaining 4 stewards (Sentinel, ConfigAuditor, PhaseROI, ChangeGate) follow in v2.1–v2.4.

## Users

Engineers running `/forge` more than a handful of times and noticing that each run starts amnesiac. The work graph is the first persistent memory across runs. StaleSkillReaper is the first practical use of it — it answers *"which skills earn their keep?"* Without the graph, that question requires manual `.history.jsonl` inspection.

Secondary users: contributors evaluating the project on GitHub. A persistent work graph and a working steward demonstrate that the "self-improvement system" claim in the README is not vaporware.

## Constraint

Must remain **markdown-only** — no Python, Node, or other runtime. `agent-forge` is a Claude Code skill, not an installed binary. All graph writes use shell append (`>> ~/.claude/work-graph.jsonl`); all reads use `grep` / `jq` one-liners embedded in phase prompts, executed by sub-agents via the `Bash` tool.

Must work on **Windows (Git Bash / WSL)** AND **Unix**. No PowerShell-only commands. If a command genuinely needs OS branching, the phase prompt detects via `uname -s` and forks.

The event schema must be **append-only and forward-compatible** — adding new event types must not break `StaleSkillReaper` reads. `/forge` runs that predate v2 must continue to work even when `~/.claude/work-graph.jsonl` does not yet exist (treat missing file as empty graph; first write creates it).

**Stable Core remains untouchable.** Phase 11 is a new phase, not a modification to an existing Stable Core phase. The steward framework is explicitly outside the Stable Core (so future contributors can extend it).

## Tech preference

- **Graph format:** plain JSONL. One event per line: `{"ts":"2026-05-11T18:42:03Z","type":"phase_completed","run_id":"forge-csv-cli-20260511","payload":{"phase":"4","name":"spec-writer","duration_s":47,"model":"opus"}}`.
- **Event types for v2.0:** `run_started`, `brief_filed`, `phase_completed`, `decision_logged`, `pr_opened`, `run_ended`, `subagent_dispatched`, `verification_failed`, `gap_logged`.
- **Schema validation:** none beyond `jq -e .` — every line must parse as valid JSON. Cheaper than a JSON schema and sufficient for an append-only log.
- **Inbox format:** plain markdown, one recommendation per `## ` section, with a `> evidence:` line pointing to graph event IDs. Triage glyphs (`a` / `r` / `s`) are documented but not interactive in v2.0 — the user edits the file by hand.
- **Arg-form parser in SKILL.md:** lightweight string match before existing brief-source resolution. `/forge inbox` → print file + exit. `/forge steward <name>` → load `stewards/<name>.md`, dispatch, exit. Anything else falls through to the existing flow.

## Acceptance criteria

The forge's own `## Decision Log` should reflect these as constraints upstream phases must honor, and the spec/plan must wire them into verification commands.

1. **New file** `phases/observe.md` declares `## Recommended model: haiku` and `## Context sections needed: Brief, Run Stats, Decision Log, Phase Log`. Dispatched as **Phase 11** after Phase 10 (Open PR).
2. **After a successful `/forge` run**, `~/.claude/work-graph.jsonl` has gained **at least 5 new events**: `run_started`, `brief_filed`, `phase_completed` (one or more), `pr_opened`, `run_ended`. Verification: `tail -50 ~/.claude/work-graph.jsonl | jq -c 'select(.run_id == "<this-run-slug>")' | wc -l` returns ≥ 5.
3. **New directory** `stewards/` with **new file** `stewards/stale-skill-reaper.md`. Declares its inputs (`~/.claude/work-graph.jsonl`, `~/.claude/skills/.history.jsonl`), threshold (60 days), and output path (`~/.claude/inbox.md`).
4. **Running `/forge steward stale-skill-reaper`** writes `~/.claude/inbox.md` with either `## No stale skills` (when nothing is stale) or one `## Stale skill: <name>` section per finding, each including a `> evidence:` line referencing the last 3 graph events for that skill.
5. **Running `/forge inbox`** prints `~/.claude/inbox.md` to stdout (or `Inbox empty.` if the file is missing) and exits with code 0. **No phases are dispatched.** Verification: dry-run produces no `subagent_dispatched` events in the graph.
6. **`SKILL.md` arg-form parser** handles the two new forms before the existing brief-source resolution. Test: `/forge inbox` after a fresh install must not error on missing files.
7. **`CONTRACT.md`** gains one new clause: *"The crowd may write to `~/.claude/work-graph.jsonl` (append-only) and `~/.claude/inbox.md` (full rewrite). Reads from these files are unrestricted. The graph schema is part of the Stable Core; new event types may be added but existing types may not be renamed or have their semantics changed."*
8. **`STABLE_CORE.md`** gains one new entry: *"The work-graph event schema (event types and required fields) — additive evolution only."* This protects the schema from accidental drift via Phase 9 Roster Review.
9. **`README.md`** gets a new top-level section *"v2: Persistent work graph"* immediately after *"The 16-phase loop"*, linking to `phases/observe.md` and `stewards/stale-skill-reaper.md`. Updates the phase-loop diagram to add `Phase 11   Observe                    (write work graph)`.
10. **`CHANGELOG.md`** gets a `## [2.0.0]` entry under `[Unreleased]` that summarizes Observe + StaleSkillReaper + the two new arg forms.
11. **All 16 existing phases unchanged in behavior.** Verification: every existing phase verification command listed in `SKILL.md` still passes on a smoke-test brief (a tiny greenfield run).
12. **Smoke test** added to `phases/smoke-test.md` (or a sidecar script) that exercises the v2 paths: invoke `/forge inbox` on a fresh install, invoke `/forge steward stale-skill-reaper` against an empty graph, run a tiny `/forge "build a hello-world CLI"` brief end-to-end, then re-invoke `/forge steward stale-skill-reaper` and assert at least one event was observed for the just-completed run.

## Out of scope (explicitly v2.1+)

- The remaining 4 stewards: `Sentinel`, `ConfigAuditor`, `PhaseROI`, `ChangeGate` — each follows in its own minor release.
- Cron / always-on / file-watcher triggering for stewards. v2.0 is manual-only.
- Salience filter and inbox prioritization. v2.0 lists everything; the user triages by hand.
- Graph DB migration (Neo4j / Memgraph / SQLite-with-CTEs). v2.0 is flat JSONL.
- Web UI / dashboard. CLI-only.
- Cross-machine sync of the graph. v2.0 is local-only.
- Inverted tasking (`"Sir, you have 3 stale PRs…"`). Requires more stewards before it's useful.

## Risks the forge should pre-mortem (Phase 3.7 equivalent — manual hint to the Spec Writer)

- **Schema drift** — if a new event type is added without updating StaleSkillReaper's query, the reaper silently misses cases. Mitigation: schema is in Stable Core (criterion 8 above); any new type must be added with a one-line note in `phases/observe.md`'s event catalog.
- **Privacy** — the graph captures run metadata across all projects on the user's machine. Make this explicit in README (no source code or brief content is logged — only metadata + decision summaries) and add a `~/.claude/work-graph.optout` sentinel: if it exists, Phase 11 is a no-op.
- **Bootstrap empty state** — first run on a fresh machine has no `.history.jsonl` and no graph. StaleSkillReaper must not error; it returns `## No stale skills` and exits 0.
- **Self-modification recursion** — this brief asks the forge to modify itself. The Stable Core protects the orchestrator, spec writer, plan writer, code execution, build+test gate, and code review. Phase 11 (Observe) is *new*, not a modification of those — so it's permitted. But the Decision Log should explicitly flag any moment a phase considers touching Stable Core files.

---

## How to run

From inside your local `agent-forge` clone:

```
/forge BRIEF-v2.md
```

The orchestrator detects enhance mode from `## Mode: enhance`, validates `## Target Repo`, creates a worktree under your platform's tmp directory (e.g. `C:\tmp\forge-<slug>` on Windows or `/tmp/forge-<slug>` on Unix) branched off `origin/main`, copies this brief in, and runs the 16-phase loop. The PR opens against the `agent-forge` repo's main branch when the loop completes.

Expected duration: 25–45 minutes wall clock (most of which is Phase 4 opus + Phase 6 parallel code execution).

Expected PR size: 8–12 files changed (1 new phase, 1 new steward, SKILL.md, CONTRACT.md, STABLE_CORE.md, README.md, CHANGELOG.md, smoke-test, plus the new `stewards/` directory).
