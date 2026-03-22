# Backlink Intelligence — Free Stack

## DataForSEO Backlinks: The $100/Month Wall

DataForSEO backlinks API requires a minimum $100/month balance. This is **not bypassable**
via n8n integration, direct API calls, or any other method. Tested and confirmed —
the API returns "Payment required" below $100 balance.

The free $1 signup credit does NOT cover backlinks — only keyword data endpoints.

**Do not waste time trying to make DataForSEO backlinks work for free.**

---

## Free Alternative: Common Crawl CDX API

Common Crawl archives the web regularly. Their CDX (Capture Index) API lets you
query for pages that linked to any domain — effectively free backlink discovery.

### Endpoint

```
https://index.commoncrawl.org/CC-MAIN-{crawl-id}-index?url=*.example.com&output=json&fl=url,timestamp,status
```

### Querying Multiple Crawl Indexes

Query the 3 most recent crawl indexes for better coverage:

```javascript
async function discoverBacklinks(targetDomain) {
  // CRITICAL: Use native fetch(), NOT axios
  // Axios returns 404 inside Express servers for CC CDX — root cause unknown
  // This is a confirmed, reproducible bug. Always use fetch() for CC CDX.

  const crawls = ['2026-09', '2026-05', '2025-52']; // update periodically
  const allResults = [];

  for (const crawl of crawls) {
    // CRITICAL: Do NOT encodeURIComponent the domain
    const url = `https://index.commoncrawl.org/CC-MAIN-${crawl}-index?url=${targetDomain}&output=json&fl=url,timestamp,status&matchType=domain`;

    try {
      const res = await fetch(url);
      if (!res.ok) continue;
      const text = await res.text();
      const lines = text.trim().split('\n').filter(Boolean);
      const results = lines.map(line => JSON.parse(line));
      allResults.push(...results);
    } catch (e) {
      console.error(`CC crawl ${crawl} failed:`, e.message);
    }
  }

  return allResults;
}
```

### Key Warnings

1. **Use `fetch()`, not `axios`**: axios returns 404 for CC CDX URLs when called from
   inside an Express server. This is a confirmed bug with no known root cause. Native
   `fetch()` works reliably.

2. **Do NOT `encodeURIComponent()` the domain**: CC CDX expects the raw domain format.
   Encoding it will return zero results.

3. **Rate limits**: CC CDX has no official rate limit but be respectful — 1 request/second
   is a good practice.

### Recency Scoring

Backlinks found in multiple crawl indexes are likely still live:

```javascript
function recencyScore(link, crawlsFound) {
  if (crawlsFound >= 3) return 'high';   // found in all 3 crawls
  if (crawlsFound >= 2) return 'medium'; // found in 2 crawls
  return 'low';                           // found in 1 crawl only
}
```

---

## Live HTTP Verification

After discovering links via Common Crawl, verify they still exist:

```javascript
async function verifyBacklink(sourceUrl, targetDomain) {
  try {
    // Try HEAD first (faster)
    let res = await fetch(sourceUrl, { method: 'HEAD', redirect: 'follow' });

    // Some servers return 405/403 for HEAD — fallback to GET
    if (res.status === 405 || res.status === 403) {
      res = await fetch(sourceUrl, { method: 'GET', redirect: 'follow' });
    }

    if (!res.ok) return { live: false, status: res.status };

    // GET the page to check link presence
    const getRes = await fetch(sourceUrl);
    const html = await getRes.text();
    const $ = cheerio.load(html);

    let found = false;
    let dofollow = true;
    let anchorText = '';
    let context = '';

    $('a[href]').each((i, el) => {
      const href = $(el).attr('href') || '';
      if (href.includes(targetDomain)) {
        found = true;
        anchorText = $(el).text().trim();
        dofollow = !$(el).attr('rel')?.includes('nofollow');

        // Determine link context (what section of the page)
        const parent = $(el).closest('article, main, .content, .post-content, .entry-content');
        if (parent.length > 0) context = 'content';
        else if ($(el).closest('aside, .sidebar, .widget').length) context = 'sidebar';
        else if ($(el).closest('footer').length) context = 'footer';
        else if ($(el).closest('nav, .nav, .menu').length) context = 'nav';
        else context = 'unknown';
      }
    });

    return { live: true, found, dofollow, anchorText, context, status: res.status };
  } catch (e) {
    return { live: false, error: e.message };
  }
}
```

---

## Context-Based Quality Scoring

Each backlink scored 0–100 with letter grade:

### Scoring Breakdown

| Factor | Weight | Scoring |
|---|---|---|
| Link location | 40 | content: 40, sidebar: 15, footer: 5, nav: 5 |
| Anchor quality | 20 | keyword-rich: 20, branded: 15, partial-match: 12, generic: 5, naked URL: 3 |
| Follow status | 15 | dofollow: 15, nofollow: 5 |
| Link density | 15 | < 20 outbound links: 15, 20-50: 10, 50-100: 5, 100+: 2 |
| Page freshness | 10 | < 1 year: 10, 1-2 years: 7, 2-5 years: 4, 5+: 1 |

### Letter Grades

| Score | Grade | Assessment |
|---|---|---|
| 80–100 | A | Excellent — high-quality editorial link |
| 60–79 | B | Good — solid backlink worth keeping |
| 40–59 | C | Average — contributes some value |
| 20–39 | D | Low quality — monitor for spam signals |
| 0–19 | F | Poor — consider disavow if toxic |

---

## Toxic Link Detection

4-tier classification system:

```javascript
function toxicScore(url, anchorText) {
  let score = 0;
  const domain = new URL(url).hostname;

  // Spam TLDs
  const spamTLDs = ['.xyz', '.top', '.buzz', '.click', '.loan', '.work',
                     '.gq', '.cf', '.tk', '.ml', '.ga', '.pw', '.bid'];
  if (spamTLDs.some(tld => domain.endsWith(tld))) score += 30;

  // Numeric-heavy domains
  const numericRatio = (domain.match(/\d/g) || []).length / domain.length;
  if (numericRatio > 0.3) score += 20;

  // Excessive hyphens
  const hyphens = (domain.match(/-/g) || []).length;
  if (hyphens >= 3) score += 15;

  // Spam keywords in URL path
  const spamKeywords = ['casino', 'pharmacy', 'payday', 'viagra', 'poker',
                         'gambling', 'loan', 'forex', 'crypto-trading'];
  const urlLower = url.toLowerCase();
  if (spamKeywords.some(kw => urlLower.includes(kw))) score += 25;

  // Spam anchor text
  if (anchorText && spamKeywords.some(kw => anchorText.toLowerCase().includes(kw))) score += 20;

  // Classification
  if (score >= 50) return { level: 'toxic', score, action: 'disavow' };
  if (score >= 30) return { level: 'suspicious', score, action: 'review' };
  if (score >= 15) return { level: 'low-quality', score, action: 'monitor' };
  return { level: 'clean', score, action: 'keep' };
}
```

---

## Anchor Text Diversity (Shannon Entropy)

Healthy backlink profiles have diverse anchor text. Calculate Shannon entropy:

```javascript
function anchorEntropy(anchors) {
  const total = anchors.length;
  if (total === 0) return 0;

  const freq = {};
  anchors.forEach(a => {
    const normalised = a.toLowerCase().trim();
    freq[normalised] = (freq[normalised] || 0) + 1;
  });

  let entropy = 0;
  Object.values(freq).forEach(count => {
    const p = count / total;
    if (p > 0) entropy -= p * Math.log2(p);
  });

  return entropy;
}

// Interpretation:
// entropy < 1.0  → Very low diversity — over-optimised anchors (risky)
// entropy 1.0–2.5 → Moderate diversity — OK but could improve
// entropy 2.5–4.0 → Good diversity — natural-looking profile
// entropy > 4.0  → Excellent diversity — very natural
```

---

## Competitor Gap Finder

Common Crawl-based competitor backlink gap analysis:

```javascript
async function competitorGap(yourDomain, competitorDomain) {
  const yourLinks = await discoverBacklinks(yourDomain);
  const compLinks = await discoverBacklinks(competitorDomain);

  const yourDomains = new Set(yourLinks.map(l => new URL(l.url).hostname));
  const compDomains = new Set(compLinks.map(l => new URL(l.url).hostname));

  // Domains linking to competitor but NOT to you
  const gaps = [...compDomains].filter(d => !yourDomains.has(d));

  return {
    yourBacklinkDomains: yourDomains.size,
    competitorBacklinkDomains: compDomains.size,
    gapDomains: gaps,
    gapCount: gaps.length,
    topOpportunities: gaps.slice(0, 50) // prioritise by authority later
  };
}
```

---

## Paste Importers

Import backlink data from free online tools via copy-paste:

### Supported Formats

| Tool | Format | Detection |
|---|---|---|
| OpenLinkProfiler | TSV/CSV | Headers: "Source URL", "Anchor" |
| Neil Patel (Ubersuggest) | CSV | Headers: "Source Page", "Domain Score" |
| SEO Review Tools | HTML table | `<table>` with "Backlink" column |
| Moz Link Explorer | CSV | Headers: "Target URL", "Linking Domain" |
| Generic HTML tables | HTML | Auto-detect `<table>` with URL columns |

### Auto-Detection

```javascript
function detectImportFormat(text) {
  if (text.includes('Source URL') && text.includes('Anchor')) return 'openlinkprofiler';
  if (text.includes('Source Page') && text.includes('Domain Score')) return 'neilpatel';
  if (text.includes('Target URL') && text.includes('Linking Domain')) return 'moz';
  if (text.includes('<table')) return 'html-table';
  if (text.includes('\t')) return 'tsv';
  if (text.includes(',')) return 'csv';
  return 'unknown';
}
```

---

## Broken Outbound Link Checker

Check all outbound links on a page for broken targets:

```javascript
async function checkBrokenLinks(pageUrl) {
  const res = await fetch(pageUrl);
  const html = await res.text();
  const $ = cheerio.load(html);

  const links = [];
  $('a[href^="http"]').each((i, el) => {
    const href = $(el).attr('href');
    if (href && !href.includes(new URL(pageUrl).hostname)) {
      links.push({ href, text: $(el).text().trim() });
    }
  });

  const results = [];
  for (const link of links) {
    try {
      let res = await fetch(link.href, {
        method: 'HEAD',
        redirect: 'follow',
        signal: AbortSignal.timeout(10000)
      });

      // Fallback: some servers return 405/403 for HEAD
      if (res.status === 405 || res.status === 403) {
        res = await fetch(link.href, {
          method: 'GET',
          redirect: 'follow',
          signal: AbortSignal.timeout(10000)
        });
      }

      // Exclude known bot-blocking status codes
      // LinkedIn returns 999 — it's not broken, just bot-blocking
      // Some CDNs return 503 for bots — not necessarily broken
      const botBlocking = [999, 503];
      if (botBlocking.includes(res.status)) {
        results.push({ ...link, status: res.status, broken: false, note: 'bot-blocking' });
      } else {
        results.push({ ...link, status: res.status, broken: res.status >= 400 });
      }
    } catch (e) {
      results.push({ ...link, status: 0, broken: true, error: e.message });
    }
  }

  return results;
}
```

### Bot-Blocking Exclusions

| Status | Domain | Reason |
|---|---|---|
| 999 | LinkedIn | Custom bot-blocking — link is not broken |
| 503 | Various CDNs | Rate limiting / bot detection — retry before flagging |
| 403 | Twitter/X | Requires login — link is not broken |

---

## Link Extractor

Extract all links from any URL using cheerio:

```javascript
async function extractLinks(url) {
  const res = await fetch(url);
  const html = await res.text();
  const $ = cheerio.load(html);

  const internal = [];
  const external = [];
  const domain = new URL(url).hostname;

  $('a[href]').each((i, el) => {
    const href = $(el).attr('href');
    if (!href || href.startsWith('#') || href.startsWith('mailto:') || href.startsWith('tel:')) return;

    try {
      const resolved = new URL(href, url).href;
      const linkDomain = new URL(resolved).hostname;
      const data = {
        href: resolved,
        text: $(el).text().trim(),
        rel: $(el).attr('rel') || '',
        dofollow: !$(el).attr('rel')?.includes('nofollow')
      };

      if (linkDomain === domain || linkDomain.endsWith('.' + domain)) {
        internal.push(data);
      } else {
        external.push(data);
      }
    } catch (e) { /* invalid URL */ }
  });

  return { internal, external, totalLinks: internal.length + external.length };
}
```
