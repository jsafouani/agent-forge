# Changelog

All notable changes to `agent-forge` are recorded here. The project follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) and [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

_Nothing yet. Next: `ConfigAuditor` steward (v2.2.0)._

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
