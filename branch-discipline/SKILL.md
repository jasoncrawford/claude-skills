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

1. **Pull latest main** before making any changes — `git pull --rebase origin main` in the main checkout
2. **Create a branch and worktree** from main before making any changes
3. **Do all work** in the worktree — commits, edits, iterations
4. **Push the branch and open a PR immediately** — but do not auto-merge
5. **CI runs** automatically on the PR
6. **User reviews** → merges when satisfied → branch is deleted
7. **CI fails** → fix on the same branch, push again, CI re-runs

**Never enable auto-merge.** Always leave the PR open for the user to review and merge.

**Create the PR proactively.** Don't wait to be asked — when the branch is ready, push it and open the PR as part of completing the work.

```
# Start work — pull latest main first (in main checkout)
git pull --rebase origin main

# Create branch + worktree
git worktree add .claude/worktrees/short-description -b short-description

# ... make changes, commit ...

# Push and create PR — do NOT run gh pr merge --auto --merge
git push -u origin short-description
gh pr create --title "Short description" --body "..."
# If this PR resolves a GitHub issue, include a closing keyword in the body:
# "Closes #42" / "Fixes #42" / "Resolves #42"
# A plain link ("see #42") does NOT auto-close the issue on merge.

# After user merges — pull main first, then remove the worktree and branch
git pull --rebase origin main
git worktree remove .claude/worktrees/short-description
git branch -d short-description
```

## Always Use Worktrees

**Every branch MUST have its own worktree.** Never work on a branch by switching branches in the main checkout with `git checkout` or `git switch`. This is the single most important rule after "main is read-only."

**Use `git worktree add` to create worktrees.** Create them in `.claude/worktrees/` inside the project directory:

```bash
git worktree add .claude/worktrees/short-description -b short-description
```

Why:
- **Switching branches in a shared working directory causes cross-contamination** — uncommitted changes bleed between branches, the index gets confused, and you end up debugging git instead of shipping code
- **Worktrees are fully isolated** — each branch gets its own working directory with its own files and index, so nothing can interfere
- **The main checkout stays pristine** — always pointing at main, always clean, always available to start new work or read reference code

If you find yourself running `git checkout <branch>` or `git switch <branch>` to do work, stop — you're doing it wrong. Use `git worktree add` instead.

**Do not use `EnterWorktree` or `ExitWorktree` tools** — they are broken and non-functional. Use `git worktree add` and `git worktree remove` directly.

**`cd` does not persist between Bash tool calls.** When running git commands in a worktree across multiple separate Bash calls, `cd /path/to/worktree` in call N is gone by call N+1. Always use `git -C <worktree-path>` to explicitly target the correct directory:

```bash
# WRONG — cd evaporates after the Bash call ends
cd .claude/worktrees/my-feature && git rebase origin/main
# next Bash call — now git runs in /workspace, not the worktree!
git status

# RIGHT — every call is explicit about its directory
git -C .claude/worktrees/my-feature status
git -C .claude/worktrees/my-feature rebase origin/main

# RIGHT — chain multi-step ops in a single Bash call when order matters
cd .claude/worktrees/my-feature && git status && git rebase origin/main
```

## Never Cherry-Pick

**`git cherry-pick` is an anti-pattern. Do not use it. If you think it is necessary, always get explicit user permission first.**

**Cherry-pick** creates duplicate commits with different SHAs, fractures history, and causes confusion about what's actually been merged. If you need a commit on a different branch, you've got a workflow problem — fix the workflow.

### Depending on an unmerged PR

The most tempting misuse: needing code that exists in another branch that hasn't been merged yet. **Do not cherry-pick those commits.** Instead:

1. **Stop work** on the current task
2. **Identify the blocking PR** — note its number and what it provides
3. **Tell the user** clearly: "I can't proceed until PR #N is merged. It provides [X]. Please merge it and let me know when it's done."
4. **Wait** — do not work around it, do not duplicate the code, do not cherry-pick

This is the correct behavior even if the wait feels inefficient. Cherry-picking from an unmerged PR creates a hidden dependency, risks double-applying changes when the PR eventually merges, and bypasses the review process for that code.

**What to do if `git rebase` fails unexpectedly:** If `git rebase origin/main` fails with "local changes would be overwritten" despite a clean working tree, **stop and ask the user**. Do not use `git cherry-pick` as a substitute. Explain what you tried, what failed, and let the user decide the path forward.

**What to do in other bad states:** Got into a messy git state through some other means? Ask the user before taking any destructive action. Explain what went wrong and propose a safe fix.

## Never Force-Push to Main

**`git push --force` (or `--force-with-lease`) to `main` is an anti-pattern. Do not use this on main.**

**Force-push to main** rewrites shared history. It destroys other people's (and other agents') references to commits, causes divergence that requires manual recovery, and is the #1 cause of lost work in collaborative repos.

**On feature branches, `--force-with-lease` is correct after a rebase.** Rebasing rewrites commit SHAs, so a plain `git push` will be rejected if the branch was already pushed to the remote. Use `--force-with-lease` (not `--force`) — it fails safely if someone else pushed to the branch since your last fetch.

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

## When PR Review Requests Changes

Fix on the same branch. Do not:
- Open a new PR on top of the existing one
- Create a new branch for the review-requested changes
- Commit directly to main

Push to the existing PR's branch and the PR updates automatically. A PR comment requesting changes is a request to update *that PR* — not to open a new one.

## Why Worktrees

Git worktrees let multiple branches be checked out simultaneously in different directories. This is critical for parallel work:

- **Multiple agents** can work on different branches at the same time without interfering with each other — each has its own worktree
- **The main checkout stays clean** — you can read main, start new branches, or review work while agents are mid-change in their worktrees
- **No stashing or context switching** — each worktree is a fully independent working directory

Without worktrees, only one branch can be active at a time. With them, every branch gets its own directory and agents can run truly in parallel.

Worktrees are created with `git worktree add` in `.claude/worktrees/` inside the project directory.

## Multiple Workstreams

When working on several things at once (common with AI agents):

- Each piece of work gets its own branch and worktree
- Branches are independent — don't stack branches on branches
- Each branch is based on current main
- If main moves forward (another PR merged), rebase before merging:

```bash
# From inside the worktree
git fetch origin main
git rebase origin/main
git push --force-with-lease
```

Use `--force-with-lease` (not plain `--force`) — rebasing rewrites commit SHAs so the remote will reject a plain push. `--force-with-lease` is safe: it fails if someone else pushed to your branch since your last fetch, preventing accidental overwrites.

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

## One Concern Per Branch

**Each branch must contain only work related to a single concern.** Never commit work for concern X onto a branch that belongs to concern Y.

This applies especially to spec and plan documents: a spec or plan for feature X must go on its own dedicated branch (e.g., `docs/feature-x`), not on an existing feature branch for unrelated work. Mixing concerns:
- Makes PR review confusing ("why is this plan doc here?")
- Can require destructive cleanup (force-push) to separate them
- Creates misleading history

Before committing any spec, plan, or document: check your current branch. If it belongs to different work, create a new dedicated branch first.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "It's just a one-line fix" | One-line fixes can break things. Branch takes 30 seconds. |
| "CI is slow, I'll push to main and fix later" | "Fix later" means "forget and ship broken." |
| "I'm the only one working on this" | Once CI exists, you're not — AI agents may be working in parallel, and future-you is a collaborator too. |
| "I need this on main right now" | Push the branch, CI runs in minutes. If it's truly urgent, the test suite is your friend, not your obstacle. |
| "Branch protection is slowing me down" | The "slowness" is CI doing its job. Push, open a PR, let the user review. |
| Deleting a branch without pulling main first | `git branch -d` can't verify the branch is merged until local main is up to date. Always `git pull --rebase origin main` before cleanup — otherwise you'll need `git branch -D` (force delete). |

## Checklist

- [ ] Currently on a branch, not main
- [ ] Working in a worktree, not the main checkout
- [ ] `git pull --rebase origin main` run in main checkout before branching
- [ ] Branch is based on latest main
- [ ] Tests written for any new behavior or bug fixes (see `no-skipped-tests`)
- [ ] All tests pass with zero skipped
- [ ] PR created — auto-merge NOT enabled; left for user to review and merge
- [ ] PR body includes `Closes #N` / `Fixes #N` / `Resolves #N` if this resolves a GitHub issue (a plain link does not auto-close)
- [ ] CI passing before merge (enforced by branch protection)
- [ ] Worktree removed and branch deleted after merge
