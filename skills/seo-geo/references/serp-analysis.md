# SERP Analysis

## Critical: Google SERP Scraping Is Dead

**Do not attempt to scrape google.com directly.** As of 2025, Google returns
JS-only noscript pages from any IP address, residential proxy, or user agent.
The HTML returned contains no organic results — just a `<noscript>` redirect.

This applies to:
- Datacenter proxies
- Residential proxies (Webshare, Bright Data, etc.)
- Any custom headers or user agent strings
- Headless browsers (Puppeteer, Playwright) — Google detects and blocks them

**The only reliable method is Serper.dev API.**

---

## Serper.dev API

### Setup

- Sign up at [serper.dev](https://serper.dev)
- Free tier: 2,500 searches/month
- API key via dashboard

### Basic Search

```javascript
async function searchSerper(query, options = {}) {
  const response = await fetch('https://google.serper.dev/search', {
    method: 'POST',
    headers: {
      'X-API-KEY': process.env.SERPER_API_KEY,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      q: query,
      gl: options.country || 'us',
      hl: options.language || 'en',
      num: options.num || 10
    })
  });
  return response.json();
}
```

### Response Structure

```json
{
  "organic": [
    {
      "title": "Page Title",
      "link": "https://example.com/page",
      "snippet": "Description text...",
      "position": 1
    }
  ],
  "aiOverview": {
    "text": "AI-generated summary...",
    "references": [
      { "title": "Source", "link": "https://..." }
    ]
  },
  "answerBox": {
    "title": "Featured Snippet Title",
    "snippet": "Direct answer text...",
    "link": "https://..."
  },
  "peopleAlsoAsk": [
    { "question": "What is...?", "snippet": "Answer...", "link": "https://..." }
  ],
  "knowledgeGraph": {
    "title": "Entity Name",
    "type": "Organization",
    "description": "..."
  },
  "videos": [
    { "title": "Video Title", "link": "https://youtube.com/...", "channel": "..." }
  ],
  "relatedSearches": [
    { "query": "related keyword" }
  ]
}
```

### Other Endpoints

| Endpoint | URL | Use case |
|---|---|---|
| Web search | `google.serper.dev/search` | Standard SERP |
| News | `google.serper.dev/news` | News results |
| Images | `google.serper.dev/images` | Image results |
| Autocomplete | `google.serper.dev/autocomplete` | Suggestions (trends proxy) |

---

## AI Overview Tracking

### Daily Monitoring

Track AI Overview presence per keyword with a daily cron job:

```javascript
async function trackAIOverview(keyword) {
  const data = await searchSerper(keyword);
  return {
    keyword,
    timestamp: new Date().toISOString(),
    hasAIOverview: !!data.aiOverview,
    aiOverviewText: data.aiOverview?.text || null,
    aiOverviewSources: data.aiOverview?.references || [],
    yourDomainCited: data.aiOverview?.references?.some(
      r => r.link?.includes(YOUR_DOMAIN)
    ) || false
  };
}
```

### Metrics to Track

| Metric | Calculation | Why it matters |
|---|---|---|
| AO Rate | Keywords with AI Overview / Total tracked | Overall AI presence in your niche |
| Citation Rate | Keywords where your domain cited / Keywords with AO | Your AI visibility |
| AO Appearance | Date AI Overview first appeared for keyword | Timing of AI expansion |
| AO Disappearance | Date AI Overview removed for keyword | Google testing/rollback |
| Competitor Citation Rate | Same as citation rate but for competitors | Competitive intelligence |

### Storage

SQLite table for AI Overview history:

```sql
CREATE TABLE ai_overview_tracking (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  keyword TEXT NOT NULL,
  date TEXT NOT NULL,
  has_ai_overview INTEGER NOT NULL,
  your_domain_cited INTEGER DEFAULT 0,
  sources_json TEXT,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(keyword, date)
);
```

---

## Content Gap Analysis

### Process

1. Search keyword via Serper
2. Take top 5 organic results
3. Fetch each page via HTTP (with proxy if needed)
4. Parse with cheerio
5. Extract: headings, word count, schema types, outbound links
6. Compare against your page

### Implementation

```javascript
const cheerio = require('cheerio');

async function analyseCompetitor(url) {
  const res = await fetch(url, {
    headers: { 'User-Agent': 'Mozilla/5.0 (compatible; SEOBot/1.0)' }
  });
  const html = await res.text();
  const $ = cheerio.load(html);

  // CRITICAL: Never pass a plain JS object to $() — cheerio throws
  // "Unexpected type of selector". Only use $(domElement) or $('selector').

  const headings = [];
  $('h1, h2, h3').each((i, el) => {
    headings.push({
      tag: el.tagName,
      text: $(el).text().trim()
    });
  });

  const wordCount = $('body').text().split(/\s+/).filter(w => w.length > 0).length;

  const schemas = [];
  $('script[type="application/ld+json"]').each((i, el) => {
    try {
      const json = JSON.parse($(el).html());
      schemas.push(json['@type'] || 'Unknown');
    } catch (e) { /* malformed JSON-LD */ }
  });

  return { url, headings, wordCount, schemas };
}
```

### Gap Report Format

```
Target Keyword: "content gap analysis"
Your Page: https://yourdomain.com/content-gap
Your Word Count: 1,200

Competitor Analysis:
| # | URL | Words | H2 Count | Schema |
|---|-----|-------|----------|--------|
| 1 | competitor-a.com/... | 3,400 | 12 | Article, FAQ |
| 2 | competitor-b.com/... | 2,800 | 9 | Article |
| 3 | competitor-c.com/... | 2,100 | 8 | HowTo |

Gaps Found:
- Missing topics: "competitor analysis tools", "free gap analysis"
- Word count gap: you have 1,200 vs median 2,800
- Missing schema: FAQ (present in 2 of top 5)
- Missing H2: "How to Find Content Gaps" (present in 3 of top 5)
```

### Field Name Warning

When building content gap tools, use consistent field names for the target URL.
A confirmed bug: using `targetUrl` in the API but `yourUrl` in the frontend
causes the comparison to fail silently. Pick one name and use it everywhere.

---

## Content Brief Generation

Auto-generate a content brief from SERP data:

```javascript
function generateBrief(keyword, competitorData, serpData) {
  const wordCounts = competitorData.map(c => c.wordCount);
  const medianWords = wordCounts.sort((a, b) => a - b)[Math.floor(wordCounts.length / 2)];

  // Collect all H2 headings, count frequency
  const h2Freq = {};
  competitorData.forEach(c => {
    c.headings.filter(h => h.tag === 'h2').forEach(h => {
      const normalised = h.text.toLowerCase().trim();
      h2Freq[normalised] = (h2Freq[normalised] || 0) + 1;
    });
  });

  // Must-have topics: present in 3+ of top 5
  const mustHave = Object.entries(h2Freq)
    .filter(([_, count]) => count >= 3)
    .map(([topic]) => topic);

  return {
    keyword,
    recommendedWordCount: Math.max(medianWords, 1500),
    mustHaveTopics: mustHave,
    serpFeatures: {
      hasAIOverview: !!serpData.aiOverview,
      hasFeaturedSnippet: !!serpData.answerBox,
      hasPAA: (serpData.peopleAlsoAsk?.length || 0) > 0,
      hasVideos: (serpData.videos?.length || 0) > 0
    },
    schemaRecommendations: getSchemaRecommendations(serpData, competitorData),
    paaQuestions: serpData.peopleAlsoAsk?.map(p => p.question) || []
  };
}
```

---

## SERP Feature History

Track how SERP features change over time per keyword:

```sql
CREATE TABLE serp_feature_history (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  keyword TEXT NOT NULL,
  date TEXT NOT NULL,
  has_ai_overview INTEGER DEFAULT 0,
  has_featured_snippet INTEGER DEFAULT 0,
  has_paa INTEGER DEFAULT 0,
  has_knowledge_panel INTEGER DEFAULT 0,
  has_videos INTEGER DEFAULT 0,
  has_local_pack INTEGER DEFAULT 0,
  top_3_domains TEXT,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP,
  UNIQUE(keyword, date)
);
```

Weekly diff report:
- Which keywords gained/lost AI Overviews
- Which keywords gained/lost featured snippets
- Domain shuffles in top 3 positions
- New SERP features appearing for tracked keywords

---

## Serper Budget Management

2,500 free searches/month = ~83/day.

### Budget Allocation

| Task | Searches/day | Monthly |
|---|---|---|
| Daily rank tracking (20 keywords) | 20 | 600 |
| Daily AI Overview monitoring (20 keywords) | 0 | 0 (same queries) |
| Weekly content gap analysis (5 keywords) | ~25/week | 100 |
| Ad-hoc research | ~10 | 300 |
| **Total** | | **~1,000** |

This leaves 1,500 searches/month as buffer. If you need more:
- Paid plans start at $50/month for 50,000 searches
- Cache results aggressively (SERP data doesn't change hourly)
- Combine rank tracking + AI Overview tracking in single queries
