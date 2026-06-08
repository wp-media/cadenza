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

*A Claude Code plugin containing four standalone utility agents — each one useful on its own, without the full Maestro delivery pipeline.*

---

[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-blueviolet?style=flat-square)](https://claude.ai/code)
[![License: MIT](https://img.shields.io/badge/license-MIT-green?style=flat-square)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.0-orange?style=flat-square)](https://github.com/wp-media/cadenza/releases)

</div>

---

## What Is Cadenza?

Cadenza is a **Claude Code plugin** containing four standalone utility agents — changelog generation, PR description writing, PHPUnit test authoring, and pipeline retrospectives.

Like the cadenza passage in a concerto, these agents perform solo: they require no orchestration layer, no delivery pipeline, and no other agents to be useful. Install Cadenza by itself, or pair it with Maestro — they share the same config schema and coexist without conflict.

---

## Install

```
/plugin marketplace add wp-media/claude-marketplace
/plugin install cadenza@wp-media
```

Then add a config file to your project:

```
/cadenza:onboard
```

Or create `.claude/cadenza.json` manually — see [Configuration](#configuration) below.

---

## Agents

Four specialists. Each one has a single job.

| Agent | Command | Role |
|---|---|---|
| `changelog-agent` | `/cadenza:changelog` | Generates a PO-ready grouped changelog from merged PRs since the last release |
| `pr-agent` | `/cadenza:pr` | Writes a structured PR description for the current branch |
| `test-writer` | `/cadenza:test` | Authors PHPUnit tests for PHP source files |
| `retrospective-agent` | `/cadenza:retrospective` | Analyses a completed pipeline run and surfaces learnings |

---

## Commands

| Command | What it does |
|---|---|
| `/cadenza` | Show the full command map — quick reference for all agents |
| `/cadenza:changelog` | Generate a PO-ready grouped changelog from merged PRs |
| `/cadenza:pr` | Generate a PR description for the current branch |
| `/cadenza:test` | Write PHPUnit tests for PHP source files |
| `/cadenza:retrospective` | Analyse a completed pipeline run |

---

## One Config, Any Project

Every agent reads a single config file — `.claude/cadenza.json` — committed in your repo. The same agents work across all your projects; only the config changes.

```
Without Cadenza                    With Cadenza
─────────────────────              ──────────────────────────────
project-a/                         Cadenza plugin (installed once)
  .claude/agents/ ──┐                agents/        ← one score, always current
  .claude/skills/ ──┤                commands/      ← one score, always current
     (drifting)    │
                   │              project-a/
project-b/         │                .claude/cadenza.json   ← project identity
  .claude/agents/ ──┤
  .claude/skills/ ──┤              project-b/
   (older, drifted) │                .claude/cadenza.json   ← project identity
```

**Updates are automatic.** When Cadenza ships a new version, every project picks it up on the next Claude session — no action needed.

---

## Configuration

Create `.claude/cadenza.json` in your project root. This is a **different file from `maestro.json`** — both plugins can be installed at the same time without conflict.

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

| Key | Required | Description |
|---|---|---|
| `ai.repo` | Yes | GitHub repository in `owner/repo` format. Used for PR and issue links. |
| `ai.temp_root` | Yes | Directory where agents write output files (e.g. changelogs, test stubs). |
| `ai.display_name` | Yes | Human-readable project name used in generated output. |
| `ai.architecture_skill` | No | Name of the project-specific architecture skill. Loaded by agents that need structural context. |

---

## Using Cadenza Alongside Maestro

Cadenza and Maestro share the same config schema but use different filenames (`.claude/cadenza.json` vs `.claude/maestro.json`). You can install both plugins in the same project — they operate independently and do not conflict.

When Maestro is installed, its built-in `changelog-agent`, `pr-agent`, `test-writer`, and `retrospective-agent` take precedence within the Maestro pipeline. Cadenza's versions are available as standalone commands via the `/cadenza:*` prefix.

---

## Repository Layout

```
cadenza/
│
├── AGENTS.md                        ← Base guardrails every project extends
├── .claude-plugin/
│   └── plugin.json                  ← Claude Code plugin manifest
│
├── agents/                          ← 4 standalone agents
│   ├── changelog-agent.md
│   ├── pr-agent.md
│   ├── test-writer.md
│   └── retrospective-agent.md
│
└── commands/                        ← Skills (slash commands)
    ├── cadenza.md
    ├── changelog.md
    ├── pr.md
    ├── test.md
    └── retrospective.md
```

---

<div align="center">

*Play the solo.*

</div>
