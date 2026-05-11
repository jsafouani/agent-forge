# Smoke-Test Interactive Surfaces — Spec Template

Spec writers: copy the two template blocks below into the spec, edit the rows to match the project's actual UI gestures, and keep the section headings exactly as written. Phase 7.5 (Smoke Test) parses the resulting `forge-context.md` for these literal headings and falls back to v3 `/health`-only behavior when either heading is missing or empty.

Three example rows are provided below — one terminal surface, one input field, one button. Empty table is allowed when the build has no UI; in that case still emit the section heading and write `_n/a (headless library / docs-only)_` underneath so the smoke-test parser sees the section and skips cleanly.

---

## Critical Interactive Surfaces

| Surface name | Type | How to inject the gesture | Expected side-effect to assert |
|--------------|------|---------------------------|--------------------------------|
| Cockpit terminal | terminal (xterm) | Focus `.xterm-helper-textarea`, type `ls\n` | Stdout area receives `>` prompt within 2s; PTY child process visible in process list |
| Login form email field | input | Click `input[name="email"]`, type `user@example.com`, Tab, type `pw`, Enter | `POST /auth/login` issued; URL becomes `/dashboard` within 5s |
| Refresh button | button | Click `button[data-testid="refresh"]` | `GET /api/v1/items` issued; grid row count > 0 within 3s |

For a build with no UI (CLI tool, headless library, docs-only repo), emit the heading and a single line:

```markdown
## Critical Interactive Surfaces

_n/a (headless library / docs-only)_
```

---

## Default Focus Element

Single line. CSS selector of the element that should be focused on first paint, OR the literal token `n/a (no UI)`.

For a browser build:

```markdown
## Default Focus Element

.xterm-helper-textarea
```

For a headless / CLI / docs-only build:

```markdown
## Default Focus Element

n/a (no UI)
```

---

## Notes for spec writers

- **When to populate vs when to use `n/a`.** Populate `## Critical Interactive Surfaces` whenever the build ships any user-facing gesture surface — a browser page, a terminal, a CLI prompt with a TTY, a server route that renders HTML. Use `_n/a (headless library / docs-only)_` only when there is genuinely no human-driven gesture (pure library, docs-only repo, background daemon with no UI). For `## Default Focus Element`, use `n/a (no UI)` for every non-browser build (CLI tools, server-only APIs, headless libraries) — only browser builds with a renderable HTML page need a real selector.
- **Never omit either section.** Phase 7.5 parses for the literal headings; missing headings are treated as a parse error and the smoke-test phase logs a warning before falling back. Always emit both, even if the body says `n/a`.
- **Surface names must be unique** inside one spec — the smoke-test phase logs each pass/fail by the surface name.
- **Selectors must be CSS selectors** (querySelector-compatible). XPath is not supported. Selector typos break silently — validate selectors against the audit's feature surface in enhance mode, or against the planned component design in greenfield mode, before shipping the spec.
- **Gestures are described in plain English.** smoke-test.md maps them to `mcp__chrome-devtools__*` calls (`click`, `fill`, `type_text`, `press_key`) for browser builds and to curl / direct HTTP for server-only surfaces.
- **"Expected side-effect to assert" must be observable.** A network request, a DOM mutation, a log line, a file write, or a process becoming visible in the process list. "It feels right" / "the page looks correct" is not an assertion and will be rejected by Phase 8 review.
