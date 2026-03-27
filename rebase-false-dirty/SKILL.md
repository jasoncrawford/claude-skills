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

## Step 5a — Check for stale rebase-merge directory

If the apply backend fails mid-rebase and subsequent `--abort` doesn't cleanly reset, there may be a leftover `rebase-merge` directory inside the worktree's git state directory. Check for it:

```bash
find /workspace/.git/worktrees/$WORKTREE_NAME -name "rebase-merge" -o -name "rebase-apply"
```

If found, delete it manually:
```bash
rm -rf /workspace/.git/worktrees/$WORKTREE_NAME/rebase-merge
rm -rf /workspace/.git/worktrees/$WORKTREE_NAME/rebase-apply
```

After deleting, verify HEAD is still on the feature branch (not detached):
```bash
git -C $WORKTREE status
git -C $WORKTREE branch
```

If HEAD is detached, re-attach it: `git -C $WORKTREE checkout <branch-name>`

## Step 6 — If rebase keeps failing, use git merge as a fallback

When all rebase approaches fail, `git merge origin/main` is a valid escape hatch:

```bash
git -C $WORKTREE merge origin/main
git -C $WORKTREE push
```

This brings the branch up-to-date with main via a merge commit. GitHub's `strict: true` branch protection accepts this — it only requires the branch to be a descendant of the base, not that the history be linear.

**Important:** After pushing a merge-from-base commit to a PR branch, GitHub Actions may NOT automatically trigger a new CI run (no `synchronize` event fires). If CI is required to pass before merge and the `statusCheckRollup` shows empty, push an empty commit to force it:

```bash
git -C $WORKTREE commit --allow-empty -m "chore: trigger CI"
git -C $WORKTREE push
```

## Step 7 — If nothing works, stop and ask

If the branch still can't be brought up-to-date, collect diagnostics and report to the user:
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
fetch → sync main → merge-backend rebase → apply-backend rebase → check stale rebase-merge dir → git merge fallback → push (+ empty commit if CI doesn't trigger)
```

Most of the time, Step 2 (syncing local main) is all that's needed. The apply-backend fallback is only necessary when your branch and main both modified the same files. If all rebase approaches fail, `git merge` is the escape hatch.
