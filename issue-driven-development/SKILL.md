---
name: issue-driven-development
description: Use when working through issues from an issues file sequentially - picks highest-priority unresolved issue, plans, implements, reviews, and commits in isolated subagent contexts
---

# Issue-Driven Development

Sequentially resolve issues from a tracker file. Each phase runs in a fresh subagent to prevent context bias.

## When to Use

- You have an issues file (ISSUES.md, BUGS.md, TODO.md) with multiple unresolved issues
- You want to work through them one at a time, highest priority first
- You want planning, implementation, and review isolated from each other

## The Process

```dot
digraph process {
    rankdir=TB;

    "1. Select issue" [shape=box];
    "Read issues file, pick highest-priority unresolved" [shape=box];
    "Present issue to user for approval" [shape=box];
    "User approves?" [shape=diamond];

    "2. Plan (fresh subagent)" [shape=box style=filled fillcolor=lightyellow];
    "Subagent reads relevant code, produces plan" [shape=box];
    "Present plan to user for approval" [shape=box];
    "Plan approved?" [shape=diamond];

    "3. Implement (fresh subagent)" [shape=box style=filled fillcolor=lightblue];
    "Subagent receives plan text, implements and tests" [shape=box];

    "4. Review (fresh subagent)" [shape=box style=filled fillcolor=lightgreen];
    "Subagent reviews diff, runs tests, checks quality" [shape=box];
    "Review passes?" [shape=diamond];
    "Fix subagent addresses review issues" [shape=box];

    "5. Finalize" [shape=box];
    "Update issues file, commit" [shape=box];
    "More issues?" [shape=diamond];
    "Done" [shape=box style=filled fillcolor=lightgray];

    "1. Select issue" -> "Read issues file, pick highest-priority unresolved";
    "Read issues file, pick highest-priority unresolved" -> "Present issue to user for approval";
    "Present issue to user for approval" -> "User approves?";
    "User approves?" -> "2. Plan (fresh subagent)" [label="yes"];
    "User approves?" -> "Read issues file, pick highest-priority unresolved" [label="no, skip"];

    "2. Plan (fresh subagent)" -> "Subagent reads relevant code, produces plan";
    "Subagent reads relevant code, produces plan" -> "Present plan to user for approval";
    "Present plan to user for approval" -> "Plan approved?";
    "Plan approved?" -> "3. Implement (fresh subagent)" [label="yes"];
    "Plan approved?" -> "2. Plan (fresh subagent)" [label="no, revise"];

    "3. Implement (fresh subagent)" -> "Subagent receives plan text, implements and tests";
    "Subagent receives plan text, implements and tests" -> "4. Review (fresh subagent)";

    "4. Review (fresh subagent)" -> "Subagent reviews diff, runs tests, checks quality";
    "Subagent reviews diff, runs tests, checks quality" -> "Review passes?";
    "Review passes?" -> "5. Finalize" [label="yes"];
    "Review passes?" -> "Fix subagent addresses review issues" [label="no"];
    "Fix subagent addresses review issues" -> "4. Review (fresh subagent)" [label="re-review"];

    "5. Finalize" -> "Update issues file, commit";
    "Update issues file, commit" -> "More issues?";
    "More issues?" -> "1. Select issue" [label="yes"];
    "More issues?" -> "Done" [label="no"];
}
```

## Phase Details

### 1. Select Issue (controller does this directly)

Read the issues file. Pick the highest-priority unresolved issue (by section order: Critical > High > Medium > Lower). Present the issue to the user and get approval before proceeding.

### 2. Plan (fresh subagent)

Dispatch a `general-purpose` subagent with the issue text. Use `./planner-prompt.md` template. The planner:
- Reads relevant source files referenced in the issue
- Explores surrounding code for context
- Produces a concrete plan: what to change, where, and why
- Specifies what tests to write (REQUIRED — see below)
- Does NOT write any code

Present the plan to the user. If rejected, dispatch a new planner with feedback.

### 3. Implement (fresh subagent)

Dispatch a `general-purpose` subagent with the approved plan text (not a file path). Use `./implementer-prompt.md` template. The implementer:
- Receives the full plan text and issue context
- Writes the tests specified in the plan FIRST
- Implements exactly what the plan specifies
- Runs all tests (`npm test`) — new tests should now pass
- Does NOT commit (controller handles that after review)
- Reports what was changed, what tests were written, and test results

### 4. Review (fresh subagent)

Dispatch a `general-purpose` subagent to review the implementation. Use `./reviewer-prompt.md` template. The reviewer:
- Reads the git diff of uncommitted changes
- Compares implementation against the plan and original issue
- Verifies planned tests were written and actually test the fix
- Runs tests independently
- Checks for regressions, missed requirements, over-engineering
- Reports pass/fail with specific issues

If review fails, dispatch a fix subagent with the review feedback, then re-review.

### 5. Finalize (controller does this directly)

- Update the issues file (strikethrough title, add "FIXED" and commit hash)
- Commit all changes with a descriptive message
- Ask user if they want to continue to the next issue

## Key Rules

- **Never skip the user approval gate** after issue selection and after planning
- **Never pass file paths to subagents** — pass the full text content
- **Never let the implementer commit** — the controller commits after review passes
- **Each subagent gets a fresh context** — no resuming previous agents
- **Run tests in every phase** that touches code (implement and review)
- **Plan must specify tests, implementer must write them, reviewer must check them**
- **One issue at a time** — finish completely before moving to the next

## Red Flags

- Implementing without a plan (skip phase 2)
- Plan without a concrete Tests section
- Implementer skipping test-writing ("existing tests pass" is not enough)
- Reviewer not checking whether planned tests were written
- Reviewer trusting implementer's report instead of reading the diff
- Committing before review passes
- Moving to next issue with failing tests
- Controller doing implementation work instead of delegating to subagent
