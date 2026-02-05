---
description: Review conflict resolution after rebase or merge to verify changes are correct
---

Review the conflict resolution that the user has made during a rebase or merge operation.

## Language

Respond in the same language the user is using. Match the user's language for all output.

## Important

- You are NOT resolving conflicts - the user has already resolved them
- Your job is to REVIEW the resolution and identify potential issues
- Understanding the CONTEXT of why conflicts occurred is crucial for a good review

## Steps

1. First, check the current git status to understand the state:
   ```bash
   git status
   ```

2. Identify if we're in the middle of a rebase or merge:
   - Look for `.git/MERGE_HEAD` (merge in progress)
   - Look for `.git/rebase-merge/` or `.git/rebase-apply/` (rebase in progress)

3. **Understand the conflict context by analyzing both branches:**

   For rebase:
   ```bash
   # Get the target branch (upstream)
   cat .git/rebase-merge/onto 2>/dev/null || cat .git/rebase-apply/onto 2>/dev/null

   # Get the current commit being rebased
   cat .git/rebase-merge/stopped-sha 2>/dev/null

   # See commits in the branch being rebased
   git log --oneline HEAD...REBASE_HEAD 2>/dev/null

   # See commits in the target branch that might have caused conflicts
   git log --oneline REBASE_HEAD..$(cat .git/rebase-merge/onto) 2>/dev/null
   ```

   For merge:
   ```bash
   # Compare commits between current branch and merge target
   git log --oneline HEAD...MERGE_HEAD

   # See what changed in our branch
   git log --oneline MERGE_HEAD..HEAD

   # See what changed in their branch
   git log --oneline HEAD..MERGE_HEAD
   ```

4. **Analyze the history of conflicting files:**
   ```bash
   # See recent changes to conflicting files in both branches
   git log --oneline -5 --follow -- <conflicting_file>
   ```

5. Find the files that had conflicts and were resolved:
   ```bash
   git diff --cached --name-only
   ```

6. For each resolved file, analyze the changes:
   ```bash
   # Show what was changed
   git diff --cached <file>
   ```

7. Compare with both sides of the conflict to understand intent:
   - For merge: compare with `HEAD` (ours) and `MERGE_HEAD` (theirs)
   - For rebase: compare with the original commit being rebased and the target branch

## Review Checklist

When reviewing the resolution, check for:

1. **Context understanding**: Explain WHY the conflict occurred based on commit history
   - What was the intent of changes in each branch?
   - Do both intents get preserved in the resolution?
2. **Logical correctness**: Does the merged code make sense logically?
3. **Missing code**: Was any important code accidentally removed?
4. **Duplicate code**: Is there any accidentally duplicated logic?
5. **Syntax errors**: Are there any obvious syntax issues?
6. **Import/dependency issues**: Are all necessary imports present?
7. **Conflict markers left behind**: Check for any remaining `<<<<<<<`, `=======`, or `>>>>>>>` markers
8. **Inconsistent state**: Does the resolution maintain consistent state across the codebase?
9. **Intent preservation**: Does the resolution honor the goals of BOTH branches?

## Output Format

Provide a clear summary:

1. **Conflict Context**:
   - Briefly explain what each branch was trying to do
   - Why the conflict occurred (both branches modified same area, etc.)
2. **Files Reviewed**: List all files that were checked
3. **Status**: Overall assessment (OK / Issues Found)
4. **Analysis per file**:
   - What was "ours" trying to do?
   - What was "theirs" trying to do?
   - Does the resolution preserve both intents?
5. **Issues** (if any):
   - File name
   - Line number(s)
   - Description of the issue
   - Suggested fix
6. **Recommendations**: Any suggestions for the user

If everything looks good, confirm that the resolution appears correct and the user can proceed with:
- `git rebase --continue` (for rebase)
- `git commit` (for merge)
