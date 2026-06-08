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

Each agent is fully self-contained. They require no orchestration layer, no delivery pipeline, and no other agents. They read project identity from `.claude/cadenza.json`.

---

# 2. Plugin Architecture

## Directory layout

```
cadenza/
├── agents/       ← Sub-agent definitions (one .md per agent)
└── commands/     ← Skill definitions (slash commands, one .md per command)
```

**`agents/`** contains the sub-agent markdown files. Each file defines the agent's
instructions, tools, maxTurns, and frontmatter metadata. Agents are invoked as
sub-agents by their corresponding command skill.

**`commands/`** contains the skill (slash command) files. Each skill is a thin
entry point: it loads project config from `.claude/cadenza.json`, resolves any
user-supplied arguments, and then spawns the corresponding agent as a sub-agent.

## Command → agent flow

```
User runs /cadenza:changelog
       ↓
commands/changelog.md (skill)
  - reads .claude/cadenza.json
  - resolves baseline argument
       ↓
agents/changelog-agent.md (sub-agent)
  - does all the work
  - writes output to {temp_root}/
  - reports back to the skill
       ↓
Skill surfaces the result to the user
```

## Config loading

Every agent reads `.claude/cadenza.json` at startup and extracts the variables it needs.
Agents must never hardcode project-specific values — all identity comes from config.

---

# 3. cadenza.json Config Schema

The canonical config file is `.claude/cadenza.json`, committed in each project repo.

```jsonc
{
  "ai": {
    "repo":               "my-org/my-plugin",
    "temp_root":          ".cadenza",
    "display_name":       "My Plugin",
    "architecture_skill": "my-plugin-architecture"
  }
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `ai.repo` | string | Yes | GitHub repository in `owner/repo` format. Used for PR and issue links in all generated output. |
| `ai.temp_root` | string | Yes | Root directory where agents write output files (changelogs, test stubs, retrospective reports, etc.). Committed or gitignored — your choice. |
| `ai.display_name` | string | Yes | Human-readable project name used in generated output headers and PR descriptions. |
| `ai.architecture_skill` | string | No | Name of the project-specific architecture skill (matches a `.claude/commands/<name>.md` file). Loaded by `test-writer` to discover test paths, naming conventions, and `@group` annotations. |

### Notes

- The filename is `.claude/cadenza.json`, not `maestro.json`. This allows both Cadenza and Maestro to be installed in the same project simultaneously without conflict.
- All agents read the same file. There is no per-agent config — the schema is intentionally minimal.
- If `.claude/cadenza.json` is absent, agents must stop immediately and instruct the user to create it. Never proceed with hardcoded fallbacks.

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
