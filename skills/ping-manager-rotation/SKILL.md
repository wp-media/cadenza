---
name: ping-manager-rotation
description: Determines and publishes the ping-manager/on-call rotation for the next two weeks on a Notion page — applies no-back-to-back, time-off, and yearly-balance rules. Requires the Notion MCP. Takes an optional notion-page-id arg, defaulting to the WP Media rotation page.
---

# Ping Manager Rotation

Updates the ping-manager (on-call) rotation for the next two weeks directly on a
Notion page — archives past weeks, applies eligibility rules (no back-to-back,
time-off ≥ 2 days, yearly balance), and writes the new calendar section back.

Requires the **Notion MCP** (`mcp__notion__notion-fetch`,
`mcp__notion__notion-update-page`). If it's not connected, the agent stops early
with a clear message instead of guessing.

## Steps

1. Resolve the target page: use `$ARGUMENTS` as `notion-page-id` if provided,
   otherwise fall back to the default WP Media rotation page
   (`328ed22a22f080cba113d64fc5ab79a9`).

2. Spawn `ping-manager-rotation` as a sub-agent, passing the resolved page ID.

3. Return the updated rotation summary: next two weeks with assignees, yearly
   totals per teammate, any balance warning, and which past weeks (if any) were
   archived.
