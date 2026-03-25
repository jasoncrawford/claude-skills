---
name: rebase-false-dirty
description: Use when git rebase fails with "your local changes to the following files would be overwritten" but git status shows a clean working tree with no local changes. Common in linked worktrees.
---

# Rebase False-Dirty Failure

## Overview

Git sometimes reports phantom local changes during a rebase when `git status` shows a clean working tree. In linked worktrees (e.g. `/workspace/.claude/worktrees/*`), the primary cause is:

**The main workspace's local `main` branch is behind `origin/main`.** When a PR merges on GitHub, `origin/main` advances but the local `main` ref (checked out in `/workspace`) stays stale. Git's rebase pre-flight check in linked worktrees gets confused by this inconsistency and misidentifies committed feature-branch changes as uncommitted local modifications.

A secondary trigger makes this worse: **when the feature branch and the new main commits both touch the same files**, the default merge rebase backend fails even after syncing local main. The `--apply` backend handles this case but requires careful verification afterward.

## Step 1 — Diagnose

Confirm the symptom:
```bash
git status                          # should show clean
git diff                            # should show nothing
git diff --cached                   # should show nothing
```

Check whether the main workspace is behind origin/main:
```bash
git -C /workspace log --oneline HEAD..origin/main
```

If this shows commits, the main workspace is stale — proceed to Step 2.

## Step 2 — Sync local main

Fast-forward the main workspace. Use `merge --ff-only` (not `pull --rebase`) because the main workspace may have dirty build artifacts that block pull:
```bash
git -C /workspace merge --ff-only origin/main
```

## Step 3 — Retry rebase (merge backend)
```bash
git -C $WORKTREE rebase origin/main
```

If this succeeds, push and you're done:
```bash
git -C $WORKTREE push --force-with-lease
```

## Step 4 — If Step 3 fails, use the apply backend

This happens when the feature branch and new main commits modify the same files. The merge backend's pre-flight check chokes on the overlap; the apply backend uses a patch-based approach that handles it.
```bash
git -C $WORKTREE rebase --abort
git -C $WORKTREE rebase --apply origin/main
```

If `--apply` succeeds cleanly, push and you're done.

## Step 5 — Handling partial failures in --apply

The `--apply` backend may fail on individual commits with "no changes" or "local changes would be overwritten." When this happens, git's 3-way merge fallback may silently drop some or all of that commit's changes while reporting "all conflicts fixed."

**Do NOT blindly run `git rebase --skip`.** Instead:

1. Check which files the failing commit was supposed to touch:
```bash
git show --name-only <commit-hash>
```

2. For each file, verify whether the commit's changes are actually present:
```bash
# What the commit intended to change:
git diff <commit-hash>^ <commit-hash> -- <file>
# What's actually in the working tree vs the prior commit:
git diff HEAD -- <file>
```

3. If changes are missing, manually reapply them to the working tree.

4. Stage everything and continue:
```bash
git -C $WORKTREE add -A
git -C $WORKTREE rebase --continue
```

5. If `--continue` says "No changes — did you forget to use git add?", the commit is genuinely a no-op after your manual fixes were folded into a prior commit. In that case `git rebase --skip` is safe.

6. After the rebase completes, verify the final state matches what you expect before pushing. Run tests.

## Step 6 — If nothing works, stop and ask

If the rebase still fails after all steps above, collect diagnostics and report to the user:
```bash
git -C $WORKTREE status
git -C $WORKTREE diff
git -C $WORKTREE log --oneline origin/main..HEAD
git -C /workspace log --oneline HEAD..origin/main
git -C /workspace status --short
```

Do not cherry-pick, hard reset, or force-push without explicit user permission.

## Why not cherry-pick

Cherry-pick creates duplicate commits with different SHAs, fractures history, and causes confusion when the branch is eventually merged. It is not a substitute for rebase.

## Quick reference
```
fetch → sync main → merge-backend rebase → apply-backend rebase → verify → push
```

Most of the time, Step 2 (syncing local main) is all that's needed. The apply-backend fallback is only necessary when your branch and main both modified the same files.
