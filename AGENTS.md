# AI Coding & Architecture Guidelines

This file defines NON-NEGOTIABLE rules for any AI-assisted work
(Claude Code, ChatGPT, JetBrains AI Assistant, Cursor, etc.)
in this repository.

Skills define behavioral guidance.
AGENTS.md defines mandatory guardrails.
If a conflict exists, AGENTS.md prevails.

This is the **Cadenza base**. Each project copies this file and extends it with a
**Project Overview** section and a **Session Learnings** section at the bottom.
The Cadenza plugin never overwrites `AGENTS.md`.

---

## Operating Principles

These five rules apply to every agent, in every phase, before any skill-specific guidance loads.

1. **Surface assumptions before building.** If the spec or codebase leaves something ambiguous, state the assumption explicitly before acting on it — don't silently guess.
2. **Stop when requirements conflict.** If the issue, the spec, and the codebase contradict each other, stop and surface the conflict. Proceeding on a guess produces bugs that are hard to trace.
3. **Push back when warranted.** If the simplest correct solution differs from the plan, say so. Prefer boring, obvious solutions over clever ones. An elegant approach that introduces risk is worse than a dull one that doesn't.
4. **Touch only what you are asked to touch.** Scope discipline is the single biggest determinant of whether a PR is mergeable. Do not refactor adjacent code, rename unrelated identifiers, or "clean up while you're in the area."
5. **Verification is not optional.** "Seems right" never closes a task. Every change must be confirmed by running tests, tools, or a manual scenario — not by reading the code and inferring it should work.

---

# 1. Project Overview

Cadenza is a **Claude Code plugin** containing standalone utility agents:

| Agent | Purpose |
|---|---|
| `changelog-agent` | Generates a PO-ready grouped changelog from merged PRs since the last release |
| `pr-agent` | Writes a structured PR description for the current branch |
| `test-writer` | Authors PHPUnit tests for PHP source files |
| `retrospective-agent` | Analyses a completed pipeline run and surfaces learnings |
| `issue-writer` | Turns raw context or a thread into a well-structured GitHub issue |
| `sentry-triage` | Fetches Sentry production errors, scores them P1–P4, posts triage notes, and drafts GitHub issues for the important ones (requires the Sentry MCP) |
| `sprint-planner` | Writes a paste-ready Slack sprint message (kick-off / mid-sprint / end-of-sprint) from a GitHub Projects sprint (Notion MCP optional) |
| `sentry-stats` | Produces a per-person Sentry activity dashboard (archives, resolves, notes with a heatmap) across projects for a period, published as an HTML artifact (requires the Sentry MCP) |

Each agent is fully self-contained. They require no orchestration layer, no delivery pipeline, and no other agents. They auto-detect project identity — no config file to read (see §3).

**Orchestrator skills.** A few skills coordinate a small fan-out of sub-agents from the main session rather than spawning a single worker. These have no dedicated `agents/*.md` file — the flow lives entirely in the skill:

| Orchestrator skill | Purpose |
|---|---|
| `backlog-triage` | Audits the current repo's open GitHub-issue backlog: an interactive scope gate, then a fan-out (feature-map code review + per-issue categorizers + already-fixed verification), producing a ranked shortlist of AI-autonomous quality wins plus a full Should Keep / Already Fixed / Unsure / Close Candidate triage as an HTML artifact. Read-only against GitHub — never closes, comments on, or edits issues. |

---

# 2. Plugin Architecture

## Directory layout

```
cadenza/
├── agents/       ← Sub-agent definitions (one .md per agent)
└── skills/       ← Skill definitions (one <name>/SKILL.md per slash command)
```

**`agents/`** contains the sub-agent markdown files. Each file defines the agent's
instructions, tools, maxTurns, and frontmatter metadata. Agents are invoked as
sub-agents by their corresponding command skill.

**`skills/`** contains the skill (slash command) files, one `<name>/SKILL.md` per
command. Each skill is a thin entry point: it auto-detects project identity (see §3),
resolves any user-supplied arguments, and then spawns the corresponding agent as a sub-agent.

## Command → agent flow

```
User runs /cadenza:changelog
       ↓
skills/changelog/SKILL.md (skill)
  - auto-detects project identity (no config file)
  - resolves baseline argument
       ↓
agents/changelog-agent.md (sub-agent)
  - does all the work
  - writes output to {temp_root}/
  - reports back to the skill
       ↓
Skill surfaces the result to the user
```

## Identity resolution

Every agent auto-detects the variables it needs at startup, per the cascade in §3.
Agents must never hardcode project-specific values — identity is always detected fresh,
never read from a config file, and never causes the agent to stop.

---

# 3. Project Identity (auto-detected, no config file)

Cadenza reads **no config file**. There is no `.claude/cadenza.json` and never will be —
every agent auto-detects its own project identity at startup, per the cascade below.

## Resolver block

Every agent reproduces this exact block to resolve `REPO`, `TEMP_ROOT`, and `DISPLAY_NAME`:

```bash
# --- Cadenza project identity (no config file — pure auto-detection) ---
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null)
[ -z "$REPO" ] && REPO=$(git remote get-url origin 2>/dev/null | sed -E 's#.*[:/]([^/]+/[^/]+?)(\.git)?$#\1#')
# REPO may be empty for local-only repos — warn, use TODO(repo), do NOT exit.

TEMP_ROOT=".ai/cadenza"   # nested under .ai/ so scratch output stays gitignored

DISPLAY_NAME=$(grep -rhoE '^\s*\*?\s*Plugin Name:\s*.+' . --include=*.php 2>/dev/null | head -1 | sed -E 's/.*Plugin Name:\s*//')
[ -z "$DISPLAY_NAME" ] && DISPLAY_NAME=$(jq -r '.name // empty' composer.json 2>/dev/null)
[ -z "$DISPLAY_NAME" ] && DISPLAY_NAME=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)")
```

## ARCH_SKILL resolution (test-writer only)

Optional, resolved by glob — first hit wins:

1. `.claude/skills/*architecture*/SKILL.md`
2. `.claude/commands/*architecture*.md`

If neither matches, skip silently — `test-writer` falls back to sampling existing tests.

## Non-abort rule

Agents **never exit on missing identity**. If `REPO` can't be resolved, use a `TODO(repo)`
placeholder in output plus a one-line warning — do not stop, do not ask the user to create
a config file.

## Values at a glance

| Value | Detection source | Fallback |
|---|---|---|
| `REPO` (`owner/repo`) | `gh repo view --json nameWithOwner`, else parsed from `git remote get-url origin` | unresolved → `TODO(repo)` placeholder + one-line warning, never abort |
| `TEMP_ROOT` | — | default `.ai/cadenza` (under `.ai/`, which projects gitignore) |
| `DISPLAY_NAME` | `Plugin Name:` header in main plugin PHP → `composer.json .name` → repo dir name | repo dir name |
| `ARCH_SKILL` (test-writer only) | glob `.claude/skills/*architecture*/SKILL.md` then `.claude/commands/*architecture*.md`; first hit. **Stored value is the full matched file path — read it directly, never re-template it into another path.** | skip — test-writer samples existing tests |

---

# 4. Git & Commit Policy

By default, Cadenza agents **do not commit or push**. They produce output files or console output and stop.

Exception: if a user explicitly instructs an agent to commit a generated file (e.g. a test stub), the agent may run a single atomic `git commit` with a Conventional Commits message. It must never `git push` without explicit instruction.

---

# 5. Output File Conventions

Agents write output files under `{temp_root}/`:

| Agent | Output path |
|---|---|
| `changelog-agent` | `{temp_root}/changelog/changelog-next-version-po-YYYY-MM-DD.md` |
| `retrospective-agent` | `{temp_root}/retrospectives/retrospective-YYYY-MM-DD.md` |
| `test-writer` | Mirrors source path under `tests/` — does not use `temp_root` |
| `pr-agent` | `{temp_root}/issues/<N>/pull.md` (or `issues/<branch-slug>/pull.md` if no issue number) |

If an output file already exists, append `-v2`, `-v3`, etc. Never overwrite silently.

`issue-writer` is the exception: it does not write a persistent output file under `{temp_root}/`. It creates the GitHub issue directly (via `gh issue create` / `gh issue edit`) once the user confirms the drafted body and labels. A scratch file may be used transiently to pass `--body-file` to `gh`, but it is not a retained output artifact.

`sentry-triage` is also an exception: it writes its triage report and GitHub issue drafts to a local `.sentry-triage/` directory at the repo root (gitignored — the agent appends it to `.gitignore` if missing), not under `{temp_root}/`. Nothing is posted to GitHub automatically; a human reviews each draft and runs the `gh issue create` command in its header.

`sentry-stats` is also an exception: it writes its HTML dashboard to the session scratchpad and publishes it as an artifact, not under `{temp_root}/`.

`sprint-planner` writes the composed Slack message to `{temp_root}/sprint-planner/sprint-<number>-message.txt`.

`backlog-triage` is an orchestrator skill (no agent file): it writes its HTML audit to the session scratchpad and publishes it as an artifact, not under `{temp_root}/`.

---

# 6–12. Reserved for Project Extensions

Sections 6 through 12 are intentionally absent from this base file. Projects that
copy-extend this AGENTS.md should add their own domain-specific sections here
(e.g. coding standards, deployment rules, test policy, security guardrails).

---

# 13. Session Learnings

**Human-curated only.** Never regenerate this section with an LLM — doing so degrades
agent success rates. After each run, a human adds entries for findings that were
surprising and are not already derivable from the code or other sections of this file.

Format per entry:
```
- **[YYYY-MM-DD] [agent or area]**: What was surprising. What the correct approach is.
```

Agents MUST read this section. It takes precedence over any assumption derived from
skill files when there is a conflict.

---

_No entries yet. Add one after the first surprising agent finding._
