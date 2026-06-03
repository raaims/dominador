# code-reviewer subagent — design

Date: 2026-06-03
Status: approved (pending implementation)

## Goal

Add a `code-reviewer` subagent to `~/.config/opencode/opencode.json` that performs a general-purpose PR-style review of changed files in the working tree and returns severity-tagged findings. The subagent is invoked manually with `@code-reviewer`.

## Non-goals

- Auto-invocation from `build` or any other primary agent. Reviewer is manual-only.
- A `code-review` slash command, a `code-reviewer` skill, an MCP server, or a plugin. The subagent's prompt is the entire surface.
- Wiring into `plan`, `gsd-*`, or any other agent.
- Overriding `steps`, `reasoningEffort`, `temperature`, or `top_p` on the subagent.
- Adding a `provider` block. We rely on opencode's auto-discovery and the `MINIMAX_API_KEY` env var.

## Decisions

| Question | Decision |
| --- | --- |
| Scope | General-purpose PR reviewer (correctness, security, performance, design, style) |
| Invocation | Manual only — user invokes `@code-reviewer` |
| Auto-prompt | None — `build`, `plan`, and `AGENTS.md` are unchanged |
| Permissions | `edit: deny`, `bash: ask`, `webfetch: deny`, `websearch: deny` |
| Model | `minimax/MiniMax-M3` (provider `minimax`, env `MINIMAX_API_KEY`) |
| Output | Markdown list of severity-tagged findings |
| Location | Inline under `agent` in `opencode.json` |
| `steps` / `reasoningEffort` | Defaults (no override) |

## Architecture

One component. The `code-reviewer` subagent.

```
User ── "@code-reviewer <scope>" ──► code-reviewer (subagent, read-only)
                                          │
                                          ├─► reads changed files in working tree
                                          ├─► reads immediate callers/callees
                                          └─► returns severity-tagged findings
```

Data flow:
1. User invokes `@code-reviewer` with a description of what to review (files, a diff, a feature, or "everything I just changed"). If they pass nothing, the subagent falls back to `git diff` against the parent commit (or `git diff HEAD` if there is no parent).
2. Subagent reads the full content of each changed file plus the surrounding context.
3. Subagent emits a markdown report in the shape defined below.
4. Caller (the user) acts on the report.

## Configuration changes

### `opencode.json`

A new entry under the existing `agent` block. Position within the object does not matter for opencode; placing it last (after `gsd-debugger`) keeps the diff localized:

```json
"code-reviewer": {
  "description": "General-purpose code reviewer. Reads changed files in the current project and returns severity-tagged findings across correctness, security, performance, design, and style. Invoke manually with @code-reviewer on a set of files, a diff, a commit, or a feature description. Does not edit files.",
  "mode": "subagent",
  "model": "minimax/MiniMax-M3",
  "permission": {
    "edit": "deny",
    "bash": "ask",
    "webfetch": "deny",
    "websearch": "deny"
  },
  "prompt": "..."
}
```

The `prompt` field is the multi-paragraph body shown in the "Subagent prompt" section below.

No other fields in `opencode.json` are touched. `$schema`, the `plugin` block, and every existing `agent` entry (including the `build` override) are preserved verbatim.

### Environment

The user must have `MINIMAX_API_KEY` set in the environment before starting opencode. If it is missing, opencode will fail to start once the `code-reviewer` agent is referenced.

Document this in the implementation plan, not in the spec.

## Subagent prompt

```
You are a general-purpose code reviewer. You do not write code. You read code
and report findings.

## When you are invoked
You will be given a description of what changed (a set of files, a diff, a
commit, or a feature description). If none is given, run `git diff` against
the parent commit (or `git diff HEAD` if there is no parent) to discover the
changes. If the working tree is clean, ask the user what they want reviewed.

## How to investigate
1. Read the full content of every changed file, not just the diff hunks.
   Diff context is often where the real bug lives.
2. Read the immediate callers and callees of changed functions/types. A
   correct change in isolation can still be wrong in context.
3. For security findings, check the boundary: where does untrusted input
   enter, and what does it touch on the way out?
4. For performance findings, look for O(n^2) in hot paths, unbounded
   allocations, N+1 queries, and missing indexes.
5. For design findings, ask: would the next person to touch this code
   understand it without a tour?

You may use read-only shell commands when the user approves: `git`, `cat`,
`grep`, language formatters, and the like. You may not run tests, build,
install, or network commands.

## How to report
Return findings as a markdown list. Each finding has this exact shape:

- **<severity>** — <category>: <one-line summary>
  - **Location:** `path/to/file.ts:LINE` (or `path/to/file.ts:LINE-LINE`)
  - **Why it matters:** <1-2 sentences explaining the real-world impact>
  - **Suggested fix:** <concrete change, not a vague direction>

Severity levels (use them in this order of strictness):
- **blocker** — must fix before merge. Data loss, security hole, crash, or
  the change does not do what its description claims.
- **major** — should fix before merge. Bug under realistic conditions,
  significant performance regression, missing required handling, API
  contract violation.
- **minor** — fix when convenient. Edge case, defensive gap, readability
  issue that will hurt later.
- **nit** — style, naming, comment, formatting. Optional.

Categories: `correctness`, `security`, `performance`, `design`, `style`,
`docs`, `tests`.

Order the list by severity (blocker first). Within a severity, group by
file. If there are no findings, say so explicitly: "No findings. Reviewed
N files, M lines." Do not invent findings to look thorough.

## Things you do not do
- You do not edit files. You do not apply fixes. You report, the caller
  acts.
- You do not run tests, builds, or package managers.
- You do not look up external documentation unless the user explicitly
  asks. Stay in the working tree.
- You do not produce a "verdict" line (approve / request changes). Severity
  tags carry the same signal more honestly.
- You do not repeat the change back to the user. They can read the diff.
```

## Permissions

| Tool | Action | Why |
| --- | --- | --- |
| `edit` | `deny` | Reviewer is read-only by design. Caller acts on findings. |
| `bash` | `ask` | Reviewer may run read-only commands (`git diff`, `git log`, `cat`, `grep`, formatters). Destructive or build/install commands require explicit user approval. |
| `webfetch` | `deny` | No external lookups. Review is local to the working tree. |
| `websearch` | `deny` | Same as above. |

All other tools (`read`, `glob`, `grep`, `list`, `lsp`, `skill`, `todowrite`, `question`) inherit the user's defaults. The user can still deny any individual tool call at runtime.

## Error handling

- **`MINIMAX_API_KEY` missing.** opencode will refuse to start the first time the agent is referenced. The user is told to set the env var and restart. This is a startup failure, not a runtime error, and the spec does not handle it beyond noting it.
- **Working tree clean, no scope given.** Subagent asks the user what they want reviewed. Does not invent a review.
- **Diff too large to read in one pass.** Subagent reads files in priority order (changed files first, then direct callers/callees). It does not need to read every file in the repo.
- **No findings.** Subagent reports "No findings. Reviewed N files, M lines." Empty reports are valid.

## Testing and verification

Verification of the config change itself (not the agent's behavior):

1. After saving, run `python -c "import json; json.load(open('opencode.json'))"` to confirm the file parses as valid JSON.
2. Re-read `opencode.json` and visually confirm:
   - `$schema` and `plugin` block are unchanged.
   - The `build`, `general`, `explore`, `ui-ux-designer`, `mr-boss-vic`, and `gsd-*` entries are unchanged.
   - The new `code-reviewer` entry sits inside the `agent` object and matches the schema in `https://opencode.ai/config.json` (`AgentConfig`).
3. Run `git diff opencode.json` and confirm only the new entry was added.
4. Tell the user to quit and restart opencode. Config is loaded once at startup; the running session will keep using the already-loaded config.

Behavioral verification (manual, after restart):

1. Invoke `@code-reviewer` in a small, known-good change in any project. Confirm the report shape matches the spec.
2. Invoke `@code-reviewer` in a project with an obvious bug. Confirm it produces a finding of the right severity.

## Open follow-ups (not in this spec)

- Whether to add a `code-review` slash command in `opencode.json` for a one-keystroke invocation. (Skipped for now; `@code-reviewer` is enough.)
- Whether to wire the auto-prompt back in later, possibly via `AGENTS.md` or a per-project opt-in flag. (Skipped per user's decision.)
- Whether to add a `code-reviewer` skill that front-loads `find-docs` for the languages/libraries under review. (Skipped; review should stay in the working tree per the prompt.)
