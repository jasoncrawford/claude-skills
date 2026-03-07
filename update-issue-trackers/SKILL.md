---
name: update-issue-trackers
description: Use when fixing a bug tracked in GitHub Issues - ensures the issue is closed with a comment when resolved
---

# Update Issue Trackers

## Overview

When fixing a bug tracked in GitHub Issues, close the issue with a comment noting the commit hash. Don't leave stale issues open.

## Applies When

- Your fix addresses a tracked GitHub issue, even partially
- You discover a tracked issue was already fixed or not actually an issue
- A commit message references `fixes #N` or `closes #N`

## Actions

- Close the issue: `gh issue close <number> --comment "Fixed in <commit-hash>"`
- If only partially fixed, add a comment instead: `gh issue comment <number> --body "Partially addressed in <commit-hash>: <what was done>"`
- If not actually an issue, close with explanation: `gh issue close <number> --reason "not planned" --comment "<explanation>"`

## Before Closing

Confirm a test was written that would have caught this bug. If no test was written, note why in the closing comment (e.g., "purely cosmetic," "already covered by existing test X").

## Not Acceptable

- Fixing the code but forgetting to close the issue
- "I'll close it later"
- Leaving issues open when they're resolved
