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
    4. Produce a plan with:
       - **Root cause**: What exactly is wrong and why
       - **Fix**: What specific changes to make, in which files, at which lines
       - **Testing**: How to verify the fix works (existing tests, new tests needed)
       - **Risks**: What could go wrong, what else might be affected

    ## Output Format

    Produce a plan that is specific enough for someone with no context to implement.
    Include file paths and line numbers. Describe the change precisely —
    "change X to Y" not "fix the bug".
```
