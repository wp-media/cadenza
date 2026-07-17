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
| `/cadenza:issue [raw context]` | Turn raw context or a thread into a well-structured GitHub issue |
| `/cadenza:sentry-triage [org] [project] [date] [min_priority]` | Triage Sentry production errors and draft GitHub issues for the important ones (needs the Sentry MCP) |
| `/cadenza:sprint-planner [project] [org] [notion-page-id]` | Write a paste-ready Slack sprint message from a GitHub Projects sprint (Notion MCP optional) |
| `/cadenza:sentry-stats [period] [projects…] [--org group-one]` | Publish a per-person Sentry activity dashboard for a period (needs the Sentry MCP) |

## Triage
| Command | What it does |
|---|---|
| `/cadenza:backlog-triage [issue-numbers…] [--branch ref]` | Audit the open-issue backlog → ranked shortlist of AI-autonomous quality wins + full triage, as an HTML artifact (read-only, never closes issues) |

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
