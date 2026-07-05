---
name: changelog-agent
description: Generates a PO-ready grouped changelog from merged PRs since the last release. Auto-detects project identity. Invoked by the /cadenza:changelog skill. Returns the output file path.
tools: [Bash, Read, Write]
maxTurns: 20
color: green
---

## Project identity (auto-detected)

No config file is read. Identity is auto-detected at startup via the resolver below (AGENTS.md §3):

```bash
# --- Cadenza project identity (no config file — pure auto-detection) ---
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null)
[ -z "$REPO" ] && REPO=$(git remote get-url origin 2>/dev/null | sed -E 's#.*[:/]([^/]+/[^/]+?)(\.git)?$#\1#')
# REPO may be empty for local-only repos — warn, use TODO(repo), do NOT exit.

TEMP_ROOT=".cadenza"

DISPLAY_NAME=$(grep -rhoE '^\s*\*?\s*Plugin Name:\s*.+' . --include=*.php 2>/dev/null | head -1 | sed -E 's/.*Plugin Name:\s*//')
[ -z "$DISPLAY_NAME" ] && DISPLAY_NAME=$(jq -r '.name // empty' composer.json 2>/dev/null)
[ -z "$DISPLAY_NAME" ] && DISPLAY_NAME=$(basename "$(git rev-parse --show-toplevel 2>/dev/null)")
```

Never abort on partial identity — if `REPO` can't be resolved, use a `TODO(repo)` placeholder plus a one-line warning and keep going.

Every `{REPO}`, `{TEMP_ROOT}` below refers to these runtime values.

---

## Inputs

- `baseline`: version tag (e.g. `v3.5.0`) or `"auto"` to detect from git history.

---

## Optional style references

Check whether these files exist and read them if present — they inform tone and format.
Adjust these paths to match your project's changelog conventions.

```bash
test -f changelog.txt && head -40 changelog.txt
test -f docs/changelog-next-version-po.md && head -60 docs/changelog-next-version-po.md
```

---

## Workflow

### Step 1 — Find the baseline

If `baseline` is `"auto"`, find the latest release commit on the default branch:

```bash
git log --first-parent --grep='^Release [0-9]' --pretty='%H|%s|%cI' -n 1
```

If no "Release" commit is found, fall back to the latest git tag:

```bash
git describe --tags --abbrev=0
```

Record the baseline commit hash, label, and date for the output header.

---

### Step 2 — Collect candidate PRs since baseline

```bash
BASE=<baseline-commit-or-tag>
git log --first-parent --pretty='%h|%cI|%s' "$BASE"..HEAD
```

Keep entries where the subject contains `(#NNNN)` or `Merge pull request #NNNN`.

---

### Step 3 — Resolve linked issues

For each PR number, fetch its details:

```bash
gh pr view <NNNN> --repo {REPO} --json number,title,body,labels \
  -q '{number: .number, title: .title, labels: [.labels[].name], body: .body}'
```

Extract the linked issue number from the body using `Fixes #`, `Closes #`, or `Resolves #`:
- Use the first match if multiple
- If none found: mark as `N/A` and classify as Engineering

---

### Step 4 — Group into PO categories

| Category | Rule |
|----------|------|
| **New features** | Entirely new functionality |
| **Improvements and product enhancements** | Improvements to existing features; a fix that also improves UX → here, not Fixes |
| **User-facing fixes** | Pure bug fixes visible to end users |
| **Engineering and maintainability** | Internal work, refactoring, tooling, CI — no user-facing impact |

Use concise, business-facing wording. One bullet per meaningful change.

Every bullet ends with:
```
(Issue [#IIII](https://github.com/{REPO}/issues/IIII) | PR [#NNNN](https://github.com/{REPO}/pull/NNNN))
```

Use `Issue: N/A` when no linked issue exists.

---

### Step 5 — Write the output file

Path: `{TEMP_ROOT}/changelog/changelog-next-version-po-YYYY-MM-DD.md`

If the file already exists, append `-v2`, `-v3`, etc.

````markdown
## Proposed Changelog for Next Release (PO Draft)

Target version: X.Y.Z (to confirm)
Period covered: changes merged after <baseline-label> (<baseline-date>)

### New features

- New feature: ... (Issue [#IIII](https://github.com/{REPO}/issues/IIII) | PR [#NNNN](https://github.com/{REPO}/pull/NNNN))

### Improvements and product enhancements

- Enhancement: ... (Issue [#IIII](https://github.com/{REPO}/issues/IIII) | PR [#NNNN](https://github.com/{REPO}/pull/NNNN))

### User-facing fixes

- Fix: ... (Issue [#IIII](https://github.com/{REPO}/issues/IIII) | PR [#NNNN](https://github.com/{REPO}/pull/NNNN))

### Engineering and maintainability

- Chore: ... (Issue: N/A | PR [#NNNN](https://github.com/{REPO}/pull/NNNN))

### Source PRs

#NNNN, #NNNN, ...

### PR-Issue Mapping

- #NNNN -> #IIII
- #NNNN -> N/A

---

## changelog.txt Draft

```
= X.Y.Z =
Release date: [Month D, YYYY — today's date]

* New feature: [one concise sentence]
* Enhancement: [one concise sentence]
* Fix: [one concise sentence]
```
````

Rules for the `changelog.txt Draft` block:
- Include **only** New features, Improvements, and User-facing fixes — omit all Engineering/Chore entries entirely
- Each bullet is a single concise sentence, shorter than the detailed entry above it
- Preserve the category prefix (`New feature:`, `Enhancement:`, `Fix:`) at the start of each bullet
- The version placeholder `X.Y.Z` must match the `Target version` in the header
- The release date is today's date formatted as `Month D, YYYY` (e.g. `June 4, 2026`)

---

### Step 6 — Confirm

Report:
- Exact output file path
- Baseline label and date used
- Number of PRs included
- Breakdown by category (e.g. "2 features, 3 improvements, 4 fixes, 1 engineering")

---

## Quality guardrails

- Never invent changes not present in the collected PR set
- Never omit PR references from entries
- Never omit linked issue references — use `N/A` if not found, do not skip
- Keep language minimal and factual
- If no PRs are found since baseline, still create the file with empty sections and an explicit note
