# /forge HANDOFF — auto-generated at phase boundary

> This file is written by the /forge orchestrator (v5+) when a long run crosses the phase-boundary heuristic and exits cleanly to protect work-in-flight from context exhaustion. Resume with: `/forge --resume <worktree>`.

## 1. Metadata

- **Date:** {{DATE}}
- **Slug:** {{SLUG}}
- **Worktree:** {{WORKTREE_PATH}}
- **Branch:** forge/{{SLUG}}
- **Target Repo:** {{TARGET_REPO}} (or `_n/a (greenfield)_`)
- **Originating brief:** {{BRIEF_PATH}}
- **Mode:** {{MODE}}
- **Trigger reason:** all 3 boundary signals fired (`phase_count_completed >= 6`, last phase output > 4096 bytes appended, `subagent_dispatch_count >= 5`)

## 2. What's been done

> Mirror of the `## Phase Log` section in forge-context.md as of the boundary check. Each line is one phase completion or skip. Verbatim — don't paraphrase.

- {{PHASE_LOG_LINES}}

## 3. What's in flight

> The phase that was about to start (or had just started) when the boundary check fired. Include any partial outputs already written to forge-context.md. If nothing was in flight (boundary check fired immediately after a clean phase write), say so.

- **Phase in flight:** {{PHASE_IN_FLIGHT}}
- **Partial outputs:** {{PARTIAL_OUTPUTS}} (or `_none_`)

## 4. Next 1-3 actions

> Pending phases in order. The first one is the resume target — `/forge --resume` jumps straight to it.

1. {{NEXT_PHASE_1}}
2. {{NEXT_PHASE_2}} (or `_none_`)
3. {{NEXT_PHASE_3}} (or `_none_`)

## 5. Key file paths

> Absolute paths. The resuming session uses these to recover state without re-deriving.

- forge-context.md: `{{WORKTREE_PATH}}/forge-context.md`
- Spec: `{{SPEC_PATH}}` (or `_pending_` if Phase 4 not yet run)
- Plan: `{{PLAN_PATH}}` (or `_pending_` if Phase 5 not yet run)
- Brief: `{{WORKTREE_PATH}}/BRIEF.md`
- Validation script: `{{WORKTREE_PATH}}/scripts/validate-build.sh` (enhance mode)

## 6. Open decisions

> Carry-forward entries from `## Decision Log` in forge-context.md that need attention from the resuming session (e.g., a decision logged at `confidence: 30%` that should be revisited). Most decisions don't need carry-forward — only flag ones with low confidence or downstream impact.

- {{OPEN_DECISIONS}} (or `_none_`)

## 7. Resume command

```
/forge --resume {{WORKTREE_PATH}}
```

The resuming session will:
1. Read this HANDOFF.md and validate the 7-section structure.
2. SKIP BRIEF.md validation (the brief is already embedded in forge-context.md).
3. Read forge-context.md to recover all upstream phase outputs.
4. Jump directly to the phase listed in section 4.1 above.
5. Continue the loop normally from there.
