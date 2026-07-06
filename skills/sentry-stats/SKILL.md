---
name: sentry-stats
description: Produce a per-person Sentry activity dashboard (archives, resolves, notes with a heatmap) across projects for a given period. Requires the Sentry MCP.
---

# Sentry Stats

Publishes a dark-themed HTML dashboard of per-person Sentry activity — archives, resolves, and notes — across an org's projects for a time period, with a contributors table, archive heatmap, and action breakdown.

Requires the **Sentry MCP** to be connected. If it isn't, the agent stops early rather than erroring.

## Invocation contract

```
/sentry-stats [period] [projects…] [--org group-one]
```

- `period` — optional (year / `YYYY-MM` / `Qn YYYY` / date range `YYYY-MM-DD:YYYY-MM-DD`). Defaults to the current year.
- `projects…` — optional, space-separated slugs. Defaults to the 6 group-one projects (datator, imagify, monies, rocketcdn, saas-director, wp-rocketme).
- `--org` — optional Sentry org slug. Defaults to `group-one`.

## Steps

1. Parse `$ARGUMENTS` into `period`, an optional list of `projects`, and an optional `--org` value.

2. Spawn `sentry-stats` as a sub-agent, passing the parsed `period`, `projects`, and `org`.

3. Return the published artifact link plus a short top-contributors summary (name, archives, resolves, notes, total) once the agent reports back.
</content>
