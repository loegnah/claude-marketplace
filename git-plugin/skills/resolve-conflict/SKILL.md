---
name: resolve-conflict
description: Resolve git conflicts during rebase or merge by analyzing both branches' changes. Use when the user says "resolve conflict", "fix conflict", "충돌 해결", "rebase conflict", "merge conflict", or when git status shows unmerged paths during rebase/merge. Also trigger when the user is stuck on conflicts or asks for help with conflicting files. This skill resolves conflicts - for reviewing already-resolved conflicts, use review-conflict instead.
---

# Resolve Git Conflicts

Resolve git conflicts during rebase or merge by deeply understanding what each branch intended, then producing a correct resolution that preserves both intents.

## Language

Respond in the same language the user is using. Match the user's language for all output.

## Core Principles

- **Understand before acting**: Never resolve a conflict without first understanding why it happened and what each side intended.
- **Ask when ambiguous**: If both branches made intentional, competing changes to the same logic and it's unclear which should win or how to combine them, stop and ask the user. Do not guess or silently pick one side.
- **No commit by default**: After resolving conflicts, stage the files but do NOT commit or run `git rebase --continue` / `git merge --continue` unless the user explicitly asks. Tell the user what to run next.

## Step 1: Assess the Situation

```bash
git status
```

Determine what's happening:
- **Rebase in progress**: `.git/rebase-merge/` or `.git/rebase-apply/` exists
- **Merge in progress**: `.git/MERGE_HEAD` exists
- **Neither**: No conflict operation in progress — tell the user and stop

Identify all files with conflicts (listed as "both modified" or "Unmerged paths" in git status).

## Step 2: Understand the Context of Each Branch

Understanding what each branch was trying to do is essential for a correct resolution. Don't skip this step.

### For Rebase

```bash
# What branch we're rebasing onto
cat .git/rebase-merge/onto 2>/dev/null || cat .git/rebase-apply/onto 2>/dev/null

# The commit currently being replayed
cat .git/rebase-merge/stopped-sha 2>/dev/null

# What the current commit being rebased was trying to do
git show --stat REBASE_HEAD
git log --format="%B" -1 REBASE_HEAD

# What changed on the target branch that conflicts
git log --oneline REBASE_HEAD..$(cat .git/rebase-merge/onto 2>/dev/null) -- <conflicting_files>
```

### For Merge

```bash
# What our branch changed
git log --oneline MERGE_HEAD..HEAD -- <conflicting_files>

# What their branch changed
git log --oneline HEAD..MERGE_HEAD -- <conflicting_files>
```

## Step 3: Analyze Each Conflicting File

For every conflicting file, do this analysis before attempting resolution:

### 3a. Read the conflict markers

```bash
# See the raw conflict state
git diff -- <file>
```

Read the file and identify all conflict regions (between `<<<<<<<` and `>>>>>>>`).

### 3b. Understand each side's intent

For each conflict region:

```bash
# What our side looks like for this file
git show :2:<file>    # "ours" - the version we're rebasing onto (rebase) or current branch (merge)

# What their side looks like for this file
git show :3:<file>    # "theirs" - the commit being replayed (rebase) or branch being merged (merge)

# What the common ancestor looked like
git show :1:<file>    # "base" - the merge base version
```

Compare all three versions to understand:
1. **What did the base version look like?** (the starting point both branches diverged from)
2. **What did "ours" change from the base?** (and why — check the commit messages)
3. **What did "theirs" change from the base?** (and why)

This three-way comparison is the foundation of a correct resolution. The base version tells you what each side actually modified, not just what they look like now.

### 3c. Classify the conflict

Each conflict region falls into one of these categories:

| Category | Description | Action |
|----------|------------|--------|
| **Non-overlapping** | Both sides changed different things in the same area (e.g., one added a line above, one below) | Combine both changes |
| **Complementary** | Both sides made changes toward the same goal (e.g., both fixed the same bug differently) | Pick the better one or combine, ask if unclear |
| **Competing** | Both sides made intentional, contradictory changes to the same logic | **Ask the user** |
| **One-sided** | Only one side made meaningful changes; the other side's diff is incidental (e.g., formatting, reordering) | Take the meaningful change |

## Step 4: Resolve

For each conflicting file:

1. Edit the file to produce the correct resolution based on your analysis
2. Make sure the resolution:
   - Preserves the intent of both branches where possible
   - Maintains syntactic correctness (no broken imports, missing brackets, etc.)
   - Has NO remaining conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`)
   - Is consistent with the rest of the file and codebase
3. Stage the resolved file:
   ```bash
   git add <file>
   ```

### When to Ask the User

Stop and ask when:
- Two branches made **competing intentional changes** to the same logic and it's not clear how to combine them
- A conflict involves **business logic decisions** where you can't determine the correct behavior from code context alone
- The commit messages or code context are **insufficient** to determine what was intended
- Resolving one way would **break functionality** from the other branch

Frame your question clearly: explain what each side did, why they conflict, and what the options are.

## Step 5: Verify and Report

After resolving all conflicts:

```bash
# Confirm no conflict markers remain
git diff --check --cached

# Confirm no unmerged paths remain
git status
```

Present a summary to the user:

```
## Conflict Resolution Summary

### Context
- Operation: rebase / merge
- Base branch: <branch>
- Source: <branch or commit>

### Resolved Files
For each file:
- **<filename>**
  - Ours: <what our side changed and why>
  - Theirs: <what their side changed and why>
  - Resolution: <what you did and why>

### Next Steps
- Review the changes: `git diff --cached`
- Continue rebase: `git rebase --continue`
- Continue merge: `git commit`
- Abort if needed: `git rebase --abort` / `git merge --abort`
```

Do NOT run the "next steps" commands. Just tell the user what to do.

## Important Reminders

- The `ours`/`theirs` semantics flip between merge and rebase. During **rebase**, "ours" is the branch you're rebasing **onto** (the target), and "theirs" is your commit being replayed. During **merge**, "ours" is your current branch and "theirs" is the branch being merged in. Always verify which is which by checking commit hashes before resolving.
- When analyzing the base version (`:1:<file>`), if it fails, the file may be newly added on both branches. In that case, treat both versions as entirely new content and combine them logically.
- After resolving, always double-check with `git diff --check --cached` to catch whitespace issues or remaining markers.
