---
name: building-animated-svg-ecards
description: Use when building an animated, illustrated single-page site — e-card, invitation, announcement — with hand-coded SVG scenes, scroll-triggered animations, or scroll-snap paging, especially when it must run smoothly on phones. Also use when such a page has jerky/janky animations on mobile, a "next" button that snaps back to the same page, or reveal animations that appear already-finished after a loading screen.
---

# Building Animated SVG E-Cards

## Overview

A greeting-card SPA is one self-contained `index.html` (inline CSS + SVG + vanilla JS, no build) with full-viewport scroll-snap pages, one illustrated scene per beat of the message. The binding constraint is the phone GPU: desktop hides every mistake, mobile exposes all of them, and **first-ever render is the worst case** (cold font/gradient/geometry caches).

Working code for every recipe below: see recipes.md (taken from a shipped card).

## The traps (each one shipped as a bug before being learned)

| Trap | Mechanism | Fix |
|------|-----------|-----|
| Jerky scene animations on phone only | `feGaussianBlur`/any SVG filter on an animating subtree re-rasterizes the whole filtered region every frame; a filter wrapping a parent group of many animating children is the worst case | Filters only on fully static elements. Softness on animated art comes from `radialGradient` fills (0 → soft falloff → transparent). Glow = stacked gradient circles |
| Chevron/"next" advances then snaps back | `scroll-snap-type: y mandatory` fights `scrollIntoView({behavior:'smooth'})` on mobile; snap grabs the scroll mid-flight | Disable snap during programmatic scroll, restore when position settles; timeout fallback must *complete* the scroll (jump to target), not just restore snap — restoring mid-flight snaps backward |
| First page's reveal "already finished" after loader | Removing the warm-up's force-visible class lets elements *transition* back to hidden (reverse glide, seconds long); reveal then starts from ~90% visible | One-frame `body.reset` class with `transition:none !important` while un-warming, flush styles, then remove. And attach the IntersectionObserver only when the loader fade *starts* |
| First run janky, revisits smooth | Cold caches: fonts, glyph atlases, gradient textures, SVG geometry rasterize on first view | Warm-up behind a loading veil: force all reveals to final state, scroll through every page (rAF-paced), reset, then reveal. Race `document.fonts.ready` with a timeout; hard-cap the whole thing (~6s) because rAF throttles in background tabs |
| Many small pops stutter | Dozens of independent SVG transforms per frame | Batch: one animated `<g>` per cluster (~6 leaves), staggered via `--d` custom property |
| Text reveal stutters | Transitioning CSS `filter: blur()` on text | Fade + translate only |

## Composition rules (recurring user feedback)

- Every bloom's stem must reach it; every leaf's base sits on a stem. Floating elements read as broken, not artistic.
- Density is what separates "real illustration" from "barely a decoration": generate it (leaves along an ellipse with jitter + flowers at ring fractions) instead of hand-placing. Layered translucent circles read as watercolor; stylized round fauna works, realism doesn't.
- One scene per message beat, thematically tied (celebration→wreath, warmth→sun, companionship→two birds); ambient color via static blurred CSS divs behind each section.

## Process that worked

Style concepts as clickable sample cards first → pick direction → full prototype → density pass → mobile perf pass. Verify animation states by sampling `getComputedStyle` opacity over time when screenshots are too slow to catch transitions. Personal pages: `noindex` meta + unguessable deploy name.
