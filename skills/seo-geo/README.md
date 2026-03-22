# seo-geo-skill

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://github.com/arturseo-geo/seo-geo-skill/releases)
[![Licence](https://img.shields.io/badge/licence-MIT-green.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

The most comprehensive SEO and GEO (Generative Engine Optimisation) skill for
[Claude Code](https://claude.com/claude-code) — keyword research, SERP analysis,
technical audits, content optimisation, backlink intelligence, and AI visibility
monitoring.

Built by [Artur Ferreira](https://linkedin.com/in/arturgeo) @
[The GEO Lab](https://thegeolab.net)
&middot; [GitHub](https://github.com/arturseo-geo)
&middot; [𝕏 @TheGEO_Lab](https://x.com/TheGEO_Lab)
&middot; [Reddit](https://www.reddit.com/user/Alternative_Teach_74/)

---

## What makes this different

This skill is **not theoretical**. Every recommendation, warning, and code snippet
comes from building 4 production SEO tools — a keyword research platform, a SERP
tracker, a technical audit crawler, and a backlink intelligence system. The
[failure registry](references/failures.md) alone will save you days of debugging.

**Key insights you won't find elsewhere:**
- Google SERP scraping is dead — documented why and what to use instead
- DataForSEO backlinks requires $100/month — documented the free alternative stack
- `https-proxy-agent` v7+ breaks CommonJS — must pin to v5
- axios fails on Common Crawl CDX inside Express — must use native fetch
- cheerio throws on plain objects — exact reproduction and fix
- LinkedIn returns 999 status — not broken, just bot-blocking
- SQLite cache tables need different column names — silent failure if mixed

---

## Platforms & tools covered

### Data Sources
- **Serper.dev** — SERP data, AI Overviews, featured snippets, PAA (2,500 free/month)
- **Google Search Console** — queries, impressions, clicks, position, CTR
- **Google Autocomplete** — real-time keyword suggestions (via Webshare proxies)
- **YouTube Autocomplete** — video-intent keywords
- **Common Crawl CDX** — free backlink discovery across crawl indexes
- **Google PSI API** — Core Web Vitals (LCP, INP, CLS, FCP, TTFB, TBT)
- **DataForSEO** — optional, search volume and CPC ($1 free credit)

### AI
- **Groq** (llama-3.1-8b-instant) — keyword clustering, audit fix suggestions, sentiment

### Infrastructure
- **Node.js 20** + Express + cheerio + SQLite
- **PM2** — process management and cron scheduling
- **Nginx** — reverse proxy with basic auth, SSL, noindex headers
- **Webshare** — residential proxy rotation for Google requests

---

## Install

### Method 1: Plugin Marketplace
```
/plugin install arturseo-geo@seo-geo-skill
```

### Method 2: Manual Clone
```bash
git clone https://github.com/arturseo-geo/seo-geo-skill.git ~/.claude/skills/seo-geo
```

### Method 3: Skills CLI
```bash
npx skills add arturseo-geo/seo-geo-skill
```

---

## File structure

```
seo-geo-skill/
├── .claude-plugin/
│   └── plugin.json              Plugin metadata
├── .github/
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug-report.md        Bug report template
│   │   └── platform-update.md   Tool/API update template
│   └── pull_request_template.md PR template
├── references/
│   ├── keyword-research.md      Free keyword research stack
│   ├── serp-analysis.md         SERP analysis via Serper.dev
│   ├── technical-audit.md       Technical SEO audit
│   ├── backlinks.md             Free backlink intelligence
│   ├── infrastructure.md        Building SEO tools
│   └── failures.md              Consolidated failure registry
├── .gitignore
├── CONTRIBUTING.md              Contribution guidelines
├── LICENSE                      MIT licence
├── README.md                    This file
├── SECURITY.md                  Security policy
├── SKILL.md                     Main skill file (~150 tokens when loaded)
└── project-context.md           Customisable brand/setup context
```

---

## Changelog

### v1.0.0 (March 2026)

**Initial release** — complete SEO and GEO skill covering:

- **Keyword research**: Google/YouTube autocomplete with deep alphabet soup, PAA,
  related searches, GSC API, AI clustering via Groq, intent classification,
  proxy difficulty scoring, brand detection
- **SERP analysis**: Serper.dev API integration, AI Overview tracking, content gap
  analysis, content brief generation, SERP feature history
- **Technical audit**: Sitemap-first crawl, per-page checks (title, meta, H1,
  canonical, viewport, OG, schema, images, word count), site-wide checks
  (duplicates, orphan pages, deep pages), schema validation, Core Web Vitals
  via PSI API, AI audit fix suggestions via Groq
- **Backlink intelligence**: Common Crawl CDX discovery, live HTTP verification,
  context-based quality scoring (0–100, A–F grades), toxic link detection,
  anchor diversity (Shannon entropy), competitor gap finder, paste importers,
  broken outbound link checker, link extractor
- **Infrastructure**: Webshare proxies (https-proxy-agent v5), SQLite caching,
  PM2 deployment, Nginx reverse proxy with auth/SSL/noindex, GSC service
  account setup, Serper.dev and Groq integration
- **Failure registry**: 11 confirmed failures from building 4 production SEO tools

---

## Attribution

Built from real production experience building SEO tools at
[The GEO Lab](https://thegeolab.net). This skill documents what actually works
(and what doesn't) when building SEO tooling with free and low-cost APIs.

Platform data derives from official documentation for Serper.dev, Google Search
Console, Google PSI API, Common Crawl, DataForSEO, Groq, and Webshare.

Built following Anthropic's [Agent Skills specification](https://docs.anthropic.com/en/docs/claude-code/skills).

---

## Licence

[MIT](LICENSE) — use it however you want.
