---
name: plan-milestone
description: Use when you have a big goal that requires multiple issues and PRs — helps clarify requirements, break down the work, then creates a GitHub milestone with properly ordered, dependency-linked issues.
---

# Plan Milestone

Turns a big goal into a GitHub milestone with a structured set of issues and explicit dependencies.

**Announce at start:** "I'm using the plan-milestone skill to help plan this work."

## When to use

- A feature or change is too big for a single issue/PR
- Work can be parallelized across issues
- There are dependencies between pieces of work
- You want to track progress toward a larger goal in GitHub

## The Process

### 1. Clarify the goal

Start with the description in $ARGUMENTS. Ask clarifying questions — **one at a time** — until you have enough to propose a breakdown:

- What does success look like? How will you know this milestone is done?
- Are there constraints (timeline, external dependencies, things explicitly out of scope)?
- Are there existing issues, PRs, or design docs relevant to this work?
- Is the technical approach already decided, or does it need design work first?

**If significant design work is still needed** (product spec, architecture decisions, data model choices), suggest running `superpowers:brainstorming` first. This skill assumes enough clarity to break down the work into issues.

### 2. Propose the work breakdown

Once you understand the goal, propose:

- **Milestone title** — short, describes the capability being built
- **Milestone description** — 1–2 sentences: goal + definition of done
- **Issue list** — for each issue:
  - Title (imperative verb, specific)
  - What it does and why it's needed (2–4 sentences)
  - Dependencies: which other issues in this set must be done first
  - Parallelism: which issues can be worked concurrently

Present a dependency graph so the sequencing is clear. Get user approval and iterate until the breakdown feels right.

**Breakdown principles:**
- Each issue should be independently mergeable (produces working, testable software on its own)
- Prefer more smaller issues over fewer large ones
- Put infrastructure/foundation issues first; higher-level features later
- Avoid issues that are just "glue" with no clear deliverable

### 3. Design doc (if needed)

If there's architecture, data model decisions, or API contracts that don't fit neatly in issue bodies, offer to write a design doc first:

> "Before I file the issues, is there anything worth capturing in a short design doc (e.g. data model, API shape, overall approach)? I can write it to `docs/` and reference it from the issues."

If yes, write to `docs/YYYY-MM-DD-<feature>.md`, commit it, then proceed.

### 4. Create the GitHub milestone

```bash
REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
MILESTONE=$(gh api repos/$REPO/milestones \
  --method POST \
  --field title="<milestone title>" \
  --field description="<milestone description>" \
  --jq '.number')
echo "Milestone #$MILESTONE created"
```

### 5. File the issues

File issues **in dependency order** — blockers first, so their issue numbers are known before they're referenced.

For each issue:

```bash
gh issue create \
  --title "<title>" \
  --body "<description>

Depends on #N" \
  --milestone "<milestone title>"
```

Do **not** add the `brunel:ready` label — leave that to the user.

**Dependency format recognized by the foreman:**

```
Depends on #N
Blocked by #N, #M
**Depends on:** #N
```

The foreman parses `depends on` and `blocked by` (case-insensitive, with optional bold/italic) followed by one or more `#N` issue numbers. An issue with open blockers will stay in `blocked` status until they're closed.

For issues with no dependencies, omit the depends-on line entirely.

### 6. Summary

After filing all issues, report:

- Link to the milestone
- Numbered list of issues with titles
- Which issues are immediately workable (no blockers)
- Dependency relationships shown clearly

Example:

```
Milestone: github.com/{repo}/milestone/5

Issues filed:
#42 — Set up database schema (no blockers — ready now)
#43 — Build API endpoints (depends on #42)
#44 — Add frontend UI (depends on #43)
#45 — Write integration tests (depends on #42, can run parallel with #43–44)

Ready to start: #42
```
