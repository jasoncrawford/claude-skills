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
    2. Implement the changes exactly as specified in the plan
    3. Run tests: `npm test`
    4. If tests fail, investigate and fix (staying within the plan's scope)
    5. Do NOT commit — leave changes uncommitted

    ## Self-Review Before Reporting

    Before reporting back, review your own work:
    - Did you implement everything in the plan?
    - Did you change anything NOT in the plan?
    - Do all tests pass?
    - Did you introduce any regressions?

    ## Report Format

    - What you changed (files and specific changes)
    - Test results (paste output)
    - Any deviations from the plan and why
    - Any concerns
```
