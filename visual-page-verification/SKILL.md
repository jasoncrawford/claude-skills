# Visual Page Verification

## When to invoke

Use this skill on **any PR that ships a page or page-section component** — new pages, template changes, layout tweaks, CSS edits, or component refactors that affect rendered output.

This is not optional. The reason PR #54 (homepage template) shipped with a broken layout is that no one looked at the page in a browser before merging. This skill makes browser verification mechanical.

## Required workflow

### 1. Run the script

```bash
npm run visual-diff -- <path>
```

Examples:
```bash
npm run visual-diff -- /
npm run visual-diff -- /about/
npm run visual-diff -- /demo/homepage-demo
```

This produces up to 4 PNGs in `tmp/visual-diff/`:
- `<slug>-local-desktop.png`
- `<slug>-live-desktop.png`
- `<slug>-local-mobile.png`
- `<slug>-live-mobile.png`

(`/demo/` paths produce only local screenshots — no live counterpart.)

### 2. Read both PNGs

Use the `Read` tool to view each PNG. Claude can see images — this is the key step that catches layout regressions.

```
Read: tmp/visual-diff/<slug>-local-desktop.png
Read: tmp/visual-diff/<slug>-live-desktop.png
Read: tmp/visual-diff/<slug>-local-mobile.png
Read: tmp/visual-diff/<slug>-live-mobile.png
```

Check:
- Columns, spacing, and alignment match
- Typography (font, size, weight, line-height) matches
- Images are present and correctly sized
- Colors and borders look the same
- Mobile layout stacks correctly at 375 px

### 3. Iterate until they match

If local doesn't match live, fix the code and re-run. Repeat until the screenshots match (or the difference is intentional and explained in the PR).

### 4. Document your visual comparison in the PR body

In the PR description, write what you saw when you compared the screenshots. For example:

> **Visual diff — `/about/` desktop:** Layout matches. Header, nav, body text, and footer align with the live site. No regressions on mobile.

If there are differences, describe them and explain whether they're intentional.

Note: local file paths (`tmp/visual-diff/...`) can't be embedded in GitHub PRs. Describe what you observed in words so the reviewer has confidence the comparison was actually done.

## Red flags

**"Matches the original"** — if you wrote this in a PR description without having used the `Read` tool to look at both screenshots, stop. You haven't actually verified anything.

**No screenshots in PR** — if the PR touches a page or component and has no screenshots, it should not be merged.

**Skipping because "it's a small change"** — small CSS changes break layouts. Small changes get screenshots too.

## See also

- `docs/visual-diff.md` — what the script does, Chromium install, what to look for
- GitHub issue #57 — why this exists
