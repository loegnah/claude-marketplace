---
name: rebase-worktrees
description: This skill should be used when the user asks to "rebase worktrees", "sync worktrees", "update all worktrees", "rebase all worktrees onto main", or wants to rebase multiple git worktrees onto a target branch with package install and .env synchronization.
disable-model-invocation: true
context: true
---

# Rebase Worktrees

Rebase all non-main git worktrees onto a target branch, run package installs, and synchronize .env keys.

## Arguments

- **target branch** (optional): Branch to rebase onto. If omitted, use the main worktree's current branch.

## Execution Steps

### Step 1: Collect Worktree Information

Run `git worktree list --porcelain` to gather all worktrees. Identify the main worktree (the first entry, which has no `branch` line prefixed with a worktree path containing `.claude/worktrees/`). Record the main worktree path and its branch.

Exclude the main worktree from the rebase targets. If no other worktrees exist, inform the user and stop.

### Step 2: Determine Target Branch

- If a branch argument was provided, use it as the rebase target.
- Otherwise, extract the main worktree's current branch from the porcelain output and use it.

The target is always a **local branch**, not a remote-tracking branch. Do not run `git fetch`.

### Step 3: Rebase Each Worktree

For each non-main worktree, execute the following sequence:

```bash
git -C <worktree_path> rebase <target_branch>
```

**On conflict:**

1. Run `git rebase --abort` to restore the worktree to its pre-rebase state.
2. Add this worktree to the failure list with its path and branch name.
3. Continue to the next worktree.

**On success:**

1. Add this worktree to the success list.
2. Continue to the next worktree.

### Step 4: Package Install

For each successfully rebased worktree, detect the package manager by checking for lockfiles in the worktree root and run the appropriate install command:

| Lockfile | Command |
| --- | --- |
| `bun.lockb` or `bun.lock` | `bun install` |
| `pnpm-lock.yaml` | `pnpm install` |
| `yarn.lock` | `yarn install` |
| `package-lock.json` | `npm install` |

If no lockfile is found, skip package install for that worktree. If multiple lockfiles exist, use the first match in the priority order above.

Report any install failures but do not abort the overall process.

### Step 5: Sync .env Keys

1. Check if `.env` exists in the main worktree root. If not, skip this step entirely.
2. Parse the main worktree's `.env` file and extract all key names (lines matching `KEY=...`, ignoring comments and blank lines).
3. For each successfully rebased worktree:
   - If `.env` does not exist in the worktree, report it and skip.
   - Parse the worktree's `.env` and extract key names.
   - Compare key sets:
     - **Added keys**: Keys present in main but missing in the worktree.
     - **Removed keys**: Keys present in the worktree but missing in main.
   - If differences exist, present them to the user via `AskUserQuestion` and ask whether to update the worktree's `.env`:
     - For added keys, copy the key-value pair from the main worktree's `.env`.
     - For removed keys, remove the line from the worktree's `.env`.
   - If the user approves, apply the changes. Otherwise, skip.

### Step 6: Report Results

Present a final summary:

- **Target branch**: The branch all worktrees were rebased onto.
- **Successful**: List of worktree paths and their branches that were rebased successfully.
- **Failed**: List of worktree paths and their branches that failed due to conflicts.
- **Package install**: Status per worktree (success / failed / skipped).
- **.env sync**: Status per worktree (updated / skipped / no .env).

## Important Notes

- Always abort on conflict — never leave a worktree in a mid-rebase state.
- Process all worktrees even if some fail — do not stop on first failure.
- Run package install only on successfully rebased worktrees.
- The .env sync compares **keys only**, not values. Copy values from the main worktree when adding new keys.
