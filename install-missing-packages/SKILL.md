---
name: install-missing-packages
description: Use when tests, builds, or type-checks fail with "Cannot find package", "Cannot find module", or similar import resolution errors — before diagnosing further or accepting failures
---

# Install Missing Packages

## Overview

When you see module-not-found errors, **run the package manager install command first**. Do not accept these failures as environment issues, pre-existing problems, or worktree quirks until you have confirmed packages are installed.

## When This Applies

- `Cannot find package 'X'`
- `Cannot find module 'X'`
- `Module not found: Error: Can't resolve 'X'`
- `error TS2307: Cannot find module 'X' or its corresponding type declarations`
- Tests fail with import errors, even if the same errors appear on `main`

## The Rule

**Before diagnosing or accepting any module-not-found failure, run the install command for this project's package manager:**

```bash
npm install                      # if package.json / package-lock.json present
yarn install                     # if yarn.lock present
pnpm install                     # if pnpm-lock.yaml present
pip install -r requirements.txt  # if requirements.txt present
```

Then re-run the failing command and confirm whether the error is resolved.

## Common Rationalizations to Reject

| Rationalization | Reality |
|----------------|---------|
| "Same failures exist on main — must be a pre-existing issue" | Main may also have uninstalled packages. Install and check. |
| "These are worktree/environment failures, not my changes" | Install first. Worktree issues and missing packages look identical. |
| "CI will handle it" | CI runs `npm install`. Your local environment should too. |
| "The test failures are unrelated to my work" | Confirm that by installing and re-running. Don't assume. |

## Checklist

- [ ] Saw a module-not-found error
- [ ] Identified the project's package manager and ran its install command
- [ ] Re-ran the failing tests/build
- [ ] Confirmed whether failures remain after install
