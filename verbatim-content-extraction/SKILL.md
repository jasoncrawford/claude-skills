---
name: verbatim-content-extraction
description: Use when migrating or extracting any content from a live website (WordPress, CMS, etc.) — markdown, structured data, or SVGs/icons/logos — that must be preserved verbatim. Prevents silent content corruption from LLM-mediated fetching or from the LLM "copying" inline SVG path data into string literals (which silently truncates).
---

# Verbatim Content Extraction

## The Core Rule

**Never use WebFetch (or any LLM-mediated tool) to extract content you intend to preserve verbatim.**

WebFetch returns an LLM-interpreted summary of the page, not the raw HTML. Using it for content migration will silently paraphrase, summarize, or hallucinate content — producing text that looks plausible but is not what the site actually says. This is especially dangerous because the corruption is invisible: the result looks correct at a glance.

## The Correct Pipeline

```
live site URL
     ↓
fetch() / curl  ←── no LLM here
     ↓
raw HTML
     ↓
DOM parser (cheerio, node-html-parser)
     ↓
turndown / node-html-markdown  ←── no LLM here
     ↓
verbatim markdown
```

The LLM writes the script. The script runs without the LLM. The LLM never touches the content.

## Recommended Tools

| Task | Tool |
|------|------|
| Fetch raw HTML | Node.js `fetch()`, `curl`, `axios` |
| Parse HTML | `cheerio`, `node-html-parser` |
| Convert to markdown | `turndown`, `node-html-markdown` |
| Extract structured data | DOM selectors on the parsed HTML |
| Download images | `fetch()` + `fs.writeFile()` |

## Capture-First Pattern (preferred for multi-page migrations)

For migrations involving many pages, separate the crawl from the conversion:

1. **Phase 1 — Capture:** Fetch every URL and save raw HTML to `capture/html/<slug>.html`. Record a `capture/manifest.json` with URL, HTTP status, timestamp, and SHA-256. `capture/` becomes the immutable source of truth.
2. **Phase 2 — Convert:** Read from `capture/html/`, run through Turndown, write to `src/content/`. Never re-fetch the live site in this phase.

**Why this matters:**
- You can re-run conversion without hitting the network
- `capture/` is auditable — reviewers can diff captured HTML against the live site
- If the live site changes mid-migration, a re-capture PR makes it explicit
- Conversion failures don't lose network state

The in-memory pattern below is fine for small one-off extractions.

## Typical Script Pattern (single-page / small-scale)

```js
import { JSDOM } from 'jsdom';
import TurndownService from 'turndown';
import fs from 'fs/promises';

const td = new TurndownService();

async function extractPost(url, slug) {
  const res = await fetch(url);
  const html = await res.text();
  const dom = new JSDOM(html);
  
  // Target the actual content container, not the whole page
  const content = dom.window.document.querySelector('.entry-content');
  const title = dom.window.document.querySelector('h1.entry-title')?.textContent.trim();
  const date = dom.window.document.querySelector('time')?.getAttribute('datetime');
  
  const markdown = td.turndown(content.innerHTML);
  
  await fs.writeFile(`src/content/blog/${slug}.md`, [
    '---',
    `title: "${title}"`,
    `date: ${date.slice(0, 10)}`,
    '---',
    '',
    markdown,
  ].join('\n'));
}
```

## SVGs Are Content Too

Inline SVGs — especially the long `<path d="M10.4013 27.311H41.78..."/>` numeric coordinate strings inside logos, icons, and decorative graphics — are exactly the kind of content the LLM corrupts when used as a copying channel. **The corruption looks like the SVG itself: just shorter.**

The failure mode isn't WebFetch this time; it's the LLM "remembering" or "copying" inline SVG into a string literal. Even when looking right at the source, the model truncates after a few hundred characters of path data. The result still parses and renders — it just renders a fragment of the original (e.g. the logo's outer shape but not the wordmark, or the star pattern's first few points but not the full grid). The diff against the live site looks like a layout bug.

**Extract SVGs the same way you extract markdown:** `curl` + DOM parse, save the raw `<svg>...</svg>` as a static file (e.g. `*.svg.html`), and inject via a raw-text loader (`import svg from './foo.svg.html?raw'` in Vite/Astro, `fs.readFileSync` in Node, etc.). Never paste SVG path data into a string literal.

```js
// Same pipeline as content extraction — the LLM never touches the path data.
const html = await fetch(url).then(r => r.text());
const dom = new JSDOM(html);
const svg = dom.window.document.querySelector('svg.target-logo').outerHTML;
await fs.writeFile('src/components/svg/logo.svg.html', svg);
```

The same principle applies to anything else opaque-to-humans and longer than a few hundred characters: minified scripts, base64 data URIs, font subset blobs. If you can't visually verify it's correct, treat it as content and pipe it through a script.

## WordPress-Specific Notes

- **Page builder content**: WordPress page builders (Elementor, Divi, etc.) store content in a custom format in the database. The XML export body content is often not useful. **Scrape the rendered HTML** instead.
- **Sitemap discovery**: Start from `/sitemap_index.xml` → subsidiary sitemaps (`post-sitemap.xml`, `page-sitemap.xml`, etc.) to get all URLs
- **Profile images**: Not at a predictable path. Scrape each profile page and extract the `wp-content/uploads` URL from the `<img>` tag

## What to Do With the Migration Plan

After running the script:
1. Spot-check 3–5 extracted files against the live site (title, first paragraph, last paragraph)
2. Check that dates, authors, and other metadata are accurate
3. Run `npm run build` and verify no schema errors

## Anti-Pattern: Using WebFetch for Content

```
❌ webfetch("https://example.com/post") → "Extract this as markdown"
```

This looks convenient but produces LLM-paraphrased output. Even with explicit instructions like "extract verbatim," LLMs will silently alter phrasing, omit sections, or hallucinate details.

The only safe role for LLM tools in content extraction is:
- Writing the extraction script
- Reviewing the script for correctness before running it
- Post-extraction spot checks comparing extracted markdown to the live page
