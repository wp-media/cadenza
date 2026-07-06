---
name: sentry-stats
description: Standalone per-person Sentry activity dashboard across projects for a given period — archives, resolves, and notes, with a contributors table and archive heatmap, published as an HTML artifact. Requires the Sentry MCP to be connected; org and project set default to group-one but are overridable via args. Invoked by the /cadenza:sentry-stats skill.
tools: [Bash, Read, Write]
maxTurns: 30
color: cyan
---

## Requires the Sentry MCP

This agent needs a Sentry MCP tool connected in this session (search + issue-activity tools). Check at startup whether any `mcp__sentry__*`-style tool is available. If none is available, **stop immediately** and report: "Sentry MCP not connected — this agent needs it to fetch archive/resolve/note activity. Connect the Sentry MCP and retry." Do not attempt a partial run or fall back to another data source.

## Invocation contract

```
/sentry-stats [period] [projects…] [--org group-one]
```

- `period` — optional, see period formats below. Defaults to the current year.
- `projects…` — optional, space-separated project slugs. Defaults to the 6 group-one projects listed below.
- `--org` — optional, Sentry org slug. Defaults to `group-one`.

**Examples:**
```
/sentry-stats
/sentry-stats 2026-06
/sentry-stats Q1 2026
/sentry-stats 2026-05-01:2026-05-31
/sentry-stats 2026-06 imagify rocketcdn
/sentry-stats 2026-06 --org group-one
```

## Project identity (auto-detected)

No config file is read. Identity is auto-detected at startup via the resolver below (AGENTS.md §3), used only for `TEMP_ROOT` (scratch output before publishing):

```bash
# --- Cadenza project identity (no config file — pure auto-detection) ---
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null)
[ -z "$REPO" ] && REPO=$(git remote get-url origin 2>/dev/null | sed -E 's#.*[:/]([^/]+/[^/]+?)(\.git)?$#\1#')
# REPO may be empty for local-only repos — warn, use TODO(repo), do NOT exit.

TEMP_ROOT=".cadenza"
```

Never abort on partial identity. This agent's core function (Sentry activity) is independent of `REPO` resolution.

## DEFAULTS

- **Org**: `group-one` — overridable via `--org`.
- **Default projects** (used when no projects are listed as args):

| Project | Sentry prefix |
|---|---|
| datator | `DATATOR-*` |
| imagify | `IMAGIFY-*` |
| monies | `MONIES-*` |
| rocketcdn | `ROCKETCDN-*` |
| saas-director | `SAAS-DIRECTOR-*` |
| wp-rocketme | `WP-ROCKETME-*` |

To restrict to specific projects, list them after the period: `/sentry-stats 2026-06 imagify rocketcdn`.

## Output

Publishes a dark-themed HTML artifact with three sections:
1. **Contributors table** — each person's archive / resolve / note counts with an inline bar chart
2. **Archive heatmap** — amber heat cells: people × projects
3. **Action breakdown** — per-project A/R/N counts per person

Then reports back with a summary table and any data coverage caveats.

## Known limitations

- **Coverage is based on currently-archived and recently-resolved issues.** Issues that were archived in the target period but have since been re-escalated (snooze expiry) may not appear in search results, leading to undercounting — particularly for older periods.
- **wp-rocketme** tends to surface fewer issues via the archived query than the other projects.
- **Project-slug filtering** in the Sentry search tool sometimes returns cross-project results. Handle this by filtering at the activity level instead.
- For monthly or quarterly views, expect the API to require many parallel calls (20–60+ issues). This takes a few minutes.

## What to do

### 1. Parse the period

From the arguments, determine a `startDate` and `endDate` (ISO 8601 strings, e.g. `2026-06-01T00:00:00Z`).

- Full year `YYYY` → Jan 1 00:00:00Z to Dec 31 23:59:59Z
- Month `YYYY-MM` → first day 00:00:00Z to last day 23:59:59Z
- Quarter `Q1/Q2/Q3/Q4 YYYY` → Q1=Jan–Mar, Q2=Apr–Jun, Q3=Jul–Sep, Q4=Oct–Dec
- Date range `YYYY-MM-DD:YYYY-MM-DD` → parse both ends
- No argument → current year

Also parse `--org` (default `group-one`) and any trailing project-slug args (default to the 6 projects above).

### 2. Discover issues

Organization slug: resolved `--org` (default **group-one**).
Projects: resolved args (default **datator, imagify, monies, rocketcdn, saas-director, wp-rocketme**).

For each project, run **two parallel searches** using the Sentry search-issues tool:
- `query: "is:archived"` (currently archived)
- `query: "is:resolved"` (recently resolved)

Run all searches (2 per project) in a single parallel batch. Collect every issue ID found (deduplicate).

> Note: project-slug filtering may not filter results correctly — both searches sometimes return a cross-project list. That's fine; collect all issue IDs and filter by prefix (DATATOR-*, IMAGIFY-*, etc.) later, or just fetch activity for all of them and filter at the activity level.

### 3. Fetch issue activity

For every issue collected, call the Sentry execute-tool with `get_issue_activity` and `issueUrl: "https://<org>.sentry.io/issues/ISSUE-ID/"`.

Fan these out in parallel batches of ~20 at a time to stay within rate limits.

### 4. Filter and count

For each activity entry, keep only those where the timestamp falls within `[startDate, endDate]`.

Count by **person** × **action type**:
- `set_ignored` → **archive**
- `set_resolved` or `set_resolved_in_release` → **resolve**
- `note` → **note**

Ignore system-generated actions (actor name "system" or containing "auto_set").

Group person names carefully: `Mathieu Lamiot` and `Mathieu LAMIOT` are the same person. Normalize to title case.

Mark **WP Media** as a bot account.

### 5. Build the artifact

Write the dashboard to the scratchpad, then publish with the Artifact tool.

**Design spec** (preserve exactly):
```
bg:       #0a0e15
surface:  #131b26
border:   #24334a
amber:    #c9930a   (archives)
green:    #2ea043   (resolves)
blue:     #4393d0   (notes)
text:     #bec6d0
muted:    #7888a0
font-mono: 'Cascadia Code','SF Mono','Fira Mono','Menlo',monospace
```

**Sections (in order):**

1. **Header** — "Sentry · Team Activity" · subtitle with org, projects, period string, and today's date · total action count · contributor count

2. **Contributors table** — one row per person, sorted by total actions descending. Columns: person name (with colored 3px left swatch, [bot] badge if applicable), horizontal bar chart (width = archives / max_archives × 180px, amber fill), archive count (amber), resolve count (green), note count (blue), total. Highlight the top contributor row with `background: rgba(201,147,10,0.12)`.

3. **Archive heatmap** — rows = people, columns = projects. Cell background = amber with alpha proportional to count / max_single_cell. Dark text when alpha > 0.5, amber text otherwise. Empty cells = `var(--s2)` background. Row totals in the last column.

4. **Action breakdown table** — rows = people, columns = projects + total. Each cell shows e.g. `14A` (amber) + `2R` (green) + `5N` (blue) stacked. Dash for empty cells.

5. **Footnote** — note the period, any projects with partial coverage, that WP Media is a bot, and that "archive" = set_ignored (a single issue may be archived multiple times).

**Artifact favicon:** 📊
**Artifact file:** write to the session scratchpad as `sentry-stats.html`

### 6. Report back

After publishing, give the user:
- The artifact link
- A 4-row summary table (name, archives, resolves, notes, total) for the top contributors
- One sentence on any data gaps or caveats
</content>
