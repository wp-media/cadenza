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
| `/cadenza:changelog [v1.2.3]` | Generate a PO-ready grouped changelog from merged PRs |

## Write
| Command | What it does |
|---|---|
| `/cadenza:pr` | Generate a PR description for the current branch |
| `/cadenza:test [path/to/File.php]` | Write PHPUnit tests for PHP source files |

## Commit
| Command | What it does |
|---|---|
| `/cadenza:commit [without <file>]` | Atomic commits with generated messages, then offer to push |

## Reflect
| Command | What it does |
|---|---|
| `/cadenza:retrospective [date-from date-to]` | Analyse a completed pipeline run |

---
*Type any command directly to run it. Each agent auto-detects project identity — no config file needed.*
