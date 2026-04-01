---
name: commit-detail
description: Stage all changes and create a detailed git commit. Use when the user asks to commit, says "commit-detail", or wants to stage and commit everything at once.
disable-model-invocation: true
context: fork
model: haiku
---

## Context

- Current branch: !`git branch --show-current`
- Git status: !`git status`
- All changes (staged and unstaged): !`git diff HEAD`
- Recent commits: !`git log --oneline -10`

## Rules

- No asking. Stage everything and commit immediately.
- Use a single-line commit message subject.
- Follow the project's existing commit style.
- If there is nothing to commit (clean working tree), just report that to the user and stop.

## Steps

1. Stage all changes
   - Run `git add -A`

2. Create the commit message based on context above
   - Only use English.
   - Use a conventional commit prefix: `feat:`, `fix:`, `chore:`, `refactor:`, `docs:`, etc.
   - Write a concise subject line (under 72 characters).
   - Only add a body if there are genuinely distinct changes worth calling out separately.

3. Execute the commit
   - Run `git commit -m "MESSAGE"`
