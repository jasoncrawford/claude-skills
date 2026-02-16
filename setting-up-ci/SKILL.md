---
name: setting-up-ci
description: Use when starting a new project, adding tests to a project without CI, or when a project has tests that only run locally
---

# Setting Up CI

## Overview

Set up CI at the very beginning of a project — alongside your first test, not after you have a hundred. CI ensures tests actually run every time, as part of an automated pipeline rather than depending on anyone's habits or discipline. It also enables **branch discipline** and **frees up local machines**.

## When to Use

- Starting a new project (set up CI with your first test)
- Project has tests but no CI pipeline
- Tests only run on the developer's machine
- Moving from solo development to collaboration

## Why Early

### 1. Tests Run Every Time

Tests only catch bugs if they actually run. Without CI, running tests before merging depends on the developer (or agent) remembering to do it, having time, and not skipping the slow ones. CI makes this automatic and non-negotiable: nothing reaches main without passing the full suite. This is especially important when AI agents are producing changes — the pipeline enforces quality regardless of who or what authored the code.

### 2. Branch and PR Discipline

Without CI, there's no reason to use branches and pull requests. Everything happens on main because there's nothing to gate on. This means:

- Changes blend together with no clear boundaries
- There's no audit trail of what changed and why
- Reverting a bad change means surgical `git revert` instead of reverting a merge commit
- Multiple workstreams (features, fixes, experiments) collide in the same branch

CI gives branches a purpose: every branch gets tested before it merges. With branch protection requiring CI to pass, the workflow becomes:

1. Create a branch for each change
2. Push and open a PR
3. CI runs automatically
4. When tests pass, the PR merges (auto-merge makes this zero-friction)
5. Branch is deleted

This keeps changes organized, isolated, and reversible. Each merge commit is a clean unit of work with a description, a diff, and proof that tests passed. This discipline is valuable even for solo developers — especially when working with AI agents that may be producing multiple changes in parallel.

### 3. Tests in the Cloud

Running a full test suite locally blocks the developer (or agent) for minutes. In CI, tests run on someone else's machine. The developer pushes, moves on to the next task, and gets notified when tests pass or fail. This matters most when:

- Test suites grow beyond a few seconds (browser tests, e2e tests, sync tests)
- AI agents are producing changes — an agent can push a branch, move on, and come back to fix failures later
- Multiple branches are in flight — each gets tested independently and in parallel

Local testing becomes optional spot-checking rather than a mandatory gate. The real gate is CI.

## Setup Checklist

### 1. GitHub Actions Workflow

Minimal working workflow:

```yaml
name: Tests

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 15

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - run: npm ci

      - run: npm test
        env:
          CI: true

      - uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: test-results
          path: test-results/
          retention-days: 7
```

Start with this and iterate. Add services (databases, etc.) as your project needs them.

### 2. Branch Protection

Once CI passes for the first time:

- Require the test status check to pass before merging
- Enable "strict" (branch must be up-to-date with base before merging)
- Enable auto-merge so PRs merge automatically when checks pass
- Enforce merge commits only (`--no-ff`) to preserve branch history as discrete units

### 3. Auto-Merge

Auto-merge is the key to making this workflow low-friction. Without it, every PR needs a manual "click merge" step. With it:

- Push a branch, open a PR with auto-merge enabled
- Walk away (or start the next task)
- CI passes → PR merges → branch is deleted → done

For solo developers and AI-assisted workflows, this is the difference between CI being a chore and CI being invisible infrastructure.

## Cross-Platform Pitfalls

Your dev machine is macOS. CI is Linux. These differences are trivial individually but annoying in bulk. Fix them early when you have few tests, not later when you have many.

### Keyboard Shortcuts (macOS vs Linux)

macOS `Meta` = Cmd. Linux `Meta` = the Windows/Super key (nobody presses it).

**In app code** — accept both:

```ts
if (e.key === 'k' && (e.metaKey || e.ctrlKey)) { ... }
```

**In tests** — use a platform-aware constant:

```ts
const CMD = process.platform === 'darwin' ? 'Meta' : 'Control';
await element.press(`${CMD}+a`);
```

### File Watchers vs Test Artifacts

Dev servers (Vite, vercel dev) watch the project directory. Test frameworks create and delete temp directories (`test-results/.playwright-artifacts-*/`). If the dev server scans a directory that was just deleted, it crashes.

**Fix:** Put test output outside the project in CI:

```ts
// playwright.config.ts
...(process.env.CI ? { outputDir: '/tmp/test-results' } : {}),
```

### Dev Server Auth in CI

Tools like `vercel dev` need explicit `--token` flags in CI — env vars alone aren't sufficient:

```ts
const TOKEN_FLAG = process.env.VERCEL_TOKEN
  ? ` --token=${process.env.VERCEL_TOKEN}` : '';
```

Store the token as a GitHub repository secret.

### Case Sensitivity

macOS filesystem is case-insensitive. Linux is case-sensitive. `import './Utils'` works locally but fails in CI if the file is `utils.ts`.

## Local Services in CI

If tests need a database, run it in CI too:

```yaml
- uses: supabase/setup-cli@v1
  with:
    version: latest
- run: supabase start
# ... run tests ...
- if: always()
  run: supabase stop
```

Always clean up with `if: always()`.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Waiting until "the test suite is mature" to add CI | Set up CI with your first test. The workflow benefits start immediately. |
| No branch protection after CI is set up | Enable it immediately. CI without enforcement is a suggestion, not a gate. |
| Skipping e2e/sync tests in CI "because they're slow" | Include them. Slow tests are exactly the ones you want off your laptop. |
| Not enabling auto-merge | Auto-merge is what makes the branch workflow frictionless. |
| Running full test suite locally before every push | Push to CI instead. Spot-check locally, gate on CI. |
| Doing everything on main because "it's just me" | Solo projects benefit from branch discipline too — especially with AI agents. |
| Not uploading artifacts on failure | Always upload test results — you can't SSH into CI to debug. |

## Checklist

- [ ] CI workflow runs on every PR to main
- [ ] First CI run passes green
- [ ] Branch protection requires CI to pass
- [ ] Auto-merge enabled
- [ ] Test artifacts uploaded on failure
- [ ] All tests included in CI (especially slow ones — that's the point)
- [ ] Local services started and stopped in CI (with `if: always()` cleanup)
- [ ] Keyboard shortcuts use `(metaKey || ctrlKey)` in app code and platform-aware `CMD` in tests
- [ ] Test output directory outside project root in CI (avoids file watcher crashes)
