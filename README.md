<div align="center">

```
 ██████╗ █████╗ ██████╗ ███████╗███╗   ██╗███████╗ █████╗
██╔════╝██╔══██╗██╔══██╗██╔════╝████╗  ██║╚══███╔╝██╔══██╗
██║     ███████║██║  ██║█████╗  ██╔██╗ ██║  ███╔╝ ███████║
██║     ██╔══██║██║  ██║██╔══╝  ██║╚██╗██║ ███╔╝  ██╔══██║
╚██████╗██║  ██║██████╔╝███████╗██║ ╚████║███████╗██║  ██║
 ╚═════╝╚═╝  ╚═╝╚═════╝ ╚══════╝╚═╝  ╚═══╝╚══════╝╚═╝  ╚═╝
```

**Standalone solos. No orchestra required.**

*Four utility agents that each have one job — and do it well, with no pipeline, no orchestration, and no ceremony.*

---

[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-blueviolet?style=flat-square)](https://claude.ai/code)
[![License: MIT](https://img.shields.io/badge/license-MIT-green?style=flat-square)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.0-orange?style=flat-square)](https://github.com/wp-media/cadenza/releases)

</div>

---

## What Is Cadenza?

In a concerto, the **cadenza** is the moment the orchestra pauses and a single instrument plays alone — free, expressive, complete on its own terms.

Cadenza is a **Claude Code plugin** built on that same idea. It contains four specialist agents that each handle one task: generating changelogs, writing PR descriptions, authoring PHPUnit tests, and producing retrospective reports. No pipeline. No spec. No twelve-agent handoff chain. Just ask, and it's done.

Use Cadenza on its own for projects that don't need a full delivery pipeline. Or pair it with [Maestro](https://github.com/wp-media/maestro) — both plugins share the same config schema and coexist without conflict.

---

## Install

```
/plugin install cadenza@wp-media
```

Then add a config file to your project:

```jsonc
// .claude/cadenza.json
{
  "ai": {
    "repo":               "my-org/my-plugin",
    "temp_root":          ".cadenza",
    "display_name":       "My Plugin",
    "architecture_skill": "my-plugin-architecture"
  }
}
```

That's it. All four agents are ready.

---

## The Soloists

Four agents. Each one performs alone.

| Agent | Command | What it does |
|---|---|---|
| `changelog-agent` | `/cadenza:changelog` | Collects merged PRs since the last release, groups them by user impact, and writes a PO-ready changelog draft |
| `pr-agent` | `/cadenza:pr` | Reads your branch commits and diffs, then writes a structured PR description — without pushing anything |
| `test-writer` | `/cadenza:test` | Discovers your project's test conventions, then authors PHPUnit unit and integration tests for PHP source files |
| `retrospective-agent` | `/cadenza:retrospective` | Scans completed pipeline runs, surfaces DOD pass rates and loop-back patterns, and suggests concrete `AGENTS.md` learnings |

---

## Commands

| Command | What it does |
|---|---|
| `/cadenza` | Show the full command map |
| `/cadenza:changelog [v1.2.3]` | Generate a PO-ready changelog from merged PRs |
| `/cadenza:pr` | Write a PR description for the current branch |
| `/cadenza:test [path/to/File.php]` | Author PHPUnit tests for a source file (or auto-detect from branch diff) |
| `/cadenza:retrospective [date-from date-to]` | Analyse completed pipeline runs and surface learnings |

---

## One Config, Any Project

Every agent reads `.claude/cadenza.json` — a single file committed in your repo. The agents live in the plugin. The identity lives in the config.

```
Cadenza plugin (installed once)        Your projects
──────────────────────────────         ────────────────────────────────────
agents/                                project-a/
  changelog-agent.md  ──────────────►    .claude/cadenza.json   ← identity
  pr-agent.md         ──────────────►
  test-writer.md      ──────────────►  project-b/
  retrospective-agent.md ───────────►    .claude/cadenza.json   ← identity
commands/
  changelog.md                         project-c/
  pr.md                                  .claude/cadenza.json   ← identity
  test.md
  retrospective.md
```

When Cadenza ships a new version, every project picks it up on the next Claude session — no action needed.

---

## Configuration

| Key | Required | Description |
|---|---|---|
| `ai.repo` | Yes | GitHub repository in `owner/repo` format — used for PR and issue links |
| `ai.temp_root` | Yes | Directory where agents write output files (changelogs, PR drafts, reports) |
| `ai.display_name` | Yes | Human-readable project name used in generated output |
| `ai.architecture_skill` | No | Name of the project-specific architecture skill — loaded by `test-writer` to discover test conventions |

---

## Using Cadenza Alongside Maestro

Cadenza and [Maestro](https://github.com/wp-media/maestro) share the same config schema but use **different filenames** — `.claude/cadenza.json` vs. `.claude/maestro.json`. Both plugins can be installed in the same project and operate without conflict.

When Maestro is installed, its pipeline invokes the same agents internally. Cadenza exposes them as standalone `/cadenza:*` commands — available anytime, outside any pipeline run.

Want to watch every agent event live? Add [Podium](https://github.com/wp-media/podium).

```
/plugin install podium@wp-media
/podium start   →   http://localhost:4820
```

---

## Repository Layout

```
cadenza/
│
├── AGENTS.md                        ← Base guardrails every project extends
│
├── agents/                          ← 4 standalone agents
│   ├── changelog-agent.md
│   ├── pr-agent.md
│   ├── retrospective-agent.md
│   └── test-writer.md
│
├── commands/                        ← Skills (slash commands)
│   ├── cadenza.md
│   ├── changelog.md
│   ├── pr.md
│   ├── retrospective.md
│   └── test.md
│
└── refs/                            ← Bundled reference files
    └── pr-template.md               ← Default PR description template
```

---

<div align="center">

*The orchestra pauses. You play.*

---

*Part of the Orchestra suite — [Maestro](https://github.com/wp-media/maestro) · [Podium](https://github.com/wp-media/podium) · Cadenza*

*Built at [WP-Media](https://wp-media.me)*

</div>
