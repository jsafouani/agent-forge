# BRIEF.md — Example (Greenfield)

> This is the format `/forge BRIEF.md` expects. The three required fields are `## Idea`, `## Users`, and `## Constraint`. `## Mode` defaults to `greenfield` if omitted. `## Tech preference` is optional — omit it to let the crowd decide.

---

## Mode

greenfield

## Idea

A small CLI that takes a directory of CSV files, infers a unified schema across them, and emits one normalized Parquet file with provenance columns (source filename + row number).

## Users

Data engineers doing one-off ingestion work who don't want to spin up dbt for a single batch. Sysadmins triaging data dumps from third parties before they hit the warehouse.

## Constraint

Must produce exactly one Parquet file regardless of input volume. Must include `_source_file` and `_source_row` columns. Cannot exceed 200MB of memory on a 10GB input (use a streaming reader). Must work on Windows and Linux without code changes.

## Tech preference

Python 3.11+ using `pyarrow` and `polars` (no pandas). Single-file CLI, no setup.py. `pip install` from PyPI.

---

# BRIEF.md — Example (Enhance Mode)

> Enhance mode modifies an existing repo surgically. Auto-detected from `cwd` for loose asks (`/forge "add X"` from inside a git repo). Explicit via `## Mode: enhance` for structured briefs. Requires `## Target Repo` as an absolute path.

---

## Mode

enhance

## Target Repo

/Users/you/projects/my-existing-app

## Idea

Add a `useDebounce` React hook with TypeScript types, replace the existing 250ms manual debounce inside `SearchBar.tsx` with the hook, and add a Vitest unit test that proves the timing.

## Users

The frontend team. The hook will be reused by the upcoming command palette and the filter sidebar.

## Constraint

Cannot change the public API of `SearchBar`. Hook must work with React 18 and React 19 (no `useEffect` cleanup bugs). Test must run in < 100ms (no real timers — use `vi.useFakeTimers()`).

---

## How `/forge` reads this

When you run `/forge BRIEF.md` from this directory, the orchestrator (Phase 1):

1. Validates the four required fields (`## Mode`, `## Target Repo`, `## Idea`, `## Users`, `## Constraint`).
2. Validates the Target Repo exists, is a git repo, and is readable.
3. Derives a slug from the Idea (lowercased, hyphenated, truncated to 30 chars).
4. Creates a worktree at `/tmp/forge-<slug>` branched off `origin/main` of the Target Repo.
5. Copies this `BRIEF.md` into the worktree root.
6. Writes `forge-context.md` (the shared state file each phase reads and appends to).
7. Skips Phase 2A and 2B (market research is moot in enhance mode — the substrate is fixed).
8. Continues with Phase 2C synthesis, Phase 3.5 codebase audit, Phase 3.6 baseline tests, and so on through the 16-phase loop.
9. Opens a PR against the Target Repo. The PR body includes the UAT checklist, baseline-test status, coverage delta, files-touched list, and Decision Log.

You see one progress line per phase. Nothing else.
