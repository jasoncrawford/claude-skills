---
name: branch-discipline
description: Use when making any code changes, implementing features, fixing bugs, or modifying files that will be committed — all work happens on branches, never on main
---

# Branch Discipline

## Overview

All changes go through branches and pull requests. Nothing is ever committed directly to main. The only way code reaches main is through a PR that passes CI. No exceptions — not for "quick fixes," not for "just one line," not for config changes.

## The Rule

```
main is read-only. Always.
```

## Workflow

1. **Create a branch and worktree** from main before making any changes
2. **Do all work** in the worktree — commits, edits, iterations
3. **Push the branch** and open a PR with auto-merge enabled
4. **CI runs** automatically on the PR
5. **CI passes** → PR auto-merges → branch is deleted
6. **CI fails** → fix on the same branch, push again, CI re-runs

```bash
# Start work — create branch + worktree
git branch short-description main
git worktree add /tmp/worktree-short-description short-description

# ... make changes in /tmp/worktree-short-description, commit ...

# Push and create PR with auto-merge
cd /tmp/worktree-short-description
git push -u origin short-description
gh pr create --title "Short description" --body "..."
gh pr merge --auto --merge

# After merge — clean up
cd /path/to/main/checkout
git worktree remove /tmp/worktree-short-description
git branch -d short-description
```

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
git rebase main
git push --force-with-lease
```

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
| "I'm the only one working on this" | You're not — AI agents may be working in parallel, and future-you is a collaborator too. |
| "I need this on main right now" | Push the branch, CI runs in minutes. If it's truly urgent, the test suite is your friend, not your obstacle. |
| "Branch protection is slowing me down" | Auto-merge makes it zero-effort. The "slowness" is CI doing its job. |

## Checklist

- [ ] Currently on a branch, not main
- [ ] Working in a worktree, not the main checkout
- [ ] Branch is based on latest main
- [ ] PR created with auto-merge enabled
- [ ] CI passing before merge (enforced by branch protection)
- [ ] Worktree removed and branch deleted after merge
