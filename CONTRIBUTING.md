# Contributing to agent-forge

Thanks for the interest. A few things worth knowing before you open a PR.

## What's easy to land

- **New phase prompts** — add a markdown file in `phases/`, declare its `## Recommended model:` and `## Context sections needed:` headers, and wire it into `SKILL.md`'s phase map. Phases that solve a real recurring rework problem are most welcome.
- **Smaller specialist skills** — anything that fits into one of the existing phase slots and produces better output than the current default.
- **Examples in `examples/`** — real BRIEF.md files with the resulting PR linked.
- **Documentation fixes** — typos, broken links, clarifications.

## What's deliberately hard to land

### Changes to the Stable Core

The Stable Core (`STABLE_CORE.md`) lists six skills the crowd cannot modify. **PRs that change a Stable Core skill require:**

1. A written rationale: why the existing skill is insufficient, with at least one concrete failure mode it can't handle.
2. A **champion-challenger benchmark**: run 5+ briefs through both the current Stable Core skill and the proposed replacement. Show the outputs side-by-side. Show the regression metrics (rework cycles, token cost, phase verification pass rate).
3. An updated `CONTRACT.md` reflecting any value shifts the new skill implies.
4. A migration plan for users running v(N-1).

This isn't gatekeeping — it's the same discipline the crowd applies to itself in Phase 9. The Stable Core protects against the crowd accidentally breaking its own infrastructure; PR review should hold to the same bar.

### Changes that bypass review or relax the verification commands

PRs that loosen `## Recommended model:` defaults, weaken evidence-based phase verification commands, or remove the worktree integrity check will be closed. These are load-bearing safety patterns. If you have a case for changing them, file an issue first and we'll discuss.

## How to develop locally

1. Clone the repo to `~/.claude/skills/workflow-agents-loop` (user-level) or to `<your-project>/.claude/skills/workflow-agents-loop` (project-level).
2. Make changes.
3. Run `/forge "test brief that exercises your change"` against a scratch repo.
4. Check the audit journal at `~/.claude/skills/.history.jsonl` to verify your change didn't trip Phase 9 unexpectedly.

## Code of conduct

Be useful. Be precise. The forge is a serious piece of meta-engineering, not a hackathon project — PRs and issues that respect that bar will get attention.
