# Recipes (from a shipped card)

All verified on a real deployment viewed on a phone. Reference implementation:
`~/birthday-ecard/index.html`.

## 1. Soft watercolor blob — gradient, not filter (safe to animate)

```html
<radialGradient id="rg_f0aebf">
  <stop offset="0" stop-color="#f0aebf"/>
  <stop offset=".55" stop-color="#f0aebf" stop-opacity=".92"/>
  <stop offset=".8" stop-color="#f0aebf" stop-opacity=".5"/>
  <stop offset="1" stop-color="#f0aebf" stop-opacity="0"/>
</radialGradient>
<!-- a peony = 4-6 overlapping gradient circles at opacity .5-.7; a glow/sun = concentric ones -->
<circle cx="152" cy="150" r="44" fill="url(#rg_f0aebf)" opacity="0.6"/>
```

Define one gradient per color in a hidden `<svg width="0" height="0">` at the top of
`<body>` — `url(#id)` resolves document-wide. Real `feGaussianBlur` is allowed only on
elements that never animate (background washes). Big ambient washes are cheaper as
`position:absolute` divs with CSS `filter: blur(60px)` (static ⇒ rendered once).

## 2. Reveal-on-scroll with replay

```css
.bloom { opacity:0; transform:scale(.55); transform-origin:center; transform-box:fill-box;
         transition:opacity 1.6s ease, transform 1.9s cubic-bezier(.2,.7,.3,1); }
.visible .bloom { opacity:1; transform:scale(1); }
/* stems draw in: */
.draw { stroke-dasharray:100; stroke-dashoffset:100; transition:stroke-dashoffset 2.4s ease .1s; }
.visible .draw { stroke-dashoffset:0; }   /* requires pathLength="100" on the path */
```

```js
const revealTimers = new WeakMap();
const io = new IntersectionObserver((entries) => {
  for (const e of entries) {
    if (e.isIntersecting) {          // defer so the reveal doesn't fight the snap scroll
      revealTimers.set(e.target, setTimeout(() => e.target.classList.add('visible'), 140));
    } else {
      clearTimeout(revealTimers.get(e.target));
      e.target.classList.remove('visible');   // replay on next visit
    }
  }
}, { threshold: 0.3 });
// NOTE: observe sections only when the loading veil starts fading (see recipe 4)
```

Stagger many elements with a per-element custom property:
`.wr { transition-delay: var(--d, 0s); }` and `style="--d:1.2s"` (or set in JS).
Batch clusters (~6 leaves) into one `.wr` group — dozens of independent per-frame
SVG transforms stutter on phones.

## 3. Scroll-snap paging with a working "next" button

```css
html { scroll-snap-type: y mandatory; }
section { min-height:100svh; scroll-snap-align:start; scroll-snap-stop:always; }
```

```js
// Mandatory snap yanks a smooth programmatic scroll back to the current page on
// mobile. Disable snap for the flight; the timeout fallback must COMPLETE the
// scroll (not merely restore snap — restoring mid-flight snaps backward).
const htmlEl = document.documentElement;
function smoothScrollTo(target) {
  htmlEl.style.scrollSnapType = 'none';
  const top = target.offsetTop;
  window.scrollTo({ top, behavior:'smooth' });
  let settled = 0, guard = 0, done = false;
  const restore = () => {
    if (done) return;
    done = true;
    if (Math.abs(window.scrollY - top) > 2) window.scrollTo(0, top); // finish stalled scroll
    htmlEl.style.scrollSnapType = '';
  };
  const check = () => {
    if (done) return;
    settled = Math.abs(window.scrollY - top) < 2 ? settled + 1 : 0;
    if (settled >= 3 || ++guard > 240) { restore(); return; }
    requestAnimationFrame(check);
  };
  requestAnimationFrame(check);
  setTimeout(restore, 2500);  // rAF throttles in hidden tabs — always restore
}
```

Chevron button: absolutely positioned bottom-center inside each section, gentle
CSS bounce, fades in ~1.8s after the page's reveal starts. Watch specificity:
a rule like `section > :not(.wash) { position:relative }` silently overrides the
button's `position:absolute` (classes inside `:not()` count toward specificity).

## 4. Warm-up preload (kills first-run jank) + correct reset

First view of each page pays one-time costs (fonts, glyph atlases, gradient
textures, SVG geometry). Pre-render everything behind a loading veil, then reset.

```css
/* force every reveal to final state, instantly */
body.warm .bloom, body.warm .rv, body.warm .wr, body.warm .fd, body.warm .next {
  opacity:1 !important; transform:none !important; transition:none !important; }
body.warm .draw { stroke-dashoffset:0 !important; transition:none !important; }
/* THE TRAP: removing .warm alone lets elements TRANSITION back to hidden over
   ~2s, so the first reveal starts from ~fully-visible. Suppress for one frame: */
body.reset .bloom, body.reset .rv, body.reset .wr, body.reset .fd, body.reset .next,
body.reset .draw { transition:none !important; }
```

```js
async function warmUp() {
  const loader = document.getElementById('loader');   // fixed overlay, page bg color
  const raf = () => new Promise(r => requestAnimationFrame(r));
  const started = performance.now();
  let finished = false;
  const finish = () => {
    if (finished) return;
    finished = true;
    window.scrollTo(0, 0);
    document.body.classList.add('reset');
    document.body.classList.remove('warm');           // snaps (not glides) to hidden
    void document.body.offsetHeight;                  // flush while transitions are off
    const unreset = () => document.body.classList.remove('reset');
    requestAnimationFrame(() => requestAnimationFrame(unreset));
    setTimeout(unreset, 600);                         // rAF throttles when tab hidden
    htmlEl.style.scrollSnapType = '';
    const delay = Math.max(0, 700 - (performance.now() - started)); // min veil time
    setTimeout(() => {
      loader.classList.add('done');                   // starts opacity fade (.7s)
      // reveals start only as the veil lifts — never behind it:
      document.querySelectorAll('section').forEach(s => io.observe(s));
      setTimeout(() => loader.remove(), 800);
    }, delay);
  };
  setTimeout(finish, 6000);  // hard cap: never strand the loader
  document.body.classList.add('warm');
  htmlEl.style.scrollSnapType = 'none';
  await Promise.race([document.fonts.ready, new Promise(r => setTimeout(r, 1800))]);
  for (const s of document.querySelectorAll('section')) {
    if (finished) return;
    window.scrollTo(0, s.offsetTop);                  // each page must actually paint
    await raf(); await raf(); await raf();
  }
  finish();
}
```

## 5. Generative wreath/garland (density without hand-placing)

Place leaf ellipses along an ellipse at `leafCount` fractions: position
`(cx + cos θ·(rx ± jitter), cy + sin θ·(ry ± jitter))`, rotation = tangent angle
`atan2(ry·cosθ, −rx·sinθ)` ± ~22° jitter, one leaf each side of the ring line;
flowers/daisies/berries/baby's-breath at chosen fractions (`{at:.25, kind:'flower',
s:19, palette:'deep'}`). Stagger `--d` proportional to the fraction so the ring
blooms around clockwise. ~34 ring positions for a 400px wreath reads lush;
random jitter makes each page load a unique arrangement (charming for a card;
seed it if identical loads matter). Full implementation: `buildWreath()` in the
reference file.

## 6. Verifying animations when screenshots are too slow

Screenshot round-trips (1–2s) miss sub-second states. Sample instead:

```js
// did the reveal actually start from zero?
const el = document.querySelector('.bloom');
const t = [];
for (const ms of [0, 300, 700, 1200]) {
  await new Promise(r => setTimeout(r, ms));
  t.push(getComputedStyle(el).opacity);
}
// expect a climb like ["0", "0.035", "0.79", "1"] — a start near 1 means the
// reset failed and the animation is playing from an already-visible state
```
