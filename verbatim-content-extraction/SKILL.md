---
name: verbatim-content-extraction
description: Use when migrating or extracting content from a live website (WordPress, CMS, etc.) into markdown files or structured data. Prevents silent content corruption from LLM-mediated fetching.
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
