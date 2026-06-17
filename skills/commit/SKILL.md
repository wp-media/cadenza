---
name: commit
description: Analyse changes, create atomic commits with generated messages, then offer to push. Accepts inline hints — "without <file>", "excluding <path>", "only <path>".
allowed-tools: Bash(git status:*), Bash(git diff:*), Bash(git log:*), Bash(git add:*), Bash(git commit:*), Bash(git push:*), Bash(git rev-parse:*), Bash(git branch:*)
---

## Context

- Git status: !`git status --short`
- Staged + unstaged diff: !`git diff HEAD`
- Recent commits (style reference): !`git log --oneline -8`
- Current branch: !`git branch --show-current`

## Your task

### Step 1 — Parse args

Read `args` for inline hints:
- **Exclusions**: "without `X`", "excluding `X`", "not `X`" → collect paths/patterns to leave unstaged
- **Inclusions**: "only `X`", "just `X`" → scope to those paths only
- **No hint** → include all tracked changes

Echo the parsed exclusion/inclusion list so the user can spot misinterpretations before anything is committed.

### Step 2 — Plan atomic commits

Group the changed files into logical buckets (one concern per commit). Guiding principles:
- Match the commit message style from the recent log — don't invent a new format.
- Files that only make sense together belong in the same commit.
- Apply exclusions/inclusions from Step 1.
- If everything is one coherent change, one commit is fine — don't split artificially.

Print the plan before executing:
```
Planned commits:
  1. fix(cache): correct TTL logic — src/Cache/Driver.php, src/Cache/Store.php
  2. chore(config): update .editorconfig — .editorconfig
  (excluded: .gitignore — per your instruction)
```

### Step 3 — Execute commits

For each group:
1. `git add <explicit file list>` — never `git add .` or `git add -A`
2. `git commit -m "..."` — pass message via heredoc to avoid shell escaping issues

If a commit fails (hook error, nothing staged), report clearly and stop.

After all commits, run `git log --oneline -5` to confirm what landed.

### Step 4 — Offer to push

Ask the user: **"Push these commits to remote?"** with options Yes / No.

- **Yes** → check for upstream with `git rev-parse --abbrev-ref --symbolic-full-name @{u}` (exit 0 = has upstream)
  - Has upstream → `git push`
  - No upstream → `git push -u origin <current-branch>`
- **No** → confirm commits are local only, summarise branch + commit count + top hash.

### Rules

- Never `git add .` or `git add -A` — always name files explicitly.
- Never commit files that look like secrets (`.env`, `credentials*`, `*_secret*`) — warn and skip.
- Never force-push, reset, amend, or rebase.
- Never create empty commits.
- Always print the plan (Step 2) before executing (Step 3).
