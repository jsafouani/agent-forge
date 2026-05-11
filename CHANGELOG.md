# Changelog

All notable changes to `agent-forge` are recorded here. The project follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_Nothing yet. Next: v5.0 — Inverted tasking ("Sir, you should…" — forge tells YOU what to act on)._

## [4.0.0] — 2026-05-11 — Pre-Mortem (Phase 3.7) — failure-mode drafting before Spec

### Added

- **`phases/pre-mortem.md`** — new Phase 3.7. Runs in BOTH greenfield and enhance mode, between Phase 3.6 (Baseline Tests, enhance) / Phase 3 (Skill Gap Audit, greenfield) and Phase 4 (Spec Writer).
- **Fan-out architecture** — dispatches 5 sub-agents in parallel, one per failure-mode category. Each gets 30 s budget; total under 3 min wall clock. Sonnet tier.
- **Canonical 5-category coverage** — Security, Performance/scale, Correctness/edge case, Integration/contract drift, Operational/observability. Every brief gets all 5; no skipping by claiming a category doesn't apply (CLI tool's "auth bypass" becomes "argument injection / shell escape").
- **Each failure mode is concrete and falsifiable** — structured format with Probability, Impact, Trigger, Mechanism, Mitigation, If-Deferred clauses.

### Changed

- **`phases/spec-writer.md`** — gains a non-negotiable Pre-Mortem contract clause. For each of the 5 failure modes, the spec must EITHER mitigate (with literal `pre-mortem failure mode N/5` text the orchestrator greps for) OR defer via Decision Log entry.
- **`SKILL.md`** — adds Step 5.7 (Pre-Mortem dispatch) between Step 5.6 (Baseline Tests) and Step 6 (Spec Writer). Verification command: `grep -c "^### Failure mode" {{FORGE_CONTEXT_PATH}}` must equal exactly 5.
- **Phase 4 verification extended** — `grep -c "pre-mortem failure mode [1-5]/5" <spec> >= 5`. Orchestrator retries once; on second failure logs to Known Gaps but doesn't block the run (Phase 8 Code Review is the last line of defense).
- **Phase map in SKILL.md** adds Phase 3.7 to the loop diagram.

### Architectural significance

This is **Layer 3** of the Jarvis vector — the *"I've run 6 million scenarios"* anticipation quality, except deterministic and explicit instead of opaque LLM hand-wave. Combined with v2 (observability) and v3.0 (ambient triggers), the forge now:

- **Observes** itself (v2 fleet)
- **Reacts** to its environment (v3.0 triggers)
- **Anticipates** its own failure modes (v4.0 pre-mortem)

The final layer (v5.0 inverted tasking) gives the forge the *"Sir, you should…"* quality — it surfaces what YOU should act on next, not what to build.

## [3.0.0] — 2026-05-11 — Always-on stewards + salience filter

### Added

- **`triggers/` directory** — three documented trigger configurations that turn the v2 stewards from manual `/forge steward X` into ambient, signal-driven runs:
  - **`triggers/cron-daily.md`** — full 5-steward sweep at 06:00 local time. Linux/macOS crontab + Windows Task Scheduler PowerShell. Pause via `~/.claude/triggers.pause` sentinel.
  - **`triggers/git-post-commit.md`** — runs ConfigAuditor + Sentinel after every commit (the two stewards that benefit most from fresh signal). Backgrounded; doesn't block the commit. Per-repo opt-out via `.forge-no-post-commit` sentinel.
  - **`triggers/file-watcher.md`** — runs StaleSkillReaper when `~/.claude/skills/` mutates. fswatch (macOS) / inotifywait (Linux) / Watchman (Windows). 30-second debounce.
- **`triggers/README.md`** — overview, opt-out instructions, philosophy (markdown-only; explicit user installation; no hidden daemons).
- **`phases/salience-filter.md`** — new out-of-loop phase. Re-ranks `~/.claude/inbox.md` by `salience = (severity_weight × confidence) ÷ (1 + days_since_observed)` with special rules (ChangeGate blocks → always top; Stable Core integrity × 3 multiplier; clean-section findings → always bottom).
- **`/forge inbox --reprioritize`** — new arg form. Dispatches the salience filter and prints the re-sorted inbox.

### Changed

- `SKILL.md` Step 0 (arg-form dispatch) gains the `--reprioritize` flag handling.
- Phase map in `SKILL.md` adds the salience filter as an out-of-loop entry.

### Architectural significance

This is the **qualitative tipping point** in the Jarvis vector. Before v3.0, the forge waits to be invoked. After v3.0, the forge is **ambient** — stewards run on time (cron), git activity (post-commit), and roster mutations (file watcher), and the inbox stays sorted by attention priority without the user thinking about it. v4.0 (pre-mortem) and v5.0 (inverted tasking) stack on this foundation.

## [2.4.0] — 2026-05-11 — ChangeGate (generalized risk gating)

### Added

- **`stewards/change-gate.md`** — fifth steward. Gates risky changes BEFORE Phase 9 auto-applies them. Closes the v2 four-steward observability fleet.
- **Dual-mode operation:**
  - **Hook mode** — invoked inline by Phase 9 before auto-apply. Returns `auto-apply` | `gate-manual` | `block`. Time-boxed at 20 s; on timeout defaults to `gate-manual` for safety.
  - **Standalone mode** — `/forge steward change-gate` runs a 7-day weekly summary into the inbox.
- **User-configurable policy file** — `~/.claude/change-gate.policy.md`. If absent, a sensible default applies: blocks Stable Core targets; gate-manuals auth/security paths, multi-tenant identifiers, secret material, > 500 LOC changes, runs with low-confidence forced-through decisions.
- **SKILL.md integration:** Step 11 (Roster Review) now consults ChangeGate before auto-apply and records the verdict inline in `## Roster Changes Proposed`.

### Changed

- Phase 9 auto-apply path is no longer the only auto-apply path — it now goes through ChangeGate first. Existing behavior preserved for users without a custom policy (default policy is permissive for non-risky changes).

### Drift coverage now complete (v2 fleet)

| Steward | Watches | Release |
|---|---|---|
| Sentinel | What the system does | v2.1.0 |
| ConfigAuditor | How it's configured | v2.2.0 |
| PhaseROI | Whether it earns its cost | v2.3.0 |
| ChangeGate | Whether the next change is safe | v2.4.0 ← this release |

v3.0 (next) lifts these from manual triggers to always-on.

## [2.3.0] — 2026-05-11 — PhaseROI (cost-vs-value calibration)

### Added

- **`stewards/phase-roi.md`** — fourth steward. Calibrates each phase's contribution to outcomes across three economic axes:
  - **Skip-without-regret detector** — flags phases skipped in ≥ 50% of runs with ≤ 3 tied downstream gaps. Deprecation candidates.
  - **Low-confidence pattern** — flags phases whose decisions median < 50% confidence AND downstream PRs nonetheless acted on them. Reading rule branches by declared model tier (escalate vs. narrow scope vs. revise rules).
  - **Orphan output detector** — flags phases that write a forge-context section no downstream phase declares in its `## Context sections needed`.
- Default lookback: 60 days (`ROI_WINDOW_DAYS`). Longer than other stewards — ROI signal accumulates slowly.
- Minimum sample: 15 runs per phase to evaluate; suppresses noise on rarely-used phases.
- Drift-axis coverage now complete: **what** (Sentinel), **how** (ConfigAuditor), **cost** (PhaseROI).

## [2.2.0] — 2026-05-11 — ConfigAuditor (model-tier + Stable Core drift)

### Added

- **`stewards/config-auditor.md`** — third steward. Audits the gap between *declared* configuration and *actual* runtime behavior. Three drift detectors:
  - **Model-tier drift** — phases whose declared `## Recommended model` doesn't match the model actually used in ≥ 60% of runs (over a 30-day window, min 5 runs to suppress noise).
  - **Stable Core integrity** — verifies all six immutable skill files exist, frontmatter `name:` fields match conventions, and byte counts don't decrease by > 25% (catches truncation / accidental edits).
  - **Verification dormancy** — phases whose declared verification command appears to never have fired (≥ 20 runs, 0 `verification_failed` events) — suggests the orchestrator is skipping the check.
- Default lookback: 30 days (`AUDITOR_WINDOW_DAYS` env var). Baseline byte counts stored at `~/.claude/auditor-baseline.txt`.
- Drift-axis coverage: **what the system does** (Sentinel) + **how it's configured** (ConfigAuditor). Cost calibration (PhaseROI) lands in v2.3.

## [2.1.0] — 2026-05-11 — Sentinel (cross-run contract enforcement)

### Added

- **`stewards/sentinel.md`** — second steward. Aggregates cross-run signal that no single `/forge` run can see: read-only mandate violations (Phase 3.5/3.6), verification failures in load-bearing phases (4/5/7/7.5/8), Phase 6 forced-through low-confidence decisions, Stable Core target attempts, and brief drift (≥ 3 gaps per run).
- Default lookback window: 30 days (configurable via `SENTINEL_WINDOW_DAYS` env var).
- Recommend-only. Never modifies the work graph. Idempotent.
- Five pattern detectors (A–E) each with explicit jq filters and threshold rules — fully deterministic, runs on haiku.
- Inbox section ownership: Sentinel owns `## Sentinel: ...` sections; other stewards' sections are preserved on merge.

### Changed

- README and Roadmap reference Sentinel as shipped; next milestone is v2.2 (ConfigAuditor).

## [2.0.0] — 2026-05-11 — Persistent work graph + first steward

### Added

- **Phase 11 — Observe.** New phase, runs after Phase 10 (Open PR) at the end of every `/forge` run. Distills the run into structured JSONL events and appends them to `~/.claude/work-graph.jsonl`. Event types: `run_started`, `brief_filed`, `phase_completed`, `decision_logged`, `gap_logged`, `pr_opened`, `run_ended`, `subagent_dispatched`, `verification_failed`. Privacy-preserving — logs metadata and hashes only, never brief text or source code. Opt-out via `~/.claude/work-graph.optout`. Best-effort: failure does not retroactively fail the run.
- **Steward framework.** New top-level directory `stewards/` for manually-triggered agents that read the work graph and write recommendations to `~/.claude/inbox.md`. Stewards are explicitly outside the Stable Core — they are the crowd's primary expansion surface.
- **`stewards/stale-skill-reaper.md`** — first steward. Reads the work graph + `~/.claude/skills/.history.jsonl`, identifies skills unused for 60+ days (configurable via `STALE_DAYS` env var), emits `## Stale skill: <name>` recommendations to the inbox with evidence chains. Stable Core skills are exempt and flagged separately if they're never observed. Recommend-only — never auto-removes.
- **`/forge inbox`** — new arg form. Prints `~/.claude/inbox.md` to stdout (or `Inbox empty.` if absent) and exits without dispatching phases or writing to the work graph.
- **`/forge steward <name>`** — new arg form. Resolves `stewards/<name>.md`, dispatches a single agent with that prompt, prints the agent's final status, exits with its code. No worktree is created.
- **Work-graph event schema added to the Stable Core.** Top-level keys and v2.0 event types are now protected from breaking changes by `STABLE_CORE.md`. Additive evolution (new event types) remains permitted.
- **Crowd Contract clause** permitting writes to `~/.claude/work-graph.jsonl` (append-only) and `~/.claude/inbox.md` (rewrite). Reads are unrestricted.
- **`BRIEF-v2.md`** — the brief the forge runs against itself to build v2.0. Recursive self-enhancement, validated as a 17-section enhance-mode brief.

### Changed

- **`SKILL.md` orchestrator gains Step 0** — arg-form dispatch runs before brief resolution, so `/forge inbox` and `/forge steward <name>` work even on fresh installs with no `BRIEF.md`.
- **`SKILL.md` Step 12 → Step 13** — Phase 10 (Open PR) now prints an intermediate progress line; the final `✓ /forge complete` line moves to a new Step 13 that runs after Phase 11 (Observe). The final line surfaces the event count.
- **`STABLE_CORE.md`** restructured — now distinguishes "immutable skills" (six) from "immutable schemas" (work-graph) and "outside the Stable Core" (stewards, non-core phases, templates, contract).
- **Phase map in `SKILL.md`** adds Phase 11 below Phase 10 with a `(NEW v2.0)` marker.
- **README.md** adds a v2-specific section (`v2: Persistent work graph + stewards`), two new rows in the framework comparison table, an updated repository layout, and a restructured roadmap (v2.0–v5.0 public-release numbering replaces the prior v5/v6/v7 internal milestone labels).

### Internal release-numbering policy

This is the first time the project uses public SemVer (v2.0 is the second public release). The prior internal milestones (initial 11-phase forge, enhance mode, evidence-based verification + tiering, auto-handoff) are documented below under "Pre-public internal milestones" for completeness but are not addressable via SemVer.

## [1.0.0] — 2026-05-11 — Initial public release

### Added

- Initial public release of `agent-forge` to https://github.com/jsafouani/agent-forge.
- `README.md` (hero document — three design primitives, 16-phase loop, comparison table, real-world usage).
- `LICENSE` (MIT).
- `CONTRIBUTING.md` — guidance for new phases, new specialists, examples; Stable Core change protocol.
- `SKILL.md` — Forge Orchestrator (16-phase directed graph, autonomous-mode rules, evidence-based verification, model tiering, context filtering, auto-handoff, decision-making protocol).
- `CONTRACT.md` — Crowd Contract (values, refusals, hire/fire rules, Stable Core protection).
- `STABLE_CORE.md` — six immutable skills.
- `forge-context-template.md` — shared state file each run instantiates.
- 16 phase prompts under `phases/`:
  - Phase 0 — Brief Synthesis
  - Phase 2A — Market Research
  - Phase 2B — Tech Landscape
  - Phase 2C — Intelligence Synthesis (GO/NO-GO)
  - Phase 3 — Skill Gap Audit
  - Phase 3.5 — Codebase Audit (enhance mode)
  - Phase 3.6 — Baseline Tests (enhance mode)
  - Phase 4 — Spec Writer (opus default)
  - Phase 5 — Plan Writer
  - Phase 6 — Code Execution (parallel tier subagents)
  - Phase 7 — Build + Test Gate (3-cycle retry, baseline + coverage gates)
  - Phase 7.5 — Smoke Test
  - Phase 8 — Code Review (diff-scoped in enhance mode)
  - Phase 8.5 — UAT Plan (enhance mode)
  - Phase 9 — Roster Review (champion-challenger trials, validation gate, audit journal)
  - Phase 10 — Open PR
- Three templates under `templates/`: `HANDOFF-template.md`, `smoke-test-interactive-template.md`, `CLAUDE-lean-mode-template.md`.
- `examples/BRIEF-example.md` — greenfield + enhance mode sample brief.
- `.gitignore` — runtime artifacts excluded (`forge-context.md`, `HANDOFF.md`, etc.).

## Pre-public internal milestones

The features below were developed across iterative internal milestones before the v1.0.0 public release. Listed for historical context — not addressable via public SemVer tags.

### Auto-handoff at phase boundaries (internal milestone)

- **Auto-handoff at phase boundaries.** After every phase, the orchestrator evaluates three signals (`phase_count_completed >= 6`, single-phase append > 4096 bytes, `subagent_dispatch_count >= 5`). When all three fire, the orchestrator writes `HANDOFF.md` and exits cleanly. Resume with `/forge --resume <worktree>`.
- `--resume <worktree-path>` flag on the orchestrator.
- `## Run Stats` section in `forge-context.md` for handoff signals.
- Sequential ordering of Phase 3.5 (Codebase Audit) and Phase 3.6 (Baseline Tests) — earlier versions ran them in parallel; in practice, a Phase 3.6 agent that violated the read-only mandate corrupted Phase 3.5 audit reads.

### Verification + tiering (internal milestone)

- **Evidence-based phase verification.** Deterministic shell check after every phase that produces an artifact. Narrative-only "DONE" is no longer trusted.
- **Per-phase model tiering.** Each phase declares `## Recommended model: <haiku|sonnet|opus>`. Per-task hints override the phase default.
- **Per-phase context filtering.** Each phase declares `## Context sections needed`. The orchestrator filters `forge-context.md` before dispatch — 30-50% token reduction at equal quality.
- Phase 7.5 — Smoke Test. Verifies the built artifact actually runs.

### Enhance mode (internal milestone)

- **Enhance mode.** Modifies an existing repo surgically. Adds Phase 3.5 (Codebase Audit), Phase 3.6 (Baseline Tests), Phase 8.5 (UAT Plan).
- Auto-detect of enhance mode for loose-ask invocations from inside a git repo.
- Worktree integrity checks after every read-only phase.
- Stable Core list formalized after a self-modification proposal accidentally targeted the orchestrator itself.

### Initial 11-phase forge (internal milestone)

- 11-phase directed graph: Orchestrator, Market Research, Tech Landscape, Intelligence Synthesis (GO/NO-GO), Skill Gap Audit, Spec Writer, Plan Writer, Code Execution, Build + Test Gate, Code Review, Roster Review, Open PR.
- Crowd Contract pattern.
- Phase 9 Roster Review with champion-challenger trials.
