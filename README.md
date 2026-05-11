# agent-forge

> **A self-improving multi-agent workflow that turns a 3-field `BRIEF.md` into a PR. Runs on Claude Code.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code](https://img.shields.io/badge/runs%20on-Claude%20Code-blueviolet)](https://docs.claude.com/en/docs/claude-code)
[![Status](https://img.shields.io/badge/status-v5%20in%20development-green)](#roadmap)

---

You write three sentences describing what you want built. `agent-forge` runs a 16-phase directed graph of specialist AI agents that researches the market, audits the codebase, writes the spec, plans the work, ships the code, runs the tests, reviews the diff, and opens a PR. Then the crowd reviews **its own performance** and adjusts which agent handles which phase next time — except for the six skills in the Stable Core, which the crowd is structurally forbidden from touching.

`/forge "research the data governance dashboard"` → BRIEF.md synthesized → worktree branched → research, spec, plan, code, tests, review, PR. One progress line per phase. PR URL at the end.

---

## Why this exists

Most agent frameworks treat orchestration as a runtime concern: you wire up a graph, you run it, you hope. `agent-forge` treats orchestration as a **typed system with three first-class primitives** — a **Crowd Contract** that defines values and constraints, a **Stable Core** of skills the crowd cannot modify, and a **Roster Review** phase that promotes / demotes / hires / fires agents across runs.

The three primitives together give you something the popular frameworks don't: an agentic system that **improves over time** while remaining **safe to run unattended**.

---

## The three design primitives

### 1. Crowd Contract — typed values + constraints, gating self-modification

A markdown file (`CONTRACT.md`) that declares what the crowd values, what it refuses to do, and what it cannot change. **Phase 9 reads this file before approving any roster change.** If a proposed change drifts from the contract, the change is downgraded from auto-apply to propose-to-user.

```
The crowd exists to take a 3-field BRIEF.md and produce a PR with working code.
The crowd values: accuracy, simplicity, customer-grade polish, traceability.
The crowd refuses: shortcuts that bypass review, skill changes that break the loop.
The crowd hires when a domain is stretched, fires when a skill is consistently unused.
The crowd never modifies its own Stable Core.
```

The contract is **human-managed**. The crowd cannot rewrite its own values.

### 2. Stable Core — immutable skills, safety by construction

Six skills are protected from self-modification by Phase 9:

1. `workflow-agents-loop` — the orchestrator
2. `spec-writer` — Phase 4
3. `plan-writer` — Phase 5
4. `code-execution` — Phase 6
5. `build-test-gate` — Phase 7
6. `code-review` — Phase 8

Any proposed change targeting a Stable Core skill is **rejected before validation**. If the core itself needs updating, that's a human-driven release — the crowd does not self-modify load-bearing infrastructure. This is the same pattern Linux uses for kernel headers vs. userland: separation of stable foundation from churning surface.

### 3. Roster Review (Phase 9) — promotion, demotion, hiring, firing

After every PR ships, Phase 9 reviews the run's actual telemetry: which skills were dispatched, which produced acceptable outputs, which produced rework, which were unused. Then it proposes roster changes — promote a skill that handled a stretched domain well, demote one that consistently produced rework, hire a new specialist when a domain has been stretched repeatedly, fire one that hasn't fired in N runs.

Proposed changes pass through a **champion-challenger trial**, a **validation gate**, and an **independent reviewer** before auto-apply. Every change writes an entry to `~/.claude/skills/.history.jsonl`. **The audit journal is the source of truth, not the orchestrator's narrative.**

The result: the crowd that ships PR #50 is measurably different from the one that shipped PR #1 — sharper specialists, fewer rework cycles, with a complete audit trail of every change.

---

## The 16-phase loop

```
Phase 0    Brief Synthesis           (loose ask → fully-formed BRIEF.md)
Phase 1    Orchestrator              (mode detect, worktree, validation)
Phase 2A   Market Research          ─┐ parallel
Phase 2B   Tech Landscape           ─┘
Phase 2C   Intelligence Synthesis    (GO / NO-GO gate)
Phase 3    Skill Gap Audit           (does the crowd have what it needs?)
Phase 3.5  Codebase Audit            (enhance mode only)
Phase 3.6  Baseline Tests            (enhance mode only — capture green baseline)
Phase 4    Spec Writer               (load-bearing — opus by default)
Phase 5    Plan Writer
Phase 6    Code Execution            (parallel tier subagents)
Phase 7    Build + Test Gate         (3-cycle retry, baseline + coverage gates)
Phase 7.5  Smoke Test                (does the built artifact actually run?)
Phase 8    Code Review               (diff-scoped in enhance mode, OWASP + auth checks)
Phase 8.5  UAT Plan                  (enhance mode only)
Phase 9    Roster Review             (self-improvement — see above)
Phase 10   Open PR                   (mode-aware body)
───────────────────────────────────  ← end of the loop; PR is open
Phase 11   Observe                   (v2 — persists run as events in ~/.claude/work-graph.jsonl)
```

Two modes:
- **Greenfield (default)** — produces a new project from scratch.
- **Enhance** — modifies an existing repo surgically. Auto-detected from `cwd` in loose-ask flow (git repo → enhance). Adds Phase 3.5 / 3.6 / 8.5 sub-phases for codebase audit, baseline capture, and UAT planning.

Mode is auto-detected for loose asks, explicit (`## Mode: enhance`) when in a `BRIEF.md`.

---

## v2: Persistent work graph + stewards

v1's audit journal (`~/.claude/skills/.history.jsonl`) records roster changes within Phase 9. v2 makes the forge **persistent across runs** by introducing a global event graph and a framework of agents that read it.

### Phase 11 — Observe

After every `/forge` run, the new **Observe** phase distills the run into structured JSONL events and appends them to `~/.claude/work-graph.jsonl`. Events include `run_started`, `brief_filed`, `phase_completed`, `decision_logged`, `gap_logged`, `pr_opened`, `run_ended`. Only metadata is logged — never brief text, spec content, or source code. Users can opt out by creating `~/.claude/work-graph.optout` (Phase 11 then becomes a no-op).

The schema is part of the [Stable Core](STABLE_CORE.md) — additive evolution only, no breaking changes. New event types may be added; existing ones may not be renamed or have their semantics changed.

### Stewards

A **steward** is a manually- or cron-triggered agent that reads the work graph and writes recommendations to `~/.claude/inbox.md`. Stewards are explicitly **outside the Stable Core** — they are the crowd's primary expansion surface.

v2.0 ships **one** steward:

| Steward | Job | Trigger |
|---|---|---|
| `stale-skill-reaper` | Finds skills unused for 60+ days; recommends `fire` (recommend-only — never auto-removes) | `/forge steward stale-skill-reaper` |

The remaining four (`Sentinel`, `ConfigAuditor`, `PhaseROI`, `ChangeGate`) follow in v2.1–v2.4, each its own minor release. The roster mirrors a data-governance steward pattern that ports cleanly across domains — same framework primitives (event graph + sidecar agents + recommendation inbox), different node types in the graph.

### Two new arg forms in `/forge`

- **`/forge inbox`** — print the inbox file and exit. No phases dispatched.
- **`/forge steward <name>`** — run a single steward and exit. No worktree created.

Both arg forms are handled inside `SKILL.md` *before* brief-source resolution, so they always work — even on a fresh install with no `BRIEF.md` in the working directory.

### Why this matters

The v1 forge starts each run amnesiac. The work graph gives v2 the first persistent memory layer across runs. Once the graph exists, the stewards turn it into actionable intelligence: which skills earned their keep, which phases are silently failing, which decisions you keep overriding post-merge. **The graph is the foundation; everything else stacks on it.**

---

## Engineering decisions that matter

These are the things people ask about during interviews.

### Per-phase model tiering — token efficiency without quality loss

Each phase declares `## Recommended model: <haiku|sonnet|opus>` at the top of its prompt. The orchestrator routes accordingly:

| Phase | Default model | Rationale |
|-------|---------------|-----------|
| Intelligence Synthesis (2C) | `opus` | Cross-input reasoning + GO/NO-GO call. Quality is load-bearing. |
| Spec Writer (4) | `opus` | Spec quality determines everything downstream. |
| Code Review — security-critical (8) | `opus` | Auth, multi-tenant, secrets — opt-in via per-task hint. |
| Market Research (2A) | `sonnet` | Web research + summarization. |
| Code Execution (6) | `sonnet` | Default; tasks may opt in to `haiku` for copy work. |
| Build + Test Gate (7) | `haiku` | Mechanical: run command, parse output. |
| Open PR (10) | `haiku` | Mechanical PR body assembly. |

**The cheap path stays cheap; the load-bearing reasoning gets opus.** Per-task `model:` hints in the plan override the phase default — tier-by-task, not tier-by-phase.

### Evidence-based phase verification — no narrative-only "DONE"

After every phase that produces a file / diff / exit-code artifact, the orchestrator runs a deterministic shell check before recording the phase as complete:

| Phase | Verification command | Pass condition |
|-------|----------------------|----------------|
| Spec Writer | `wc -l <spec>` | ≥ 50 lines |
| Plan Writer | `grep -c "^### Task" <plan>` | ≥ 5 tasks |
| Code Execution | `git status --short \| wc -l` | ≥ 1 file changed |
| Build + Test Gate | `bash scripts/validate-build.sh` | exit 0 |
| Smoke Test | `## Smoke Test Status` in forge-context | not `FAIL` |
| Code Review | `test -f REVIEW_NOTES.md` | file exists |
| Open PR | `test -f PR_BODY.md` OR `gh pr view` | PR body or live PR exists |

These commands are **reproducible by hand from the worktree root** — the orchestrator runs them in the same shell after the phase agent reports DONE. **If a sub-agent says "DONE" but the artifact isn't there, the phase isn't done.**

### Auto-handoff at phase boundaries — work-in-flight protection (v5)

Long runs can degrade silently when the orchestrator approaches context exhaustion or hits a rate limit. After every phase, the orchestrator evaluates three signals (read from `## Run Stats`):

1. `phase_count_completed >= 6` — substantial work has accumulated
2. The phase that just completed appended **> 4096 bytes** to `forge-context.md`
3. `subagent_dispatch_count >= 5` — substantial tool-call cost incurred

When all three fire, the orchestrator writes a `HANDOFF.md` to the worktree (metadata, what's done, what's in flight, next 1-3 actions, key file paths, open decisions, resume command) and exits cleanly. **No errors. No retries. No "is this working?".** The user resumes with `/forge --resume <worktree>` whenever they're ready.

### Per-phase context filtering — 30–50% token reduction at equal quality

Each phase prompt declares `## Context sections needed`. The orchestrator filters `forge-context.md` before dispatch — only the listed sections are passed to the sub-agent. **A sub-agent that doesn't need market research doesn't receive it.** Compounds with model tiering: cheap phases also get small contexts.

### Decision-making protocol — ambiguity gets logged, not escalated

When a phase agent encounters ambiguity, it does NOT ask the user. It runs the **Decision-making protocol v4**:

1. Enumerate plausible options (typically 2–4).
2. Score each by probability of being correct, using this evidence stack in priority order:
   - The brief
   - Intelligence Summary
   - Spec + Plan
   - Codebase Audit (enhance mode)
   - Web research (Context7 MCP, Microsoft Learn MCP, WebFetch)
3. Pick the highest-probability option.
4. Log to `## Decision Log` in `forge-context.md`: chosen option, alternatives, evidence cited, confidence.
5. Proceed. No pause, no escalation.

`BLOCKED` is permitted only when ALL three conditions are true: all options scored < 30%, no research could disambiguate, required input is missing from both the brief AND forge-context. **The human reviews the Decision Log after the run, not during.**

### Sequential vs. parallel — explicit, not implicit

Phase 2A (Market Research) and Phase 2B (Tech Landscape) run in parallel — they read different inputs and produce different outputs. Phase 3.5 (Codebase Audit) and Phase 3.6 (Baseline Tests) used to run in parallel; we moved them to sequential after observing that Phase 3.6 agents that violated the read-only mandate (wrote stub source files) corrupted Phase 3.5's audit reads. **Sequential ordering costs ~30s of wall clock and gives the orchestrator a clean rollback target.**

### Worktree integrity checks — defense against misbehaving agents

After every read-only phase, the orchestrator runs `git status --short` and verifies only allowlisted files were touched. If a sub-agent violated its read-only mandate (created source files, edited code), the orchestrator:

1. `git checkout` discards modifications
2. `git clean -f` deletes unauthorized new files
3. Appends to `## Known Gaps / Blockers`: "subagent violated read-only mandate; reverted"
4. Continues — partial outputs are preserved if they were captured before the violation

---

## Quick start

### Prerequisites

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed
- Git (for worktree creation)
- A scratch directory (`/tmp` on Unix, `C:\tmp` on Windows)

### Install as a user-level skill

```bash
git clone https://github.com/jsafouani/agent-forge.git ~/.claude/skills/workflow-agents-loop
```

### Install as a project-level skill

```bash
cd <your-project>
git clone https://github.com/jsafouani/agent-forge.git .claude/skills/workflow-agents-loop
```

### Run

**Loose ask (greenfield):**
```
/forge "build a CLI that converts CSV to JSON with schema inference"
```

**Loose ask (enhance — auto-detected when cwd is a git repo):**
```
cd ~/projects/my-app
/forge "add a debounce hook to the search input"
```

**Structured brief:**
```
/forge BRIEF.md
```

See [examples/BRIEF-example.md](examples/BRIEF-example.md) for the 3-field BRIEF format.

---

## Comparison to other frameworks

| Capability | LangChain Agents | CrewAI | Microsoft Agent Framework | **agent-forge** |
|---|---|---|---|---|
| Multi-agent orchestration | ✅ | ✅ | ✅ | ✅ |
| **Self-modifying roster** | ❌ | ❌ | ❌ | ✅ (Phase 9) |
| **Stable Core protected from self-modification** | N/A | N/A | N/A | ✅ |
| **Crowd Contract gating self-changes** | ❌ | ❌ | ❌ | ✅ |
| **Audit journal for every roster change** | ❌ | ❌ | ❌ | ✅ |
| **Per-phase model tiering (haiku / sonnet / opus)** | Manual | Manual | Manual | ✅ Built-in |
| **Evidence-based phase verification** | ❌ | ❌ | ❌ | ✅ |
| **Decision Log with confidence scoring** | ❌ | ❌ | ❌ | ✅ |
| **Auto-handoff on context exhaustion** | ❌ | ❌ | ❌ | ✅ |
| **Persistent work graph across runs (v2)** | ❌ | ❌ | ❌ | ✅ |
| **Steward framework for cross-run intelligence (v2)** | ❌ | ❌ | ❌ | ✅ |
| Runs on Claude Code (no separate runtime) | ❌ | ❌ | ❌ | ✅ |

The "popular frameworks" column is uncharitable on purpose — those frameworks are great at the orchestration *primitive* and don't try to be opinionated about the self-improvement *system*. `agent-forge` makes the opposite tradeoff: it's locked to one runtime (Claude Code) in exchange for a much more opinionated take on what an agent crowd *should* do over time.

---

## Real-world usage

I built this to ship features on a multi-tenant AI governance product. The forge has shipped real PRs across multiple iteration phases (currently v5 in development). The Stable Core pattern was added after a v2 run where a self-modification proposal accidentally targeted the orchestrator itself — that incident is what taught the system the difference between human-managed and crowd-managed surfaces.

If you ship a PR with `agent-forge`, file an issue with the PR link. I'd like a public catalog of real outputs to seed Phase 2A market research with.

---

## Roadmap

- **v2.0 (current — in progress):** Phase 11 Observe persists each run as JSONL events. First steward (`stale-skill-reaper`) recommends pruning unused skills. New arg forms `/forge inbox` and `/forge steward <name>`. See [BRIEF-v2.md](BRIEF-v2.md) for the brief the forge runs against itself.
- **v2.1 – v2.4 (planned, one steward per minor release):** `Sentinel` (cross-run contract enforcement), `ConfigAuditor` (model-tier and Stable Core drift detection), `PhaseROI` (cost-vs-value pruning of phases), `ChangeGate` (generalized risk gating for Stable Core / auth / multi-tenant changes).
- **v3.0 (planned):** Always-on stewards via cron / file-watcher / git-hook triggers. Salience filter for inbox prioritization. Optional cross-machine sync of the work graph (so the same `~/.claude/work-graph.jsonl` follows you across laptop and desktop).
- **v4.0 (longer):** Pre-Mortem phase (Phase 3.7) between Synthesis and Spec — generates top failure modes the spec must explicitly address. Multi-repo enhance mode (changes that span 2+ git repos). Heuristic cost predictor that warns when a brief is likely to consume > $X in tokens before running.
- **v5.0 (longer):** Champion-challenger trials with statistical significance gating (currently a soft heuristic). Inverted tasking (the forge surfaces what *you* should act on next, not just responds to your asks).

---

## Repository layout

```
agent-forge/
├── README.md                  ← you are here
├── LICENSE                    ← MIT
├── SKILL.md                   ← the orchestrator (read this if you want to know exactly what happens)
├── CONTRACT.md                ← the Crowd Contract
├── STABLE_CORE.md             ← six immutable skills + work-graph event schema (v2)
├── CHANGELOG.md               ← release notes
├── forge-context-template.md  ← shared state file each run creates from this
├── phases/                    ← 17 phase prompts (16-phase loop + v2 Observe)
│   ├── brief-synthesis.md     ← Phase 0
│   ├── market-research.md     ← Phase 2A
│   ├── tech-landscape.md      ← Phase 2B
│   ├── intelligence-synthesis.md  ← Phase 2C (GO/NO-GO gate)
│   ├── skill-gap-audit.md     ← Phase 3
│   ├── codebase-audit.md      ← Phase 3.5
│   ├── baseline-tests.md      ← Phase 3.6
│   ├── spec-writer.md         ← Phase 4 (opus)
│   ├── plan-writer.md         ← Phase 5
│   ├── code-execution.md      ← Phase 6
│   ├── build-test-gate.md     ← Phase 7
│   ├── smoke-test.md          ← Phase 7.5
│   ├── code-review.md         ← Phase 8
│   ├── uat-plan.md            ← Phase 8.5
│   ├── roster-review.md       ← Phase 9 (self-improvement)
│   ├── open-pr.md             ← Phase 10
│   └── observe.md             ← Phase 11 (v2 — persists run to ~/.claude/work-graph.jsonl)
├── stewards/                  ← v2 — manually/cron-triggered agents reading the work graph
│   └── stale-skill-reaper.md  ← v2.0 — flags skills unused for 60+ days
├── templates/
│   ├── HANDOFF-template.md          ← auto-handoff format
│   ├── smoke-test-interactive-template.md
│   └── CLAUDE-lean-mode-template.md
└── examples/
    └── BRIEF-example.md       ← a complete sample brief
```

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). The short version: the Stable Core is intentionally hard to change — PRs that touch it require a written rationale and a champion-challenger benchmark. PRs that add a new phase or a new specialist skill are easier and more welcome.

---

## License

MIT — see [LICENSE](LICENSE).

---

## Author

**Jaouad Safouani** — Staff Engineer building production AI systems.

- LinkedIn: [linkedin.com/in/jaouadsafouani](https://www.linkedin.com/in/jaouadsafouani)
- GitHub: [github.com/jsafouani](https://github.com/jsafouani)

If you ship a PR with this, I want to hear about it. If you have ideas for what the next design primitive should be, open an issue.
