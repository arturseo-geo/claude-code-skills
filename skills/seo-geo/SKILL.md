---
name: seo-geo-skill
description: >
  The complete SEO and GEO (Generative Engine Optimisation) skill — keyword
  research, SERP analysis, technical audits, content optimisation, backlink
  intelligence, and AI visibility monitoring. Use this skill whenever the user
  asks about SEO, GEO, keyword research, SERP analysis, Google Search Console,
  technical audit, site audit, backlinks, link building, backlink analysis,
  AI visibility, AI Overviews, schema markup, structured data, Core Web Vitals,
  page speed, content gap analysis, content briefs, anchor text, referring
  domains, broken links, orphan pages, meta descriptions, title tags, H1 tags,
  canonical tags, sitemap, robots.txt, crawl budget, link graph, featured
  snippets, People Also Ask, related searches, search intent, keyword
  clustering, topic clusters, SERP features, AI audit fixes, Serper.dev,
  DataForSEO, Common Crawl, Google Trends, or any search engine optimisation
  and AI search visibility task. Google SERP scraping is DEAD — always use
  Serper.dev API. DataForSEO backlinks requires $100/month — use free Common
  Crawl alternatives. Groq (llama-3.1-8b) is free for AI clustering and
  audit fix suggestions.
---

## Reference Files — Load When Relevant

| Topic | File | Load when... |
|---|---|---|
| Keyword research stack | `references/keyword-research.md` | Finding keywords, clustering, search volume, intent |
| SERP analysis | `references/serp-analysis.md` | Analysing SERPs, AI Overviews, content gaps, briefs |
| Technical audit | `references/technical-audit.md` | Site audits, crawling, meta tags, schema, Core Web Vitals |
| Backlink intelligence | `references/backlinks.md` | Backlink discovery, toxic scoring, competitor gaps, broken links |
| Infrastructure | `references/infrastructure.md` | Building tools, proxies, caching, deployment, APIs |
| Failure registry | `references/failures.md` | Debugging, known pitfalls, API failures, library bugs |

## Project Context

If `project-context.md` exists in the skill directory or `.claude/` folder, read it
before any task. It contains brand details, website URLs, API credentials paths,
infrastructure setup, and content defaults — avoids re-explaining context each session.

## Workflow Overview

```
1. Keyword Research    → Find opportunities, cluster by intent, assess difficulty
2. SERP Analysis       → Analyse top results, track AI Overviews, generate content briefs
3. Technical Audit     → Crawl site, check on-page SEO, validate schema, measure Core Web Vitals
4. Content Optimisation → Fill content gaps, optimise meta/headings/schema, improve AI extractability
5. Backlink Intelligence → Discover links, score quality, find toxic links, analyse competitor gaps
6. Monitoring          → Track rankings, AI Overview rates, backlink changes, technical health
```

## Step 1: Keyword Research

### Free Stack (No Paid APIs Required)

| Source | What it gives you | Cost |
|---|---|---|
| Google Autocomplete | Real-time query suggestions | Free (use Webshare proxies) |
| YouTube Autocomplete | Video-intent keywords | Free |
| People Also Ask | Question-format keywords | Free (via Serper SERP data) |
| Related Searches | Semantic variations | Free (via Serper SERP data) |
| GSC API | Queries you already rank for | Free (service account) |
| Serper Autocomplete | Google Trends proxy | Free (2,500/month) |
| DataForSEO | Volume, CPC, competition | Optional ($1 free credit) |

### AI Clustering

Send keywords to Groq (llama-3.1-8b-instant) — returns semantic topic clusters
with parent topics and sub-topics. Free tier, no credit card required.

### Intent Classification

Rule-based classifier — no API needed:
- **Transactional**: buy, price, discount, coupon, deal, order, shop
- **Commercial**: best, top, review, comparison, vs, alternative
- **Informational**: how, what, why, when, guide, tutorial, learn
- **Navigational**: brand name, login, dashboard, app, official
- **Local**: near me, in [city], directions, open now, hours

### Proxy Difficulty Score

0–100 score calculated without paid APIs:
- Count strong domains in top 10 (Wikipedia, Reddit, .gov, .edu, major brands)
- Detect AI Overview presence (+15 difficulty)
- Detect featured snippet (+10 difficulty)
- Brand detection — owned domain queries get reduced difficulty

See `references/keyword-research.md` for deep alphabet soup mode, monthly trends,
sparkline data, and the full implementation stack.

## Step 2: SERP Analysis

### Critical Rule

**Google SERP scraping is dead.** Google returns JS-only noscript pages from any
IP, proxy, or user agent. Do not attempt to scrape google.com directly.

**Use Serper.dev API instead:**
- 2,500 free searches/month
- `POST https://google.serper.dev/search`
- Returns: organic results, AI Overview, featured snippets, PAA, knowledge panel,
  videos, related searches — all structured JSON

### AI Overview Tracking

Daily cron job tracks AI Overview presence per keyword. Calculate:
- Overall AI Overview rate (percentage of tracked keywords with AO)
- Per-keyword AO history (appeared/disappeared over time)
- Content cited in AI Overviews (your domain vs competitors)

### Content Gap Analysis

For any target keyword:
1. Pull top 5 SERP results via Serper
2. Fetch and parse each page (cheerio)
3. Compare: headings, word count, schema types, topics covered
4. Generate gap report: what competitors cover that you don't

### Content Brief Generation

Auto-generate briefs from SERP data:
- Recommended word count (median of top 5)
- Must-have topics (present in 3+ of top 5)
- Gap topics (present in competitors, missing from your page)
- Schema suggestions based on SERP features
- Recommended headings structure

See `references/serp-analysis.md` for SERP feature history tracking,
content brief templates, and implementation details.

## Step 3: Technical Audit

### Crawl Strategy

Sitemap-first crawl:
1. Fetch `/sitemap.xml` (handle sitemap index with nested sub-sitemaps)
2. Extract all URLs
3. Fetch each page, parse with cheerio
4. Run per-page and site-wide checks

### Per-Page Checks

| Check | Pass Criteria |
|---|---|
| Title | Present, 30–60 characters |
| Meta description | Present, 120–160 characters |
| H1 | Exactly one per page |
| Canonical | Present, self-referencing or valid |
| Viewport | Present (`width=device-width`) |
| OG tags | og:title, og:description, og:image present |
| Schema | Valid JSON-LD with required fields |
| Images | All `<img>` have non-empty `alt` attributes |
| Word count | ≥300 for content pages |
| HTML size | Flag if >1 MB |

### Site-Wide Checks

- Duplicate titles across pages
- Duplicate meta descriptions across pages
- Orphan pages (no inbound internal links)
- Deep pages (>3 clicks from homepage)

### Schema Validation

JSON-LD required fields per type:
- **Article**: headline, datePublished, author, image
- **FAQ**: mainEntity with Question + acceptedAnswer
- **HowTo**: name, step[] with text
- **Product**: name, offers with price + priceCurrency
- **Organization**: name, url, logo

### Core Web Vitals

Google PSI API (free, no key required for basic use):
- LCP (Largest Contentful Paint) — target < 2.5s
- FID/INP (Interaction to Next Paint) — target < 200ms
- CLS (Cumulative Layout Shift) — target < 0.1
- FCP, TTFB, TBT as supplementary metrics

### AI Audit Fix Suggestions

Send failing checks to Groq (llama-3.1-8b-instant):
- Generate replacement titles at correct length
- Write meta descriptions from page content
- Suggest alt text from image context
- Flag thin content pages with expansion recommendations

See `references/technical-audit.md` for cheerio bug workarounds,
link graph construction, and full implementation details.

## Step 4: Content Optimisation

### On-Page SEO Checklist

- [ ] Primary keyword in title (front-loaded)
- [ ] Primary keyword in H1
- [ ] Primary keyword in first 100 words
- [ ] Secondary keywords in H2/H3 headings
- [ ] Internal links to related content (3–5 minimum)
- [ ] External links to authoritative sources (1–3)
- [ ] Schema markup matching content type
- [ ] Images with descriptive alt text containing keywords
- [ ] Meta description with CTA and primary keyword
- [ ] URL slug contains primary keyword (short, no stop words)

### AI Visibility (GEO) Optimisation

- Structure content with clear headings and bullet points
- Include direct, quotable answers to common questions
- Add FAQ schema for question-based queries
- Use statistics and citations that AI can extract
- Ensure schema markup is comprehensive and accurate
- Monitor AI Overview citations for your domain

## Step 5: Backlink Intelligence

### Free Discovery Stack

| Method | What it finds | Limitation |
|---|---|---|
| Common Crawl CDX | Historical backlinks across 3+ indexes | Not real-time |
| Live HTTP verification | Current link status | Slow for large sets |
| Paste importers | Links from free tools (Moz, Neil Patel, etc.) | Manual paste step |

### Quality Scoring

Each backlink scored 0–100 with A–F grade:
- **Link location**: content (+40) > sidebar (+15) > footer (+5) > nav (+5)
- **Anchor quality**: keyword-rich (+20) > branded (+15) > generic (+5) > naked URL (+3)
- **Link density**: fewer outbound links on page = higher score
- **Page freshness**: recent content scores higher
- **Follow status**: dofollow (+15) > nofollow (+5)

### Toxic Link Detection

4-tier classification based on:
- Spam TLDs (.xyz, .top, .buzz, .click, .loan, .work)
- Numeric-heavy domains
- Excessive hyphens in domain
- Spam keywords in URL path (casino, pharmacy, payday, etc.)

### Competitor Gap Analysis

Common Crawl-based diff:
1. Query CC for domains linking to competitor
2. Query CC for domains linking to you
3. Diff = opportunity domains (link to them, not you)

See `references/backlinks.md` for anchor diversity (Shannon entropy),
broken outbound link checking, and paste importer formats.

## Step 6: Monitoring

### Recommended Cron Schedule

| Task | Frequency | Tool |
|---|---|---|
| Rank tracking (via Serper) | Daily | PM2 cron |
| AI Overview monitoring | Daily | PM2 cron |
| GSC data pull | Weekly | PM2 cron |
| Technical audit | Weekly | PM2 cron |
| Backlink discovery | Monthly | PM2 cron |
| Core Web Vitals | Monthly | PSI API |

### Alert Triggers

- Rank drop >5 positions for tracked keyword
- AI Overview appears/disappears for tracked keyword
- New backlink from high-authority domain
- Technical issue detected (broken page, missing schema)
- Core Web Vitals regression

## Quick Decision Trees

### "I need more organic traffic"

```
Is your site technically sound?
├── No → Run Technical Audit (Step 3)
└── Yes
    ├── Do you have keyword targets?
    │   ├── No → Run Keyword Research (Step 1)
    │   └── Yes
    │       ├── Are you ranking for them?
    │       │   ├── No → Run SERP Analysis + Content Optimisation (Steps 2+4)
    │       │   └── Yes, but low positions
    │       │       ├── Content gap? → Fill gaps (Step 2)
    │       │       └── Authority gap? → Backlink Intelligence (Step 5)
    │       └── Are you visible in AI Overviews?
    │           ├── No → GEO Optimisation (Step 4)
    │           └── Yes → Monitor and maintain (Step 6)
```

### "My rankings dropped"

```
When did it drop?
├── After a Google update → Check Search Status Dashboard, wait 2 weeks
├── After site changes → Technical Audit (Step 3), check canonical/redirect issues
├── Gradual decline
│   ├── Content freshness issue → Update content, add new sections
│   ├── Competitors improved → SERP Analysis (Step 2), Content Gap
│   └── Lost backlinks → Backlink monitoring, re-acquisition outreach
```

### "I want to track AI visibility"

```
1. Set up daily AI Overview tracking via Serper (Step 2)
2. Monitor citation rate for your domain in AI Overviews
3. Optimise content for extractability (Step 4 — GEO)
4. Add comprehensive schema markup (Step 3)
5. Track changes weekly in monitoring dashboard (Step 6)
```
