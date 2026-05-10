---
name: no-invented-css
description: Use when porting or reproducing CSS from a captured source (e.g. migrating a live site to a static site, or styling against a reference design) — especially when about to add a CSS rule that "makes things look right" or "fills in a gap" that isn't in the captured stylesheet. Prevents invented rules that mask layout bugs and silently corrupt the port.
---

# No Invented CSS

## The Iron Law

**When porting CSS from a captured source, every rule in your output must trace to a captured source rule.**

If a rule isn't in `capture/css/*.css` (or equivalent verbatim source), don't add it — even if "it makes things look right." Invented CSS masks bugs elsewhere instead of fixing them.

## Why It Matters

Invented rules paper over layout problems instead of fixing them. The bug you're trying to "fix" is almost always a *different* bug — a rule you forgot to port, a wrong selector, an extra wrapper from your scaffolding. Patching the symptom hides the real cause.

Two real examples from a WordPress → Astro port:

- Added `section.posts { padding: 60px 0 }` because spacing "looked off." Captured had **no** padding on `section.posts`. Real cause: each preceding section already has `padding: 60px 0` and provides the gap. The invented rule added 360 px of cumulative offset across the page.
- Added `border: 0` on `section.posts a.title` to "clean up underlines." Captured had only `transition: color 0.2s ease`. The invented rule wiped out the green rule between card title and excerpt — the underline was supposed to be there.

Each "fix" introduced a new bug that took a separate debugging session to find and remove.

## Red Flags — STOP

You're about to invent a rule. Stop.

- "Just adding a little padding here…"
- "This looks like it belongs."
- "It's not in the captured CSS, but the page looks wrong without it."
- "The brand is green, so headers should be green."
- "I'll add `overflow-y: auto` to be safe."
- "I'll improve on the original."

**All of these mean: don't add the rule. Find the real cause first.**

## Diagnostic: Measure, Don't Eyeball

When the visual diff is high, query the DOM with Playwright on **both** URLs in the same browser context. Eyeballing the diff PNG takes hours and guesses wrong; measurement finds the offending property in seconds.

```js
// On both pages, in the same script:
const sections = await page.evaluate(() =>
  [...document.querySelectorAll('main > section')].map(s =>
    ({ cls: s.className, h: Math.round(s.getBoundingClientRect().height) })));
```

1. Compare section heights — find the largest delta.
2. On the offending section, query `getComputedStyle()` on it and key children. Compare property by property.
3. First property that differs is the culprit. Trace it:
   - In your CSS but not in captured? → **invented; delete it.**
   - In captured but not in yours? → **missing; port it.**
   - In both at different specificity / media query? → **fix the cascade.**

This pattern took one port from 12% diff → 2.9% by surfacing six invented rules.

## Common Rationalizations

| Excuse | Reality |
|--------|---------|
| "Adding padding fixes the spacing" | Spacing is wrong because of a different missing rule. Padding masks it and accumulates offset elsewhere. |
| "The captured CSS is incomplete" | It's complete. Grep for the exact selector before assuming. |
| "I'll improve on the original" | Reproduction must match exactly first. Improvements come later, intentionally, with sign-off. |
| "Just one small invented rule" | They compound. Three small ones can cost hundreds of pixels. |
| "The diff is high, I have to do something" | Almost always *removing* an invented rule or *porting* a missed one — not writing new CSS. |

Before committing any CSS edit: **point at the line in the captured source.** No source line = invented. Delete it.
