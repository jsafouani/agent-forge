# Stable Core — Immutable Skills + Schemas

These items cannot be fired, modified, or renamed by Phase 9 Roster Review. Any proposal that targets a Stable Core item is rejected before validation.

## Immutable skills

1. `workflow-agents-loop` — the orchestrator (this skill)
2. `spec-writer` — Phase 4
3. `plan-writer` — Phase 5
4. `code-execution` — Phase 6
5. `build-test-gate` — Phase 7
6. `code-review` — Phase 8

## Immutable schemas (v2.0)

7. **Work-graph event schema** (`~/.claude/work-graph.jsonl`) — additive evolution only.

   - **Top-level keys** (every event line): `ts`, `type`, `run_id`, `payload`. These four keys may not be renamed, removed, or have their data types changed.
   - **v2.0 event types** (catalogued in `phases/observe.md`): `run_started`, `brief_filed`, `phase_completed`, `decision_logged`, `gap_logged`, `pr_opened`, `run_ended`, `subagent_dispatched`, `verification_failed`. These types may not be renamed, removed, or have their `payload` fields renamed.
   - **New event types may be added freely.** Future stewards / phases / contributors may extend the catalog; backward compatibility is required (existing stewards that ignore unknown types must still function).

   Why this is in the Stable Core: every steward reads the graph. A breaking schema change silently invalidates every steward. The schema must evolve like a wire protocol, not like application code.

## Outside the Stable Core (the crowd's expansion surface)

- **Stewards** (files under `stewards/`) — add, remove, or modify freely. v2.0 ships `stale-skill-reaper.md`; v2.1+ adds Sentinel, ConfigAuditor, PhaseROI, ChangeGate.
- **Phases beyond the core six** (Phase 2A, 2B, 2C, 3, 3.5, 3.6, 7.5, 8.5, 9, 10, 11) — modifiable by Phase 9 with the usual safety guards.
- **Templates** under `templates/`.
- **The Crowd Contract** (`CONTRACT.md`) — human-managed but not Stable Core; the user can edit it without a release cycle.

If the Stable Core itself needs updating, that is a human-driven release. The crowd does not self-modify load-bearing infrastructure or load-bearing protocols.

To update this list: edit this file directly.
