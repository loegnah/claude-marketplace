---
name: commit-simple
description: Quick commit with auto-generated message based on project's commit style. Use when the user asks to commit, make a commit, or says "commit-simple".
disable-model-invocation: true
context: true
---

## Context

- Recent commit style: !`git log --oneline -10`
- Staged changes: !`git diff --cached`

## Rules

- No ask. Just do commit.
- Commit message must be a single line only
- Follow the project's existing commit style as closely as possible
- Always get user confirmation before committing

## Steps

1. Create commit message based on the context above
   - Only use English.
   - Use prefix like "feat:", "chore:", "fix:", and so on.
   - Write very simply (just one line).
   - Only when absolutely necessary, summarize the key points of the revised content one line at a time at the bottom.

2. Execute the commit
   - Run `git commit -m "MESSAGE"`
