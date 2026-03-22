# Keyword Research — Free Stack

## Google Autocomplete

**Endpoint:** `https://suggestqueries.google.com/complete/search?client=firefox&q=QUERY`

Returns JSON array of suggestions. Stable, free, no API key.

### Deep Alphabet Soup Mode

Append each letter a–m (or a–z) to the seed keyword to expand suggestions:

```
seed = "seo tools"
queries = ["seo tools a", "seo tools b", ..., "seo tools m"]
```

Each returns up to 10 suggestions = 130+ keyword ideas per seed.

### Proxy Requirement

Google rate-limits autocomplete from datacenter IPs. Use Webshare residential proxies:

```javascript
const { HttpsProxyAgent } = require('https-proxy-agent'); // v5 ONLY — v7+ is ESM
const agent = new HttpsProxyAgent('http://user:pass@proxy.webshare.io:80');

const res = await fetch(url, { agent });
```

**CRITICAL:** Use `https-proxy-agent@5`, NOT v7+. Version 7+ is ESM-only and breaks
`require()` in Node.js CommonJS projects.

---

## YouTube Autocomplete

**Endpoint:** `https://suggestqueries.google.com/complete/search?client=youtube&ds=yt&q=QUERY`

Returns video-intent keywords. Same proxy requirements as Google Autocomplete.

Useful for:
- Identifying video-first topics (tutorials, reviews, how-tos)
- Finding keywords where video results dominate SERPs
- Content format decisions (write a blog post vs record a video)

### Deep Alphabet Soup for YouTube

Same a–m expansion works:
```
seed = "seo tutorial"
queries = ["seo tutorial a", "seo tutorial b", ..., "seo tutorial m"]
```

---

## People Also Ask (PAA)

Extracted from Serper.dev SERP data — not scraped directly.

```javascript
const response = await fetch('https://google.serper.dev/search', {
  method: 'POST',
  headers: {
    'X-API-KEY': process.env.SERPER_API_KEY,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({ q: 'keyword research tools', gl: 'us' })
});
const data = await response.json();
const paaQuestions = data.peopleAlsoAsk?.map(p => p.question) || [];
```

PAA questions are high-value for:
- FAQ schema content
- Blog post subheadings
- AI Overview optimisation (direct answers to questions)

---

## Related Searches

Also from Serper SERP data:

```javascript
const relatedSearches = data.relatedSearches?.map(r => r.query) || [];
```

These are semantic variations Google associates with the query.
Use for topic cluster expansion and internal linking strategy.

---

## Google Search Console (GSC) API

Queries you already rank for — the most underrated keyword source.

### Setup

1. Create a GCP project + service account
2. Download credentials JSON
3. Grant service account access in GSC property settings
4. Set `GOOGLE_APPLICATION_CREDENTIALS=/path/to/credentials.json` in `.env`

### Query

```javascript
const { google } = require('googleapis');
const auth = new google.auth.GoogleAuth({
  scopes: ['https://www.googleapis.com/auth/webmasters.readonly']
});
const searchconsole = google.searchconsole({ version: 'v1', auth });

const result = await searchconsole.searchanalytics.query({
  siteUrl: 'https://yourdomain.com',
  requestBody: {
    startDate: '2026-01-01',
    endDate: '2026-03-21',
    dimensions: ['query'],
    rowLimit: 1000
  }
});
```

Returns per query:
- **Impressions** — how often you appeared
- **Clicks** — how often users clicked
- **Position** — average ranking position
- **CTR** — click-through rate

### GSC Keyword Opportunities

| Signal | Action |
|---|---|
| High impressions, low clicks | Improve title/meta description (CTR optimisation) |
| Position 5–15 | "Striking distance" — optimise content to push into top 5 |
| High CTR, low impressions | Niche wins — create more content in this topic |
| Declining position over 30 days | Content refresh needed |

---

## Google Trends (via Serper Proxy)

**Google Trends RSS is deprecated and returns 404.** Do not use `trends.google.com/trends/trendingsearches/daily/rss`.

Instead, use Serper autocomplete + news endpoints as a trends proxy:
- Autocomplete shows currently popular completions
- News endpoint shows trending topics in your niche

For actual trend data (rising/declining/stable), use DataForSEO keyword data
if available, or track GSC impressions over time as a proxy.

---

## DataForSEO (Optional)

Free $1 credit on signup — enough for ~200 keyword lookups.

Returns:
- Monthly search volume
- CPC (cost per click)
- Competition level
- 12-month search trend data

**Endpoint:** `POST https://api.dataforseo.com/v3/keywords_data/google_ads/search_volume/live`

Not required — the free stack above covers most use cases.

---

## AI Clustering via Groq

Send a batch of keywords to Groq (llama-3.1-8b-instant) for semantic clustering.

```javascript
const Groq = require('groq-sdk');
const groq = new Groq({ apiKey: process.env.GROQ_API_KEY });

const completion = await groq.chat.completions.create({
  model: 'llama-3.1-8b-instant',
  messages: [{
    role: 'user',
    content: `Cluster these keywords into semantic topic groups. Return JSON with group names as keys and keyword arrays as values:\n\n${keywords.join('\n')}`
  }],
  temperature: 0.1
});
```

### What Clustering Gives You

- **Topic clusters** for content strategy (pillar pages + supporting posts)
- **Internal linking maps** (link cluster members to pillar)
- **Content calendar** priorities (cluster size = demand signal)
- **Cannibalisation detection** (multiple pages targeting same cluster)

Free tier: 30 requests/minute, 14,400 tokens/minute. More than enough for keyword work.

---

## Monthly Search Trends

12-month sparkline data from DataForSEO or GSC impression tracking:

| Pattern | Classification | Action |
|---|---|---|
| Upward slope last 3 months | Rising | Prioritise — growing demand |
| Flat ±10% | Stable | Standard priority |
| Downward slope last 3 months | Declining | Lower priority unless evergreen |
| Seasonal spike | Seasonal | Time content for pre-spike |

---

## Intent Classification

Rule-based — no API needed. Check keyword tokens against signal word lists:

```javascript
function classifyIntent(keyword) {
  const kw = keyword.toLowerCase();
  if (/\b(buy|price|discount|coupon|deal|order|shop|purchase|cheap|affordable)\b/.test(kw))
    return 'transactional';
  if (/\b(best|top|review|comparison|vs|versus|alternative|recommend)\b/.test(kw))
    return 'commercial';
  if (/\b(how|what|why|when|where|guide|tutorial|learn|explain|definition)\b/.test(kw))
    return 'informational';
  if (/\b(login|dashboard|app|official|site|account|sign.?in)\b/.test(kw))
    return 'navigational';
  if (/\b(near me|in \w+|directions|open now|hours|local)\b/.test(kw))
    return 'local';
  return 'informational'; // default
}
```

---

## Proxy Difficulty Score

0–100 difficulty score without paid APIs:

```
Base score = 0

For each of top 10 organic results:
  +8 if domain is Wikipedia, Reddit, .gov, .edu
  +5 if domain is major brand (Forbes, NYT, HubSpot, etc.)
  +3 for other established domains

SERP features:
  +15 if AI Overview present
  +10 if featured snippet present
  +5 if knowledge panel present

Brand adjustment:
  -20 if your domain already in top 10
  -10 if your domain in top 20

Cap at 0–100
```

This gives a directionally useful difficulty score that correlates well with paid
tools but costs nothing. Not a replacement for Ahrefs/Moz DR-based difficulty
for competitive analysis, but good enough for prioritisation.

---

## Brand Detection

If the keyword contains your brand name or your domain already ranks in the top 20,
reduce the difficulty score. Branded queries are low-hanging fruit — you should
rank #1 for your own brand.

```javascript
function adjustForBrand(score, keyword, ownDomain, brandNames) {
  const kw = keyword.toLowerCase();
  for (const brand of brandNames) {
    if (kw.includes(brand.toLowerCase())) return Math.max(0, score - 30);
  }
  // ownDomain check happens during SERP analysis
  return score;
}
```

---

## Complete Research Workflow

```
1. Seed keywords (manual list + GSC top queries)
   ↓
2. Expand via Google Autocomplete (deep alphabet soup a–m)
   ↓
3. Expand via YouTube Autocomplete (deep alphabet soup a–m)
   ↓
4. Pull PAA + Related Searches from Serper
   ↓
5. Deduplicate all keywords
   ↓
6. Classify intent (rule-based)
   ↓
7. Cluster via Groq AI
   ↓
8. Score difficulty (proxy score from SERP data)
   ↓
9. Optional: enrich with DataForSEO volume/CPC
   ↓
10. Prioritise: high volume + low difficulty + rising trend + matching intent
```
