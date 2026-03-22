# Failure Registry

Consolidated failures confirmed while building 4 production SEO tools.
Every entry below was encountered, debugged, and resolved in real projects.
This is not theoretical — these are battle scars.

---

## SERP_SCRAPE_DEAD

**What failed:** Scraping google.com directly for SERP results.

**Symptoms:** Google returns a JS-only page with `<noscript>` content. No organic
results in the HTML. Happens from any IP (datacenter, residential, mobile).

**Root cause:** Google detects automated requests regardless of headers, proxy type,
or browser fingerprinting.

**Fix:** Use Serper.dev API (`POST https://google.serper.dev/search`). 2,500 free
searches/month. Returns structured JSON with organic results, AI Overview,
featured snippets, PAA, and more.

**Impact:** High — affects any SERP analysis feature.

---

## GOOGLE_TRENDS_RSS_404

**What failed:** Google Trends RSS endpoint returns 404.

**Endpoint:** `https://trends.google.com/trends/trendingsearches/daily/rss?geo=US`

**Symptoms:** HTTP 404. The endpoint is deprecated with no replacement RSS feed.

**Fix:** Use Serper.dev autocomplete endpoint as a trends proxy. For actual trend
data, use DataForSEO keyword volume with 12-month history, or track GSC
impressions over time.

**Impact:** Medium — affects trend monitoring features.

---

## HTTPS_PROXY_AGENT_ESM

**What failed:** `https-proxy-agent` v7+ breaks `require()`.

**Symptoms:** `ERR_REQUIRE_ESM` when doing `const { HttpsProxyAgent } = require('https-proxy-agent')`.

**Root cause:** Version 7+ is ESM-only (`import` syntax). CommonJS `require()` is
not supported.

**Fix:** Pin to v5: `npm install https-proxy-agent@5`. Works with `require()`.

**Impact:** High — breaks all proxy-dependent features (autocomplete, page fetching).

---

## AXIOS_CC_CDX_404

**What failed:** axios returns 404 for Common Crawl CDX API URLs.

**Symptoms:** `axios.get('https://index.commoncrawl.org/CC-MAIN-...')` returns 404
inside an Express server. The same URL works in a browser and with `curl`.

**Root cause:** Unknown. Suspected header or connection handling difference between
axios and native fetch inside Express request handlers.

**Fix:** Use native `fetch()` instead of axios for all Common Crawl CDX calls.
fetch works reliably in the same context where axios fails.

**Impact:** High — breaks backlink discovery.

---

## CC_CDX_ENCODE_URI

**What failed:** `encodeURIComponent()` on domain names for CC CDX queries.

**Symptoms:** Zero results returned. No error — just empty response.

**Root cause:** CC CDX expects the raw domain format (e.g., `example.com`), not
URL-encoded format (e.g., `example%2Ecom`).

**Fix:** Pass domain names raw, without `encodeURIComponent()`:
```javascript
// BAD
const url = `https://index.commoncrawl.org/...?url=${encodeURIComponent(domain)}`;

// GOOD
const url = `https://index.commoncrawl.org/...?url=${domain}`;
```

**Impact:** Medium — causes silent failure in backlink discovery.

---

## DATAFORSEO_BACKLINKS_PAYWALL

**What failed:** DataForSEO backlinks API returns "Payment required".

**Symptoms:** API responds with 402 status. n8n DataForSEO node also fails with
the same error.

**Root cause:** DataForSEO backlinks endpoint requires $100/month minimum balance.
The free $1 signup credit only covers keyword data endpoints, not backlinks.

**Fix:** Use Common Crawl CDX API for free backlink discovery instead. See
`references/backlinks.md` for the complete free alternative stack.

**Impact:** High — eliminates DataForSEO as a backlink source for free-tier users.

---

## SQLITE_CACHE_COLUMN

**What failed:** Cache queries return zero rows despite data existing.

**Symptoms:** `SELECT * FROM serp_cache WHERE key = ?` returns nothing, even though
data was inserted. No error thrown.

**Root cause:** The `serp_cache` table uses `keyword` as the primary key column,
not `key`. The `trend_cache` table uses `key`. Using the wrong column name
produces a valid SQL query that matches nothing.

**Fix:** Always use `keyword` for `serp_cache` and `key` for `trend_cache`.
Do not reuse queries across different cache tables without checking column names.

**Impact:** Medium — causes cache misses, leading to unnecessary API calls.

---

## CHEERIO_OBJECT_SELECTOR

**What failed:** cheerio throws "Unexpected type of selector".

**Symptoms:** Runtime error when calling `$(someObject).find('a')` where
`someObject` is a plain JavaScript object (e.g., `{ title: '...' }`).

**Root cause:** cheerio's `$()` function expects either a string CSS selector or
a DOM element. Passing a plain JS object triggers an internal type error.

**Fix:** Never pass plain objects to `$()`. Only use:
```javascript
$('selector')        // string CSS selector
$(domElement)        // DOM element from .each() callback
```

**Impact:** Medium — crashes audit/scraping processes.

---

## CONTENT_GAP_YOUR_DATA

**What failed:** Content gap comparison returns empty results.

**Symptoms:** The content gap analysis runs but shows no gaps — your page data is
always empty/undefined.

**Root cause:** Field name mismatch. The API endpoint expects `targetUrl` but the
frontend sends `yourUrl`. The backend silently ignores the unknown field.

**Fix:** Use consistent field names across frontend and backend. Pick either
`targetUrl` or `yourUrl` and use it everywhere.

**Impact:** Medium — breaks content gap analysis silently.

---

## GSC_CRON_CREDENTIALS

**What failed:** GSC API calls fail when run via PM2 cron.

**Symptoms:** `GOOGLE_APPLICATION_CREDENTIALS` is undefined when the script runs
via PM2 scheduled restart or cron.

**Root cause:** PM2 does not automatically load `.env` files for cron-triggered
processes. The environment variable is only available if `dotenv.config()` runs
at script startup.

**Fix:** Ensure `require('dotenv').config()` is the first line of your entry point.
Do NOT rely on PM2's `env` config for file paths — use dotenv directly.

**Impact:** Medium — breaks scheduled GSC data pulls.

---

## LINKEDIN_999

**What failed:** Broken link checker flags LinkedIn URLs as broken (status 999).

**Symptoms:** HTTP HEAD/GET requests to linkedin.com return status code 999.

**Root cause:** LinkedIn uses a custom 999 status code for bot-blocking. The
page is not actually broken — it is intentionally rejecting automated requests.

**Fix:** Exclude status 999 from broken link results. Add to bot-blocking
exclusion list alongside 503 from CDNs and 403 from Twitter/X.

**Impact:** Low — causes false positives in broken link reports.

---

## Quick Reference

| Code | One-line Summary | Severity |
|---|---|---|
| SERP_SCRAPE_DEAD | Google scraping dead → use Serper API | Critical |
| GOOGLE_TRENDS_RSS_404 | Trends RSS deprecated → use Serper autocomplete | Medium |
| HTTPS_PROXY_AGENT_ESM | v7+ ESM-only → pin to v5 | Critical |
| AXIOS_CC_CDX_404 | axios fails on CC CDX → use fetch() | Critical |
| CC_CDX_ENCODE_URI | Don't encodeURIComponent domains for CC | Medium |
| DATAFORSEO_BACKLINKS_PAYWALL | $100/month required → use Common Crawl | High |
| SQLITE_CACHE_COLUMN | serp_cache = 'keyword', trend_cache = 'key' | Medium |
| CHEERIO_OBJECT_SELECTOR | $(plainObject) throws → use $(selector) only | Medium |
| CONTENT_GAP_YOUR_DATA | targetUrl vs yourUrl mismatch | Medium |
| GSC_CRON_CREDENTIALS | dotenv must load at startup for PM2 crons | Medium |
| LINKEDIN_999 | Bot-blocking, not broken → exclude from results | Low |
