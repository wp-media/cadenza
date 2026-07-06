---
name: sprint-planner
description: Write a paste-ready Slack sprint message (kick-off / mid-sprint / end-of-sprint) from a GitHub Projects sprint.
---

# Sprint Planner

Writes a paste-ready Slack sprint message — kick-off, mid-sprint, or end-of-sprint, auto-detected from today's date against the sprint window — from a GitHub Projects (v2) board.

## Invocation contract

```
/sprint-planner [project] [org] [notion-page-id]
```

All optional, defaulting to WP Media's values:
- `project` — GitHub Projects (v2) number. Default `112`.
- `org` — GitHub organization login. Default `wp-media`.
- `notion-page-id` — Notion page ID for the ping-manager rotation calendar. Default `328ed22a22f080cba113d64fc5ab79a9`.

The Notion MCP is optional — it's only used for the closing ping-manager line. If it isn't connected, or the lookup resolves nothing, the message ends with `🏓 Ping manager: TBD` instead of a guess.

## Steps

1. Parse `$ARGUMENTS` into `project`, `org`, and `notion-page-id`, applying the WP Media defaults above for anything omitted. Also pass along any sprint qualifier the user gave (e.g. "Sprint 160", "current sprint", "next sprint") — the agent resolves the sprint and message type itself if none is given.

2. Spawn `sprint-planner` as a sub-agent, passing the resolved `project`, `org`, `notion-page-id`, and any sprint qualifier.

3. Once the agent reports back, return the file path it wrote the message to and confirm it's ready to paste into Slack.
