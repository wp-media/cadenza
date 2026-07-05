---
name: issue
description: Turn raw context, a thread, or a rough note into a well-structured GitHub issue.
---

# Issue Writer

Turns raw discussion, a pasted thread, or a rough note into a well-structured GitHub issue — with auto-detected labels and confirmation before anything is created.

## Project identity

Project identity is auto-detected (see AGENTS.md §3) — no config file.
This skill uses: `{REPO}`.

## Steps

1. Parse the user's raw input — pasted discussion, thread, note, or `$ARGUMENTS`. If it references an existing issue number (e.g. "update issue #6040"), pass that along so the agent edits in place instead of creating a new issue.

2. Spawn `issue-writer` as a sub-agent, passing the raw input plus `{REPO}`.

3. Return the issue URL once confirmed and created (or the draft body plus selected labels if not yet confirmed).
