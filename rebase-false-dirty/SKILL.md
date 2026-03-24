---
name: rebase-false-dirty
description: Use when git rebase (or pull --rebase) fails with "your local changes to the following files would be overwritten" but git status shows a clean working tree with no local changes
---

# Rebase False-Dirty Failure

## Overview

Git sometimes reports phantom local changes during a rebase when `git status` shows nothing. The cause is not always known. One specific hypothesis: **stale ctime entries in git's index** — file content is unchanged, but filesystem metadata (ctime) has drifted due to OS activity (spotlight, backups, network mounts, etc.), and git mistakes the drift for a modification.

## The Rule

**STOP. Do not take any modifying action until you have diagnosed the problem.**

Never cherry-pick, reset, or force-push to work around this. Gather information first, try the one approved fix, and if it fails, report and wait.

## Step 1 — Diagnose (read-only)

Confirm the symptom before doing anything:

```bash
git status                          # should show clean
git diff                            # should show nothing
git diff --cached                   # should show nothing
git diff HEAD                       # should show nothing
git update-index --refresh 2>&1     # reveals files with stale index entries
```

`git update-index --refresh` is diagnostic here — it reports which files git considers "needs update" based on ctime without actually changing anything committed.

## Step 2 — Try the approved fix

If the ctime hypothesis is correct, running the rebase with `core.trustctime=false` (which tells git to ignore ctime when detecting changes) should fix it:

```bash
git -c core.trustctime=false rebase origin/main
```

If this succeeds, push as normal:

```bash
git push --force-with-lease
```

## Step 3 — If the fix doesn't work: investigate and pause

If the rebase still fails, **do not attempt any further fixes**. Instead:

1. Collect diagnostic information:
```bash
git status
git diff
git diff --cached
git update-index --refresh 2>&1
git log --oneline -5
git log --oneline origin/main -5
```

2. Report the full output to the user.
3. **Stop and wait for instructions.** Do not cherry-pick, reset, or try alternative approaches without explicit permission.

## Why not cherry-pick

Cherry-pick is not a substitute for rebase. It creates duplicate commits with different SHAs, fractures history, and causes confusion when the branch is eventually rebased properly. See the branch-discipline skill.

## Reference

The ctime hypothesis: macOS (and some Linux setups) update ctime on files during spotlight indexing, Time Machine backups, or network mount operations. Git's index caches ctime and uses it as a cheap "is this file dirty?" check. When ctime has changed but content hasn't, `git status` may still report clean (it falls back to content comparison) while rebase's pre-flight check does not. This is one known explanation for the symptom, not a confirmed root cause.

Discussion: https://stackoverflow.com/questions/5074136/git-rebase-fails-your-local-changes-to-the-following-files-would-be-overwritte
