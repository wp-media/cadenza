---
name: cadenza
description: Show the Cadenza command map — a quick reference to all available agents and commands.
---

Print the following organised command reference to the user. Do not run anything else.

---

# Cadenza

**Standalone solos. No orchestra required.**

## Release
| Command | What it does |
|---|---|
| `/cadenza:changelog` | Generate a PO-ready grouped changelog from merged PRs |

## Write
| Command | What it does |
|---|---|
| `/cadenza:pr` | Generate a PR description for the current branch |
| `/cadenza:test` | Write PHPUnit tests for PHP source files |

## Reflect
| Command | What it does |
|---|---|
| `/cadenza:retrospective` | Analyse a completed pipeline run |

---
*Type any command directly to run it. Each agent reads `.claude/cadenza.json` for project identity.*
