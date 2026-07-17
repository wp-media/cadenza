---
name: backlog-triage
description: Orchestrated audit of the current repo's open GitHub-issue backlog — fans out code-review and per-issue categorization agents, verifies which issues are already fixed, and produces a ranked shortlist of AI-autonomous quality wins plus a full triage (Should Keep / Already Fixed / Unsure / Close Candidate), published as an HTML artifact. Read-only against GitHub — never closes, comments on, or edits issues.
---

# Backlog Triage

Turns a messy 100+ issue backlog into two things: a **ranked shortlist of low-hanging quality wins an AI agent can fix autonomously**, and a **full triage** of every issue in scope (Should Keep / Already Fixed / Unsure / Close Candidate). Published as a single HTML artifact.

This skill is the **orchestrator** — it runs in the main session and fans out sub-agents (a code-review pass, per-issue categorizers, an already-fixed verification pass), then synthesizes the result. Run the main session on a capable model (Opus, or Fable orchestrating) for best results.

**Read-only against GitHub.** It reads issues and code and writes an artifact. It never runs `gh issue close`/`comment`/`edit`/`label` — closing and prioritizing are the user's call. If ever tempted to mutate an issue, stop.

## Invocation contract

```
/cadenza:backlog-triage [issue-numbers…] [--branch <ref>]
```

- `issue-numbers…` — optional. If given, skip the scope gate and triage exactly those (re-triage mode — updates the existing artifact's rows).
- `--branch <ref>` — optional ref to verify claims against. Defaults to the repo's default branch (e.g. `develop`).

## Project identity (auto-detected)

No config file. Resolve identity at startup via AGENTS.md §3:

```bash
# --- Cadenza project identity (no config file — pure auto-detection) ---
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null)
[ -z "$REPO" ] && REPO=$(git remote get-url origin 2>/dev/null | sed -E 's#.*[:/]([^/]+/[^/]+?)(\.git)?$#\1#')
TEMP_ROOT=".cadenza"
DISPLAY_NAME=$(grep -rhoE '^\s*\*?\s*Plugin Name:\s*.+' . --include=*.php 2>/dev/null | head -1 | sed -E 's/.*Plugin Name:\s*//')
[ -z "$DISPLAY_NAME" ] && DISPLAY_NAME=$(jq -r '.name // empty' composer.json 2>/dev/null)
[ -z "$DISPLAY_NAME" ] && DISPLAY_NAME=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)")
BRANCH=$(gh repo view --json defaultBranchRef -q .defaultBranchRef.name 2>/dev/null)
```

**Deliberate override of §3's non-abort rule:** §3 says never stop on a missing `REPO` — but this skill genuinely cannot proceed without one (no repo = no backlog), so it *does* stop and ask the user which `owner/repo` to triage rather than guessing or using a `TODO(repo)` placeholder. The `BRANCH=` line is likewise a skill-specific addition, not part of the canonical §3 block. Also verify `gh auth status`; if `gh` isn't authenticated, stop and tell the user to run `gh auth login`.

## Steps

### 1. Count the backlog

```bash
gh issue list --repo "$REPO" --state open --limit 1000 --json number | jq length
```
(`gh issue list` already excludes pull requests.)

### 2. Scope gate (interactive — skip if issue-numbers were passed)

Present the count and let the user decide before spending tokens. Use the AskUserQuestion tool:

> "There are **N** open issues in `$REPO`. Processing all of them runs a code-review pass + one categorizer per issue + an already-fixed check — that can take a while and cost a lot of tokens. How do you want to scope it?"

Offer options, each with a concrete resolution:
- **Process all N** — full backlog. `gh issue list --state open --limit 1000`.
- **Recent only** — issues opened in the last ~18 months (where quality signal is freshest). Resolve to `--search "created:>=YYYY-MM-DD"` (compute the date from the system's current date).
- **Cap the count** — ask for a number M, then either the M most recent (`--limit M` with default newest-first ordering) or the M most-reacted (see note below).
- **Filter** — by label (`--label`), date range (`--search "created:YYYY-MM-DD..YYYY-MM-DD"`), or a raw `gh` search query the user supplies (`--search "<query>"`).

`gh issue list` cannot sort by reactions server-side. For "most-reacted M": fetch the full open set with `reactionGroups`, sort client-side (`jq`) by total reactions descending, and slice the top M.

Resolve the user's choice into a concrete query before fetching. Confirm the resolved scope in one line ("Scoping to 47 issues opened since 2025-01-01") so there are no surprises.

### 3. Fetch the in-scope issues

Fetch the resolved set once and hand slices to the workers — don't make each worker re-list. Use a resolved `LIMIT` and `SEARCH` rather than hardcoding:

```bash
gh issue list --repo "$REPO" --state open --limit "$LIMIT" \
  --json number,title,body,labels,createdAt,updatedAt,url,reactionGroups,comments \
  ${SEARCH:+--search "$SEARCH"}
```

### 4. Orchestrate the pipeline

Fan out sub-agents (Task tool) — don't do this inline; the point is to keep each worker's context small. **The three steps are a dependency chain and run in order a → b → c**; only the *batch workers within* step (b) run in parallel with each other.

**Every worker prompt must carry the read-only fence** — include verbatim: *"Read-only: you may read issues and code, but never run `gh issue close`/`comment`/`edit`/`label` or otherwise mutate GitHub. Return findings only."* The fence at the top of this skill governs the orchestrator; the workers hold the `gh` access, so they must be told too.

**a. Feature-map code review (1 agent, Opus) — runs first, to completion.** Spawn one agent to map the codebase by feature/subsystem and, for each area, note what changed recently on `$BRANCH` (`git log` since ~18 months) and which areas look fragile. Output: a feature → files/recent-fixes map. Its output feeds every step-(b) worker, so it must finish before (b) starts.

**b. Per-issue categorization (Sonnet, batched) — after (a).** Split the in-scope issues into batches (~8–12 per worker). Spawn the batch workers in **waves of ≤5 concurrent** to avoid overwhelming the fan-out. Pass each worker the feature-map from (a) and tell it to read the *specific subsystem* code, not just shared helpers. Each worker returns, **inline as a JSON array** (one object per issue in its batch), verdicts of this exact shape:
```json
{
  "number": 859,
  "category": "Should Keep",           // Should Keep | Already Fixed | Unsure | Close Candidate
  "ai_autonomy": "autonomous",          // autonomous | needs-human | needs-product | needs-design
  "impact": "medium",                   // low | medium | high
  "priority": "medium",                 // low | medium | high
  "rationale": "one line on impact/priority",
  "evidence": "file:line, commit SHA, or PR # backing the verdict",
  "duplicate_hint": null                 // another issue # this looks like a dup of, or null — NOT authoritative
}
```
Workers see only their own batch, so `duplicate_hint` is a *hint*, not a decision. Cross-batch duplicate reconciliation happens centrally in step 5 — do not ask workers to resolve duplicates or compute `supersedes`.

**c. Already-fixed verification (1 agent, Opus) — after (b).** Collect every issue the categorizers marked `Already Fixed` OR `Close Candidate` and spawn one Opus agent to double-check each against `$BRANCH` — confirm the code path really changed / the fix really landed. Downgrade any that don't hold up back to Should Keep or Unsure. This is the guardrail against closing something still broken.

### 5. Synthesize the shortlist (main session)

First **reconcile duplicates centrally** — the main session is the only place that sees every worker's verdicts. Cluster issues by the `duplicate_hint` signals plus title/subsystem similarity; within each cluster pick the canonical issue (usually the oldest with the most context) and set `is_duplicate: true` / `duplicate_of: <canonical>` on the rest, and `supersedes: [<dup numbers>]` on the canonical. A duplicate keeps its own "is the bug real?" category.

Then build the headline deliverable — **10 to 20 issues to prioritize**, applying these decision guidelines (from the team):

- **AI must be able to do it autonomously.** Keep only `ai_autonomy: autonomous`. Discard feature requests (`needs-product`), complex bugs needing human supervision (`needs-human`), and anything needing design or copy (`needs-design`).
- **Rank by impact × priority**, not by noise. A QA engineer having filed the ticket means the bug is *real*, but says nothing about priority — do not upweight a ticket just because QA reported it.
- Prefer issues in `Should Keep` (real + unfixed). Naturally exclude `Already Fixed`.
- If fewer than 10 clean candidates exist, say so rather than padding the list.

### 6. Build / update the artifact

Titled **"<DISPLAY_NAME> Issue Backlog Audit"**. First call the **Artifact tool with `action: "list"`** and match on the stable substring **"Issue Backlog Audit"** for this `REPO` (don't require an exact `DISPLAY_NAME` prefix — it can drift between runs and cause a silent duplicate). If found, update it in place via its `url`; otherwise publish new.

In re-triage mode (issue-numbers passed): WebFetch the existing artifact first, preserve untouched rows, replace only the re-checked ones. If no existing artifact exists in re-triage mode, publish a fresh one scoped to just those issues and note that the rest weren't audited.

Write the HTML to `$TEMP_ROOT` (or the session scratchpad), then publish. Structure:
1. **Header** — "<DISPLAY_NAME> · Issue Backlog Audit" · `REPO`, verified branch, today's date, scope (all / filter used), issues audited.
2. **⭐ Prioritized shortlist** (top section) — the 10–20 AI-autonomous quality wins, ranked, each with issue link, title, impact/priority, and a one-line "why the AI can own this".
3. **Full triage table** — every in-scope issue grouped by category (Close Candidate + Already Fixed first — the cleanup wins). Columns: number (linked), title, category badge, AI-autonomy badge, impact/priority, reason, evidence, dup flag.
4. **Summary bar** — the four category counts + the "safe to close" total (Already Fixed + Close Candidate) + shortlist size.
5. **Footnote** — branch/date the audit reflects; reminder it recommends but never performs closures.

**Categories:** *Should Keep* = real & still present. *Already Fixed* = provably resolved on `$BRANCH`. *Close Candidate* = should close for another reason (obsolete, out of scope, superseded, won't-do). *Unsure* = can't settle from static code (needs live repro / API / runtime data — not a lazy shortcut). Duplicates keep the "is the bug real?" category and add `is_duplicate`/`duplicate_of`; the canonical issue records `supersedes`.

**Design:** theme-aware (light + dark). Category colors — Should Keep = blue `#4393d0`, Already Fixed = green `#2ea043`, Close Candidate = amber `#c9930a`, Unsure = muted grey. AI-autonomy badge greened when `autonomous`. Mono font stack `'Cascadia Code','SF Mono','Fira Mono','Menlo',monospace`. Wide table scrolls inside its own `overflow-x:auto` container. **Favicon:** 🗂️. **File:** `backlog-triage.html`.

### 7. Report back

Give the user:
- The artifact link (created new / updated in place).
- The **prioritized shortlist** inline — issue number + title + one-line why, ranked.
- The four category counts and the safe-to-close set (Already Fixed + Close Candidate numbers with reasons).
- One line on what stayed **Unsure** and what would settle it.

## Turn-budget safety

A full backlog can be large. If the run risks exhausting the budget before finishing, publish the artifact with what's triaged so far, mark the remainder `Unsure` → "not yet audited (run cut off)", and report the cutoff. A partial-but-published audit beats a complete-but-lost one.
