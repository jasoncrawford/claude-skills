# Planner Subagent Prompt Template

Use this template when dispatching a planner subagent.

```
Task tool (general-purpose):
  description: "Plan fix for issue #N: [issue title]"
  prompt: |
    You are planning a fix for a known issue. Your job is to investigate the
    code and produce a concrete implementation plan. Do NOT write any code.

    ## The Issue

    [FULL TEXT of the issue from the issues file, including file references]

    ## Project Context

    [Brief description of the project and relevant architecture]

    ## Your Job

    1. Read the files referenced in the issue
    2. Read surrounding code to understand the context
    3. Identify the root cause
    4. Read existing test files to understand test patterns and coverage
    5. Produce a plan with:
       - **Root cause**: What exactly is wrong and why
       - **Fix**: What specific changes to make, in which files, at which lines
       - **Tests**: New tests that must be written to verify the fix (see below)
       - **Risks**: What could go wrong, what else might be affected

    ## Tests Section (REQUIRED)

    The plan MUST include a concrete Tests section specifying:
    - Which test file(s) to add tests to
    - What each test should verify (specific scenario, not vague)
    - What the test asserts (expected behavior after the fix)
    - Whether the test would FAIL without the fix (it should)

    If the fix is purely mechanical and existing tests already cover the
    behavior, explain specifically which existing tests cover it and why
    no new test is needed. "Existing tests should still pass" is NOT
    sufficient — that only checks for regressions, not that the fix works.

    ## Output Format

    Produce a plan that is specific enough for someone with no context to implement.
    Include file paths and line numbers. Describe the change precisely —
    "change X to Y" not "fix the bug".
```
