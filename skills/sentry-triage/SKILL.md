---
name: sentry-triage
description: Triage new, regressed, and escalating Sentry production errors — score them, post triage notes, and draft GitHub issues for the important ones (requires the Sentry MCP).
---

# Sentry Triage

Fetches new, regressed, and escalating Sentry production errors, scores them on a P1–P4 gravity matrix, posts a triage note per Sentry issue, and drafts GitHub issues locally for P1–P3 — nothing is posted to GitHub automatically.

**Requires the Sentry MCP.** If it isn't connected, the agent stops early with a clear message instead of erroring out.

## Project identity

Project identity is auto-detected (see AGENTS.md §3) — no config file. Sentry-domain settings (`org_slug`, `project_slug`, `environment`, `min_priority`) come from the repo's own `AGENTS.md` "## Sentry Triage" block, or repo-name auto-detection as a fallback.

## Steps

1. Parse optional arguments from `$ARGUMENTS`, in order: `org_slug` `project_slug` `date` `min_priority`. All are optional — the agent falls back to `AGENTS.md` config, then repo-name auto-detection, then asks the user.

2. Spawn `sentry-triage` as a sub-agent, passing along any parsed arguments.

3. Return the triage report path plus a summary table of P1–P4 counts and the paths to any drafted GitHub issue files (including fingerprinting fix drafts).
