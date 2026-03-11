---
name: branch-discipline
description: Use when editing files in projects that have CI or PRs set up, or otherwise use branches, or whenever multiple agents might work in parallel
---

# Branch Discipline

## When This Skill Applies

This skill applies when a project has:
- GitHub CI configured (Actions workflows, branch protection rules), **or**
- Pull requests as the standard integration path, **or**
- Multiple agents working in parallel on independent tasks

**It may not apply** in the earliest experimental or prototype stages of a project, or in small repos that have no automated tests and consist mainly of config rather than code — situations where the overhead of branches and PRs adds friction without providing meaningful safety. When in doubt, ask the user.

## Overview

All changes go through branches and pull requests. Nothing is ever committed directly to main. The only way code reaches main is through a PR that passes CI. No exceptions — not for "quick fixes," not for "just one line," not for config changes.

## The Rule

```
main is read-only. Always.
```

## Workflow

1. **Create a branch and worktree** from main before making any changes
2. **Do all work** in the worktree — commits, edits, iterations
3. **Push the branch and open a PR immediately** — but do not auto-merge
4. **CI runs** automatically on the PR
5. **User reviews** → merges when satisfied → branch is deleted
6. **CI fails** → fix on the same branch, push again, CI re-runs

**Never enable auto-merge.** Always leave the PR open for the user to review and merge.

**Create the PR proactively.** Don't wait to be asked — when the branch is ready, push it and open the PR as part of completing the work.

```bash
# Start work — create branch + worktree
git branch short-description main
git worktree add /tmp/worktree-short-description short-description

# ... make changes in /tmp/worktree-short-description, commit ...

# Push and create PR — do NOT run gh pr merge --auto --merge
cd /tmp/worktree-short-description
git push -u origin short-description
gh pr create --title "Short description" --body "..."
# If this PR resolves a GitHub issue, include a closing keyword in the body:
# "Closes #42" / "Fixes #42" / "Resolves #42"
# A plain link ("see #42") does NOT auto-close the issue on merge.

# After user merges — clean up
cd /path/to/main/checkout
git worktree remove /tmp/worktree-short-description
git branch -d short-description
```

## Always Use Worktrees

**Every branch MUST have its own worktree.** Never work on a branch by switching branches in the main checkout with `git checkout` or `git switch`. This is the single most important rule after "main is read-only."

Why:
- **Switching branches in a shared working directory causes cross-contamination** — uncommitted changes bleed between branches, the index gets confused, and you end up debugging git instead of shipping code
- **Worktrees are fully isolated** — each branch gets its own working directory with its own files and index, so nothing can interfere
- **The main checkout stays pristine** — always pointing at main, always clean, always available to start new work or read reference code

If you find yourself running `git checkout <branch>` or `git switch <branch>` to do work, stop — you're doing it wrong. Create a worktree instead.

## Never Cherry-Pick or Force-Push

**`git cherry-pick` and `git push --force` (including `--force-with-lease`) are anti-patterns. Do not use them.**

- **Cherry-pick** creates duplicate commits with different SHAs, fractures history, and causes confusion about what's actually been merged. If you need a commit on a different branch, you've got a workflow problem — fix the workflow.
- **Force-push** rewrites shared history. It destroys other people's (and other agents') references to commits, causes divergence that requires manual recovery, and is the #1 cause of lost work in collaborative repos.

Both are symptoms of the same root problem: working without proper worktree isolation, then trying to fix the resulting mess with dangerous git operations. If you use worktrees correctly, you will never need either one.

**What to do instead:**
- Need to rebase? That's fine — `git rebase main` in a worktree, then `git push` (which works because no one else is force-pushing to your branch)
- Got into a bad state? Ask the user before taking destructive action. Explain what went wrong and propose a safe fix.

## Branch Naming

Keep it short and descriptive. Use the work being done, not metadata:

- `add-hyperlink-support`
- `fix-cursor-position`
- `ci-setup`
- `issue-42-sync-retry`

## When CI Fails

Fix on the same branch. Do not:
- Merge with failing CI
- Bypass branch protection
- Push the fix directly to main "to unblock things"

The branch stays open until CI is green. This is the point — the gate works because it's non-negotiable.

## Why Worktrees

Git worktrees let multiple branches be checked out simultaneously in different directories. This is critical for parallel work:

- **Multiple agents** can work on different branches at the same time without interfering with each other — each has its own worktree
- **The main checkout stays clean** — you can read main, start new branches, or review work while agents are mid-change in their worktrees
- **No stashing or context switching** — each worktree is a fully independent working directory

Without worktrees, only one branch can be active at a time. With them, every branch gets its own directory and agents can run truly in parallel.

Worktrees go in `/tmp/worktree-*` by convention — outside the project directory, easy to find, automatically cleaned up on reboot.

## Multiple Workstreams

When working on several things at once (common with AI agents):

- Each piece of work gets its own branch and worktree
- Branches are independent — don't stack branches on branches
- Each branch is based on current main
- If main moves forward (another PR merged), rebase before merging:

```bash
cd /tmp/worktree-my-branch
git fetch origin main
git rebase origin/main
git push
```

Note: `git push` (no `--force`) works here because your branch's worktree is isolated — no one else is pushing to your feature branch, so the rebase doesn't cause a conflict with the remote. If `git push` is rejected, something unexpected happened — investigate rather than force-pushing.

## What Belongs on a Branch

Everything:
- Features
- Bug fixes
- Refactors
- Config changes
- Test additions
- Documentation updates
- Dependency upgrades
- "One-line fixes"

There is no change small enough to skip the branch workflow. Small changes are fast to branch, fast to PR, and fast to pass CI.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "It's just a one-line fix" | One-line fixes can break things. Branch takes 30 seconds. |
| "CI is slow, I'll push to main and fix later" | "Fix later" means "forget and ship broken." |
| "I'm the only one working on this" | Once CI exists, you're not — AI agents may be working in parallel, and future-you is a collaborator too. |
| "I need this on main right now" | Push the branch, CI runs in minutes. If it's truly urgent, the test suite is your friend, not your obstacle. |
| "Branch protection is slowing me down" | The "slowness" is CI doing its job. Push, open a PR, let the user review. |

## Checklist

- [ ] Currently on a branch, not main
- [ ] Working in a worktree, not the main checkout
- [ ] Branch is based on latest main
- [ ] Tests written for any new behavior or bug fixes (see `no-skipped-tests`)
- [ ] All tests pass with zero skipped
- [ ] PR created — auto-merge NOT enabled; left for user to review and merge
- [ ] PR body includes `Closes #N` / `Fixes #N` / `Resolves #N` if this resolves a GitHub issue (a plain link does not auto-close)
- [ ] CI passing before merge (enforced by branch protection)
- [ ] Worktree removed and branch deleted after merge
