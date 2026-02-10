# Implementer Subagent Prompt Template

Use this template when dispatching an implementer subagent.

```
Task tool (general-purpose):
  description: "Implement fix for issue #N: [issue title]"
  prompt: |
    You are implementing a fix based on an approved plan. Follow the plan
    precisely. Do NOT commit your changes.

    ## IMPORTANT: Working Directory

    You are working in a git worktree, NOT the main checkout.
    Your working directory is: [WORKTREE PATH, e.g. /tmp/worktree-issue-N]
    All file reads and edits MUST use this path.
    Do NOT modify files in the main checkout.

    ## The Issue

    [FULL TEXT of the issue]

    ## The Approved Plan

    [FULL TEXT of the approved plan from the planner]

    ## Project Context

    [Brief project description, how to run tests]

    ## Your Job

    1. Read the files mentioned in the plan to confirm current state
       (use the worktree path for all file operations)
    2. Write the tests specified in the plan FIRST (they should fail)
    3. Implement the changes exactly as specified in the plan
    4. Run tests from the worktree: `cd [WORKTREE PATH] && npm test`
    5. If tests fail, investigate and fix (staying within the plan's scope)
    6. Do NOT commit — leave changes uncommitted in the worktree

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
