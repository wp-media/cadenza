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

*Utility agents that each have one job — and do it well, with no pipeline, no orchestration, and no ceremony.*

---

[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-blueviolet?style=flat-square)](https://claude.ai/code)
[![License: MIT](https://img.shields.io/badge/license-MIT-green?style=flat-square)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.2.3-orange?style=flat-square)](https://github.com/wp-media/cadenza/releases)

</div>

---

## What Is Cadenza?

In a concerto, the **cadenza** is the moment the orchestra pauses and a single instrument plays alone — free, expressive, complete on its own terms.

Cadenza is a **Claude Code plugin** built on that same idea. It contains specialist agents, each handling one task: generating changelogs, writing PR descriptions, authoring PHPUnit tests, and producing retrospective reports. No pipeline. No spec. No twelve-agent handoff chain. Just ask, and it's done.

Use Cadenza on its own for projects that don't need a full delivery pipeline. Or pair it with [Maestro](https://github.com/wp-media/maestro) — both plugins can be installed in the same project and coexist without conflict.

---

## Install

```
/plugin install cadenza@wp-media
```

That's it. Project identity is auto-detected from your repo — no config file to add. All agents are ready.

---

## The Soloists

Each one performs alone.

| Agent | Command | What it does |
|---|---|---|
| `changelog-agent` | `/cadenza:changelog` | Collects merged PRs since the last release, groups them by user impact, and writes a PO-ready changelog draft |
| `pr-agent` | `/cadenza:pr` | Reads your branch commits and diffs, then writes a structured PR description — without pushing anything |
| `test-writer` | `/cadenza:test` | Discovers your project's test conventions, then authors PHPUnit unit and integration tests for PHP source files |
| `retrospective-agent` | `/cadenza:retrospective` | Scans completed pipeline runs, surfaces DOD pass rates and loop-back patterns, and suggests concrete `AGENTS.md` learnings |
| `issue-writer` | `/cadenza:issue` | Turns raw context, a thread, or a rough note into a well-structured GitHub issue, then shows it for confirmation before creating |
| `sentry-triage` | `/cadenza:sentry-triage` | Fetches new/regressed/escalating Sentry production errors, scores them P1–P4, posts triage notes, and drafts GitHub issues for the important ones (needs the Sentry MCP) |
| `sprint-planner` | `/cadenza:sprint-planner` | Writes a paste-ready Slack sprint message (kick-off / mid-sprint / end-of-sprint) from a GitHub Projects sprint (Notion MCP optional) |
| `sentry-stats` | `/cadenza:sentry-stats` | Publishes a per-person Sentry activity dashboard — archives, resolves, notes — across projects for a period, with a contributors table and archive heatmap (needs the Sentry MCP) |
| *(orchestrator skill)* | `/cadenza:backlog-triage` | Audits the open-issue backlog via a fan-out (feature-map review + per-issue categorizers + already-fixed check) → ranked shortlist of AI-autonomous quality wins + full triage, as an HTML artifact (read-only; never closes issues) |
| *(built-in)* | `/cadenza:commit` | Analyses changed files, groups them into atomic commits with generated messages, then offers to push |

---

## Commands

| Command | What it does |
|---|---|
| `/cadenza` | Show the full command map |
| `/cadenza:changelog [v1.2.3]` | Generate a PO-ready changelog from merged PRs |
| `/cadenza:pr` | Write a PR description for the current branch |
| `/cadenza:test [path/to/File.php]` | Author PHPUnit tests for a source file (or auto-detect from branch diff) |
| `/cadenza:retrospective [date-from date-to]` | Analyse completed pipeline runs and surface learnings |
| `/cadenza:issue [raw context]` | Turn raw context or a thread into a well-structured GitHub issue |
| `/cadenza:sentry-triage [org] [project] [date] [min_priority]` | Triage Sentry production errors and draft GitHub issues (needs the Sentry MCP) |
| `/cadenza:sprint-planner [project] [org] [notion-page-id]` | Write a paste-ready Slack sprint message from a GitHub Projects sprint |
| `/cadenza:sentry-stats [period] [projects…] [--org group-one]` | Publish a per-person Sentry activity dashboard for a period (needs the Sentry MCP) |
| `/cadenza:backlog-triage [issue-numbers…] [--branch ref]` | Audit the open-issue backlog → ranked AI-autonomous quality wins + full triage artifact (read-only) |
| `/cadenza:commit [without <file>]` | Atomic commits with generated messages, then offer to push |

---

## Zero Config, Any Project

Every agent auto-detects project identity at startup — from your git remote, your plugin's `Plugin Name:` header, `composer.json`, or the repo directory name. The agents live in the plugin. Your projects have nothing to commit.

```
Cadenza plugin (installed once)        Your projects
──────────────────────────────         ────────────────────────────────────
agents/                                project-a/
  changelog-agent.md  ──────────────►    (auto-detected)
  pr-agent.md         ──────────────►
  test-writer.md      ──────────────►  project-b/
  retrospective-agent.md ───────────►    (auto-detected)
  issue-writer.md     ──────────────►
  sentry-triage.md    ──────────────►
skills/
  changelog/                           project-c/
  pr/                                    (auto-detected)
  test/
  retrospective/
  issue/
  commit/
```

When Cadenza ships a new version, every project picks it up on the next Claude session — no action needed.

---

## Configuration

None. Everything is auto-detected from your repo — nothing to commit.

---

## Using Cadenza Alongside Maestro

Cadenza and [Maestro](https://github.com/wp-media/maestro) can both be installed in the same project and operate without conflict.

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
├── agents/                          ← 8 standalone agents
│   ├── changelog-agent.md
│   ├── issue-writer.md
│   ├── pr-agent.md
│   ├── retrospective-agent.md
│   ├── sentry-stats.md
│   ├── sentry-triage.md
│   ├── sprint-planner.md
│   └── test-writer.md
│
├── skills/                          ← Skills (slash commands)
│   ├── backlog-triage/
│   ├── cadenza/
│   ├── changelog/
│   ├── commit/
│   ├── issue/
│   ├── pr/
│   ├── retrospective/
│   ├── sentry-stats/
│   ├── sentry-triage/
│   ├── sprint-planner/
│   └── test/
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
