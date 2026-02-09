# Reviewer Subagent Prompt Template

Use this template when dispatching a reviewer subagent.

```
Task tool (general-purpose):
  description: "Review fix for issue #N: [issue title]"
  prompt: |
    You are reviewing an implementation. You have NO prior context about this
    change — do not trust summaries, read the actual code.

    ## The Original Issue

    [FULL TEXT of the issue]

    ## The Approved Plan

    [FULL TEXT of the plan that was approved]

    ## What the Implementer Claims

    [FULL TEXT of the implementer's report]

    ## CRITICAL: Verify Independently

    The implementer's report may be incomplete or optimistic. You MUST:

    1. Run `git diff` to see ALL actual changes
    2. Read the changed code in context (not just the diff)
    3. Compare changes against the plan requirement by requirement
    4. Run `npm test` independently
    5. Check for:
       - Missing requirements from the plan
       - Changes NOT in the plan (scope creep)
       - Regressions or broken existing behavior
       - Code quality issues (but don't nitpick style)
       - Security concerns
    6. **Check test coverage** (see below)

    ## Test Coverage Check (REQUIRED)

    The plan specifies tests that must be written. You MUST verify:
    - Were all planned tests actually written?
    - Do the tests verify the FIX (not just absence of regressions)?
    - Would the tests FAIL if the fix were reverted?
    - Are the tests meaningful (not just smoke tests)?

    If the plan specified tests and they were not written, REJECT.
    "All existing tests pass" is not a substitute for new tests that
    verify the specific behavior being fixed.

    ## Report Format

    - **Tests**: PASS/FAIL (paste output)
    - **Test coverage**: Were planned tests written? Do they verify the fix?
    - **Plan compliance**: Does implementation match the plan?
      - Missing: [list anything from plan not implemented]
      - Extra: [list anything implemented but not in plan]
    - **Code quality**: Any significant issues?
    - **Verdict**: APPROVE or REJECT with specific reasons

    If rejecting, be specific about what needs to change.
```
