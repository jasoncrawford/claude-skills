---
name: test-discipline
description: Use when implementing any bug fix, feature, or code change — ensures new tests are written for new behavior, and no existing tests are skipped or failing
---

# Test Discipline

## Write Tests for New Behavior

For every bug fix or feature, write at least one test that:
- Would **FAIL** before your change
- Would **PASS** after your change

"Existing tests still pass" only proves no regressions — it does **not** verify the fix or feature works.

If you believe no new test is needed, you must identify the specific existing test(s) that cover the new behavior and explain why they're sufficient. Vague claims like "existing tests cover this" are not acceptable.

## No Skipped or Failing Tests

All tests must pass. Zero skipped. If a test is failing or skipped, investigate and fix it — don't work around it or leave it for later.

Check for:
- Tests marked `.skip`, `test.skip`, `xit`, `xdescribe`, `@pytest.mark.skip`, etc.
- Tests that were skipped due to earlier failures (serial test suites)
- Flaky tests that "usually pass" — these are bugs

## Not Acceptable

- Writing no new test for a bug fix ("the bug was obvious, no test needed")
- Writing no new test for a feature ("it's small")
- "It was already skipped before I started"
- "It's unrelated to my change"
- "It's flaky but usually passes"

If you didn't break it, you still found it. Fix it or flag it.
