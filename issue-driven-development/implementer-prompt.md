# Implementer Subagent Prompt Template

Use this template when dispatching an implementer subagent.

```
Task tool (general-purpose):
  description: "Implement fix for issue #N: [issue title]"
  prompt: |
    You are implementing a fix based on an approved plan. Follow the plan
    precisely. Do NOT commit your changes.

    ## The Issue

    [FULL TEXT of the issue]

    ## The Approved Plan

    [FULL TEXT of the approved plan from the planner]

    ## Project Context

    [Brief project description, how to run tests]

    ## Your Job

    1. Read the files mentioned in the plan to confirm current state
    2. Write the tests specified in the plan FIRST (they should fail)
    3. Implement the changes exactly as specified in the plan
    4. Run tests: `npm test` — new tests should now pass
    5. If tests fail, investigate and fix (staying within the plan's scope)
    6. Do NOT commit — leave changes uncommitted

    ## Tests Are Not Optional

    The plan includes a Tests section. You MUST write those tests.
    - Write tests BEFORE or ALONGSIDE the implementation
    - If the plan says "no new tests needed", verify that claim by
      identifying the specific existing tests that cover the fix
    - "All existing tests pass" only proves no regressions — it does
      NOT prove the fix works
    - If you skip writing tests, the review WILL reject your work

    ## Self-Review Before Reporting

    Before reporting back, review your own work:
    - Did you implement everything in the plan?
    - Did you write all tests specified in the plan?
    - Did you change anything NOT in the plan?
    - Do all tests pass (old AND new)?
    - Did you introduce any regressions?

    ## Report Format

    - What you changed (files and specific changes)
    - What tests you wrote and what they verify
    - Test results (paste output)
    - Any deviations from the plan and why
    - Any concerns
```
