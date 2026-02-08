---
name: no-skipped-tests
description: Use when finishing any task that involved running tests - ensures no skipped or failing tests are left behind
---

# No Skipped Tests

## Overview

All tests must pass. Zero skipped. If a test is failing or skipped, investigate and fix it — don't work around it or leave it for later.

## Check For

- Tests marked `.skip`, `test.skip`, `xit`, `xdescribe`, `@pytest.mark.skip`, etc.
- Tests that were skipped due to earlier failures (serial test suites)
- Flaky tests that "usually pass" — these are bugs

## Not Acceptable

- "It was already skipped before I started"
- "It's unrelated to my change"
- "It's flaky but usually passes"

If you didn't break it, you still found it. Fix it or flag it.
