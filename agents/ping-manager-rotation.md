---
name: ping-manager-rotation
description: Determines and publishes the ping-manager/on-call rotation for the next two weeks on a Notion page. Applies no-back-to-back, time-off (≥2 days), and yearly-balance rules, then updates the Notion calendar section in place. The Notion page is an arg, defaulting to the WP Media rotation page. Requires the Notion MCP — stops early with a clear message if unavailable. Invoked by the /ping-manager-rotation skill.
tools: [mcp__notion__notion-fetch, mcp__notion__notion-update-page]
maxTurns: 20
color: pink
---

## Notion MCP requirement

This agent needs the Notion MCP connected (`mcp__notion__notion-fetch` and
`mcp__notion__notion-update-page`). Before doing anything else, confirm both tools
are available in this session.

If either is missing: **stop immediately** and report:
> ⚠️ Notion MCP not available — this agent needs `mcp__notion__notion-fetch` and
> `mcp__notion__notion-update-page` to run. Connect the Notion MCP and try again.

Do not attempt any workaround (no web fetch, no manual page guess) — the page content
and update mechanism both depend on the Notion MCP.

---

## No config file

No config file is read. This agent takes its target page as an argument — see below.
It does not use REPO, TEMP_ROOT, or DISPLAY_NAME (AGENTS.md §3) since it operates
purely against a Notion page, not this repo.

---

## Arg: Notion page ID

```
/ping-manager-rotation [notion-page-id]
```

- **DEFAULT** (used when no arg is passed): `328ed22a22f080cba113d64fc5ab79a9`
  (`🏓 [WIP] Ping manager role` — WP Media's rotation page, section
  `🗓️Calendar for team rotation`).
- If a `notion-page-id` arg is given, use it instead — any team can point this agent
  at their own rotation page, as long as it follows the **Page convention** below.

Let `PAGE_ID` denote whichever value is in effect for the rest of this run.

---

## Page convention

For this agent to work on any team's page, **your page must contain these sections**
(same headings, same structure) somewhere on it — typically under a
`🗓️Calendar for team rotation` heading, but the agent locates them by heading text,
not by position:

- **Teammates in the rotation** — bullet list of names in the rotation pool.
- **Next weeks** — bullet list of `{label, assignee}` pairs for upcoming weeks.
- **Previous weeks** — bullet list of `{label, assignee}` pairs, most recent first.
- **Number of times a teammate was ping manager (per year)** — nested list of yearly counts per teammate.
- **Time-offs to consider** — bullet list of `{who, start_date, end_date}` ranges.

If any of these sections is missing from the target page, stop and report which
section(s) are absent rather than guessing their content.

---

# PING MANAGER ROTATION SKILL

**Notion page:** `PAGE_ID` (resolved above; defaults to `328ed22a22f080cba113d64fc5ab79a9`)
(`🏓 [WIP] Ping manager role` — section `🗓️Calendar for team rotation`)

---

## Step 1 — Fetch the current page state

Call `mcp__notion__notion-fetch` with `id: PAGE_ID`.

From the response, locate the `🗓️Calendar for team rotation` section (see Page
convention above) and extract:

| Variable | Source in page |
|---|---|
| `rotation_members` | Bullet list under **Teammates in the rotation** |
| `next_weeks` | Bullet list under **Next weeks** — parse as `{label, assignee}` pairs |
| `previous_weeks` | Bullet list under **Previous weeks** — parse as `{label, assignee}` pairs, most recent first |
| `yearly_counts` | Nested list under **Number of times a teammate was ping manager (per year)** |
| `time_offs` | Bullet list under **Time-offs to consider** — parse as `{who, start_date, end_date}` |

**Date parsing rules:**
- All dates are in the current year unless context suggests otherwise (e.g. "January" after November = next year).
- Ranges like `July 10-20` mean July 10 to July 20 (same month).
- Ranges like `Dec 28 - Jan 3` span year boundary.
- `N/A` in any field means "none / empty".

---

## Step 2 — Compute reference dates

Use the `currentDate` from system context (format: `YYYY-MM-DD`).

Compute:
- **`today`** = system date
- **`this_week_monday`** = most recent Monday ≤ today (subtract `(weekday_index % 7)` days, where Monday = 0)
- **`target_week_1`** = `this_week_monday + 7 days` (next Monday)
- **`target_week_2`** = `this_week_monday + 14 days` (the Monday after)

Format all dates for display as `"Month Day"` (e.g. `"June 29"`, `"July 6"`).

---

## Step 3 — Archive past weeks

From `next_weeks`, move any entry whose Monday date ≤ today into `previous_weeks`.

- **Prepend** them to `previous_weeks` (most recent at top).
- Keep only the remaining (future) entries in `next_weeks`.

---

## Step 4 — Determine existing assignments for the two target weeks

Check `next_weeks` (after archiving) for entries matching `target_week_1` and `target_week_2`.

- If an entry already exists and its assignee is a real teammate (not `TBD` or `N/A`), **keep it** — do not reassign.
- If missing or `TBD`/`N/A`, it needs assignment — continue to Step 5.

---

## Step 5 — Assign PM for each unassigned week

For each week that needs an assignee, run this algorithm:

### 5a. Identify the "previous PM"

The teammate who must be excluded to avoid back-to-back weeks.

Walk backwards through time from the target week:
1. Check `next_weeks` for the week immediately before the target (if already assigned).
2. If not found there, check the most recent entry in `previous_weeks` that has a real assignee.

The previous PM is the assignee of that entry.

### 5b. Compute time-off overlap for each teammate

For each teammate in `rotation_members`, compute how many days of the target week
(Monday through Sunday, inclusive) fall within their time-off ranges from `time_offs`.

A teammate is **ineligible due to time-off** if the overlap is **≥ 2 days**.

### 5c. Build the eligible pool

Start with all `rotation_members`. Remove:
1. The **previous PM** (no back-to-back).
2. Anyone **ineligible due to time-off** for this week (rule 5b).

If the eligible pool is empty (can happen with a very small team), assign `TBD` and emit a warning:
> ⚠️ No eligible teammate found for [date] — assign manually.

### 5d. Pick from the eligible pool

Among eligible teammates:
1. Sort by their **yearly count for the current year** (ascending — least assigned first).
2. For ties: sort alphabetically.
3. Assign the **first** teammate in this sorted list.

Update `yearly_counts` for the current year: increment the chosen teammate's count by 1.

---

## Step 6 — Yearly balance check

After all assignments are resolved, compute:

```
weeks_remaining_in_year  = floor((Dec 31 − today) / 7) + 1
rotation_size            = len(rotation_members)
weeks_per_person         = weeks_remaining_in_year / rotation_size   # float

max_count = max yearly count among all rotation_members (current year)
min_count = min yearly count among all rotation_members (current year)
```

**Risk condition:** `max_count − min_count ≥ 2`

If at risk, append this warning block below "Next weeks":

```
⚠️ **Balance warning:** [Name] has served [N] times vs [Name] with [M] times.
With ~[weeks_remaining_in_year] weeks left and [rotation_size] people in rotation,
consider prioritising [underserved names] in upcoming weeks.
```

Remove the warning block if the condition is no longer met.

---

## Step 7 — Build the updated Calendar section

Produce the full replacement text for the `🗓️Calendar for team rotation` section.

**Format (keep identical markdown structure to the original):**

```markdown
## 🗓️Calendar for team rotation
**Teammates in the rotation:**
- [member 1]
- [member 2]
- …

**Next weeks:**
- [Month Day]: [Assignee]
- [Month Day]: [Assignee]
[optional ⚠️ balance warning block here]

**Previous weeks:**
- [Month Day]: [Assignee]
- [Month Day]: [Assignee]
- …

**Number of times a teammate was ping manager (per year):**
- [Year]:
	- [Teammate]: [count]
	- …

**Time-offs to consider:**
- [Who]: [date range]
- …
```

**Ordering rules:**
- **Next weeks**: reverse-chronological descending (furthest week first).
- **Previous weeks**: reverse-chronological descending (most recent first), keep all existing history.
- **Yearly counts**: current year first, then past years descending.

---

## Step 8 — Update the Notion page

Call `mcp__notion__notion-update-page` with:

```json
{
  "page_id": "PAGE_ID",
  "command": "update_content",
  "content_updates": [
    {
      "old_str": "<exact text of the 🗓️Calendar for team rotation section from the fetched page>",
      "new_str": "<new calendar section text from Step 7>"
    }
  ]
}
```

Use the **exact** `old_str` as it appeared in the fetched content (including the `## 🗓️Calendar for team rotation` heading). Do not guess — copy it verbatim from the Step 1 fetch result.

---

## Step 9 — Report to the user

After the Notion update succeeds, output a summary:

```
✅ Ping manager rotation updated.

📅 Next weeks:
  • [Month Day] (Week of [Mon]–[Sun]): **[Assignee]**
  • [Month Day] (Week of [Mon]–[Sun]): **[Assignee]**

📊 Yearly totals ([year]):
  • [Teammate]: [count]  …

[⚠️ balance warning if applicable]

📝 Archived to Previous weeks:
  • [Month Day]: [Assignee]  (if any were archived this run)
```

If no weeks were archived and both target weeks were already assigned, note:
> ℹ️ Both upcoming weeks were already assigned. No changes made.

---

## Edge cases

| Situation | Handling |
|---|---|
| All teammates are time-off that week | Assign `TBD`, emit missing-assignee warning |
| Only one eligible teammate | Assign them (unavoidable) |
| Teammate added mid-year | Their count starts at 0 for the current year |
| Yearly counts missing for a teammate | Treat as 0 |
| Page fetch fails | Report the error; do not attempt to update |
| `update_content` fails (old_str not found) | Report mismatch; show the proposed new Calendar section as plain text so the user can update manually |
| Page missing a required section (see Page convention) | Stop; report which section(s) are missing rather than guessing |
