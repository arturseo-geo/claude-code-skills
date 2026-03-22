# Technical SEO Audit

## Sitemap-First Crawl

### Strategy

Always start from the sitemap, not from the homepage. This ensures you audit every
indexed page, including deep pages that might be orphaned from internal navigation.

### Implementation

```javascript
const cheerio = require('cheerio');

async function crawlSitemap(siteUrl) {
  // Step 1: Fetch sitemap index or single sitemap
  const sitemapUrl = `${siteUrl}/sitemap.xml`;
  const res = await fetch(sitemapUrl);
  const xml = await res.text();
  const $ = cheerio.load(xml, { xmlMode: true });

  // Check if this is a sitemap index (contains <sitemap> elements)
  const subSitemaps = [];
  $('sitemap > loc').each((i, el) => {
    subSitemaps.push($(el).text().trim());
  });

  if (subSitemaps.length > 0) {
    // Sitemap index — fetch each sub-sitemap
    const allUrls = [];
    for (const subUrl of subSitemaps) {
      const subRes = await fetch(subUrl);
      const subXml = await subRes.text();
      const sub$ = cheerio.load(subXml, { xmlMode: true });
      sub$('url > loc').each((i, el) => {
        allUrls.push(sub$(el).text().trim());
      });
    }
    return allUrls;
  }

  // Single sitemap — extract URLs directly
  const urls = [];
  $('url > loc').each((i, el) => {
    urls.push($(el).text().trim());
  });
  return urls;
}
```

---

## Per-Page Checks

### Title Tag

```javascript
function checkTitle($) {
  const title = $('title').text().trim();
  const issues = [];
  if (!title) issues.push({ type: 'error', msg: 'Missing title tag' });
  else if (title.length < 30) issues.push({ type: 'warning', msg: `Title too short (${title.length} chars)` });
  else if (title.length > 60) issues.push({ type: 'warning', msg: `Title too long (${title.length} chars)` });
  return { value: title, length: title.length, issues };
}
```

| Severity | Condition | Recommendation |
|---|---|---|
| Error | Missing title | Add a unique, keyword-rich title |
| Warning | < 30 characters | Too short — add descriptive words |
| Warning | > 60 characters | May be truncated in SERPs |

### Meta Description

```javascript
function checkMetaDescription($) {
  const desc = $('meta[name="description"]').attr('content')?.trim() || '';
  const issues = [];
  if (!desc) issues.push({ type: 'error', msg: 'Missing meta description' });
  else if (desc.length < 120) issues.push({ type: 'warning', msg: `Meta description too short (${desc.length} chars)` });
  else if (desc.length > 160) issues.push({ type: 'warning', msg: `Meta description too long (${desc.length} chars)` });
  return { value: desc, length: desc.length, issues };
}
```

### H1 Tag

- **Missing H1**: Error — every content page needs exactly one H1
- **Multiple H1s**: Warning — use only one H1 per page, use H2+ for subsections
- **Empty H1**: Error — H1 exists but contains no text

### Canonical Tag

```javascript
function checkCanonical($, pageUrl) {
  const canonical = $('link[rel="canonical"]').attr('href')?.trim() || '';
  const issues = [];
  if (!canonical) {
    issues.push({ type: 'warning', msg: 'Missing canonical tag' });
  } else if (canonical !== pageUrl && !canonical.endsWith(new URL(pageUrl).pathname)) {
    issues.push({ type: 'info', msg: `Canonical points elsewhere: ${canonical}` });
  }
  return { value: canonical, issues };
}
```

### Viewport Meta

Check for `<meta name="viewport" content="width=device-width, initial-scale=1">`.
Missing viewport = not mobile-friendly.

### Open Graph Tags

Required for social sharing:
- `og:title` — page title for social
- `og:description` — description for social
- `og:image` — preview image (minimum 1200x630px recommended)
- `og:url` — canonical URL
- `og:type` — usually "website" or "article"

### Image Alt Attributes

```javascript
function checkImages($) {
  const issues = [];
  const images = [];
  $('img').each((i, el) => {
    const src = $(el).attr('src') || '';
    const alt = $(el).attr('alt');
    if (alt === undefined || alt === null) {
      issues.push({ type: 'error', msg: `Image missing alt: ${src}` });
    } else if (alt.trim() === '') {
      // Empty alt is OK for decorative images but flag for review
      issues.push({ type: 'info', msg: `Empty alt (decorative?): ${src}` });
    }
    images.push({ src, alt: alt || '', hasAlt: !!alt });
  });
  return { total: images.length, missingAlt: issues.filter(i => i.type === 'error').length, issues };
}
```

### Word Count

```javascript
function checkWordCount($) {
  // Remove script and style content
  $('script, style, nav, header, footer').remove();
  const text = $('body').text();
  const words = text.split(/\s+/).filter(w => w.length > 0).length;

  const issues = [];
  if (words < 300) issues.push({ type: 'warning', msg: `Thin content (${words} words)` });
  return { wordCount: words, issues };
}
```

### HTML Size

Flag pages over 1 MB — may cause slow rendering and crawl budget waste.

---

## Site-Wide Checks

### Duplicate Titles

```javascript
function findDuplicateTitles(pages) {
  const titleMap = {};
  pages.forEach(p => {
    const title = p.title?.toLowerCase().trim();
    if (title) {
      if (!titleMap[title]) titleMap[title] = [];
      titleMap[title].push(p.url);
    }
  });
  return Object.entries(titleMap)
    .filter(([_, urls]) => urls.length > 1)
    .map(([title, urls]) => ({ title, urls, count: urls.length }));
}
```

### Duplicate Meta Descriptions

Same pattern as duplicate titles. Even more common — many CMS themes use the
same meta description template across pages.

---

## Link Graph

### Building the Graph

During the crawl, extract all internal links per page:

```javascript
function extractInternalLinks($, pageUrl, siteDomain) {
  const links = new Set();
  $('a[href]').each((i, el) => {
    let href = $(el).attr('href');
    if (!href) return;

    // Resolve relative URLs
    try {
      const resolved = new URL(href, pageUrl).href;
      if (resolved.includes(siteDomain)) {
        links.add(resolved.split('#')[0].split('?')[0]); // strip fragment + query
      }
    } catch (e) { /* invalid URL */ }
  });
  return [...links];
}
```

### Orphan Pages

Pages with zero inbound internal links. These are invisible to users navigating
the site and may be under-crawled by search engines.

```javascript
function findOrphanPages(pages, linkGraph) {
  const linkedTo = new Set();
  Object.values(linkGraph).forEach(links => links.forEach(l => linkedTo.add(l)));

  return pages.filter(p => !linkedTo.has(p.url));
}
```

### Deep Pages

Pages requiring >3 clicks from the homepage. Calculate via BFS from homepage:

```javascript
function calculateDepth(homepage, linkGraph) {
  const depth = { [homepage]: 0 };
  const queue = [homepage];

  while (queue.length > 0) {
    const current = queue.shift();
    const links = linkGraph[current] || [];
    for (const link of links) {
      if (!(link in depth)) {
        depth[link] = depth[current] + 1;
        queue.push(link);
      }
    }
  }
  return depth;
}
```

Flag pages with depth > 3 — they are hard for search engines to discover.

---

## Schema Validation

### JSON-LD Required Fields

| Schema Type | Required Fields |
|---|---|
| Article | `headline`, `datePublished`, `author` (with `name`), `image` |
| FAQ | `mainEntity[]` each with `@type: Question`, `name`, `acceptedAnswer.text` |
| HowTo | `name`, `step[]` each with `text` (and optionally `name`, `image`) |
| Product | `name`, `offers` with `price`, `priceCurrency`, `availability` |
| Organization | `name`, `url`, `logo` |
| BreadcrumbList | `itemListElement[]` each with `position`, `name`, `item` (URL) |
| LocalBusiness | `name`, `address`, `telephone` |
| Event | `name`, `startDate`, `location` |

### Validation Implementation

```javascript
function validateSchema(schemaJson) {
  const type = schemaJson['@type'];
  const issues = [];

  const requiredFields = {
    'Article': ['headline', 'datePublished', 'author', 'image'],
    'FAQPage': ['mainEntity'],
    'HowTo': ['name', 'step'],
    'Product': ['name', 'offers'],
    'Organization': ['name', 'url', 'logo'],
    'BreadcrumbList': ['itemListElement']
  };

  const required = requiredFields[type] || [];
  required.forEach(field => {
    if (!schemaJson[field]) {
      issues.push({ type: 'error', msg: `${type} missing required field: ${field}` });
    }
  });

  return { type, issues, valid: issues.length === 0 };
}
```

---

## Core Web Vitals via Google PSI API

### API Call

```javascript
async function getCoreWebVitals(url) {
  const apiUrl = `https://www.googleapis.com/pagespeedonline/v5/runPagespeed?url=${encodeURIComponent(url)}&strategy=mobile&category=performance`;
  const res = await fetch(apiUrl);
  const data = await res.json();

  const crux = data.loadingExperience?.metrics || {};
  const lighthouse = data.lighthouseResult?.audits || {};

  return {
    // CrUX field data (real users)
    lcp: crux['LARGEST_CONTENTFUL_PAINT_MS']?.percentile,
    fid: crux['FIRST_INPUT_DELAY_MS']?.percentile,
    cls: crux['CUMULATIVE_LAYOUT_SHIFT_SCORE']?.percentile,
    inp: crux['INTERACTION_TO_NEXT_PAINT']?.percentile,

    // Lighthouse lab data
    fcp: lighthouse['first-contentful-paint']?.numericValue,
    ttfb: lighthouse['server-response-time']?.numericValue,
    tbt: lighthouse['total-blocking-time']?.numericValue,
    performanceScore: data.lighthouseResult?.categories?.performance?.score * 100
  };
}
```

### Thresholds

| Metric | Good | Needs Improvement | Poor |
|---|---|---|---|
| LCP | < 2,500ms | 2,500–4,000ms | > 4,000ms |
| INP | < 200ms | 200–500ms | > 500ms |
| CLS | < 0.1 | 0.1–0.25 | > 0.25 |
| FCP | < 1,800ms | 1,800–3,000ms | > 3,000ms |
| TTFB | < 800ms | 800–1,800ms | > 1,800ms |
| TBT | < 200ms | 200–600ms | > 600ms |

### Rate Limits

PSI API without an API key: ~1 request per second, ~25,000/day.
With an API key: higher limits. For most audit purposes, keyless is sufficient.

---

## AI Audit Fix Suggestions

Send failing audit checks to Groq for AI-generated fixes:

```javascript
async function generateFixes(failingChecks) {
  const groq = new Groq({ apiKey: process.env.GROQ_API_KEY });

  const prompt = `You are an SEO expert. For each failing check below, provide a specific fix.

Failing checks:
${failingChecks.map(c => `- ${c.url}: ${c.msg}`).join('\n')}

For each, provide:
1. The specific fix (e.g., a rewritten title, meta description, alt text)
2. Keep titles under 60 chars, meta descriptions under 160 chars
3. Be specific — don't just say "add a title", write the actual title`;

  const completion = await groq.chat.completions.create({
    model: 'llama-3.1-8b-instant',
    messages: [{ role: 'user', content: prompt }],
    temperature: 0.3
  });

  return completion.choices[0].message.content;
}
```

This is useful for:
- Bulk-fixing missing meta descriptions across dozens of pages
- Generating alt text suggestions from image filenames and page context
- Rewriting titles that are too long/short
- Flagging thin content pages with expansion topics

---

## Cheerio Bug Warning

**CRITICAL:** cheerio throws `"Unexpected type of selector"` when you pass a plain
JavaScript object to `$()`. This happens when you accidentally do:

```javascript
// BAD — will throw
const obj = { title: 'something' };
$(obj).find('a');

// GOOD — use DOM elements or selectors
$('div.content').find('a');
$(domElement).find('a');
```

This is easy to trigger when iterating over mixed data structures. Always verify
what you are passing to `$()` is either a string selector or a DOM element, never
a plain object.

---

## Complete Audit Report Structure

```json
{
  "site": "https://example.com",
  "auditDate": "2026-03-21",
  "pagesAudited": 45,
  "summary": {
    "errors": 12,
    "warnings": 28,
    "passed": 185
  },
  "siteWide": {
    "duplicateTitles": [...],
    "duplicateDescriptions": [...],
    "orphanPages": [...],
    "deepPages": [...]
  },
  "pages": [
    {
      "url": "https://example.com/page",
      "title": { "value": "...", "length": 45, "issues": [] },
      "metaDescription": { "value": "...", "length": 140, "issues": [] },
      "h1": { "count": 1, "values": ["..."], "issues": [] },
      "canonical": { "value": "...", "issues": [] },
      "schema": [{ "type": "Article", "valid": true, "issues": [] }],
      "images": { "total": 8, "missingAlt": 2, "issues": [...] },
      "wordCount": 1200,
      "coreWebVitals": { "lcp": 2100, "inp": 150, "cls": 0.05 }
    }
  ],
  "aiFixes": [...]
}
```
