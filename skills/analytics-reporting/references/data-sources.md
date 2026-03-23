# Data Sources Reference

## Google Search Console API

**Endpoint:** `https://www.googleapis.com/webmasters/v3/sites/{siteUrl}/searchAnalytics/query`

**Authentication:** OAuth 2.0 (service account or personal token)

**Query dimensions:** date, page, query, country, device, searchType

**Metrics:** clicks, impressions, ctr, position

**Limits:**
- 5,000 rows per query
- Lag: 2-3 days behind real-time
- Retention: 16 months

**MCP:** `reference_mcp_gsc.md` — Full query examples

## Google Analytics 4 API

**Property ID:** 528383456 (thegeolab.net)

**Endpoint:** `https://analyticsreporting.googleapis.com/v4/reports:batchGet`

**Authentication:** OAuth 2.0 service account

**Dimensions:** date, pagePath, eventName, userCity, deviceCategory, source, medium

**Metrics:** sessions, users, screenPageViews, engagementRate, conversions, userEngagementDuration

**Reporting lag:**
- Real-time: 0-5 minutes (low precision)
- Standard: 24-48 hours (high precision)

**Sampling:** Active if <1,000 sessions/day per dimension combo

**MCP:** `reference_mcp_ga4.md` — Full query examples

## PostgreSQL (OpenSEO Internal DB)

**Connection:**
```
Host: 100.87.191.9
Port: 5432
Database: openseo
User: (see VPS secrets)
```

**Key tables:**
- `pages` — Crawled pages, metadata, scores
- `backlinks` — Incoming links, anchor text, authority
- `keywords` — Tracked keywords, positions, serps
- `job_logs` — System job execution logs

**SSH example:**
```bash
ssh root@100.87.191.9 \
  'psql -U postgres -d openseo -c "SELECT * FROM pages LIMIT 5;"'
```

**Retention:** 24+ months of historical snapshots

## Redis Cache

**Connection:**
```
Host: 100.87.191.9
Port: 6379
(no auth required on internal VPS)
```

**Key patterns:**
- `job:*` — BullMQ job state
- `queue:*` — Queue depth, processing time
- `rank:*` — Rank tracking cache
- `link:*` — Link profile cache

**Common commands:**
```bash
redis-cli -h 100.87.191.9 -p 6379 \
  KEYS 'job:*' | wc -l  # Total jobs in system

redis-cli -h 100.87.191.9 -p 6379 \
  LRANGE "queue:seo-intelligence" 0 -1  # Pending jobs
```

**Retention:** 7 days rolling window

## WordPress REST API

**Base URL:** `https://thegeolab.net/wp-json/wp/v2/`

**Authentication:** OAuth 2.0 app password (see MCP WordPress config)

**Endpoints:**
- `/posts` — Blog posts
- `/pages` — Static pages
- `/media` — Uploaded images
- `/categories` — Post categories
- `/users` — Authors

**Key fields for analytics:**
- `id` — Post ID
- `date` — Publish timestamp
- `slug` — URL slug for matching GA4
- `link` — Canonical URL
- `title` — Post title
- `meta.seo_score` — Custom field (if available)

**Example query:**
```bash
curl -H "Authorization: Bearer [TOKEN]" \
  'https://thegeolab.net/wp-json/wp/v2/posts?per_page=100&orderby=modified'
```

**MCP:** `reference_mcp_wordpress.md` — Full API examples

## VPS Service APIs

### seo-intelligence (:3001)

**Base URL:** `http://100.87.191.9:3001`

**Endpoints:**
- `GET /api/ranks` — Rank tracking snapshots
- `GET /api/alerts` — Rank drop alerts
- `GET /api/competitors` — Competitor ranking data

**Example:**
```bash
curl 'http://100.87.191.9:3001/api/ranks?domain=thegeolab.net'
```

### geolab-links (:3002)

**Base URL:** `http://100.87.191.9:3002`

**Endpoints:**
- `GET /api/pages` — Page metrics, backlink count
- `GET /api/backlinks` — Backlink details
- `GET /api/competitors` — Competitor link profiles

**Requires PostgreSQL + Redis access**

### geolab-backlinks (:4000)

**Base URL:** `http://100.87.191.9:4000`

**Endpoints:**
- `GET /api/backlinks` — Full backlink snapshots
- `GET /api/domains` — Referring domain authority
- `GET /api/competitors` — Competitor backlink gap

### geolab-keywords (:4001)

**Base URL:** `http://100.87.191.9:4001`

**Endpoints:**
- `GET /api/keywords` — Tracked keyword positions
- `GET /api/serps` — SERP feature snapshots
- `GET /api/gaps` — Content gap analysis vs competitors

### geolab-writer (:4002)

**Base URL:** `http://100.87.191.9:4002`

**Endpoints:**
- `GET /api/articles` — Published content metadata
- `GET /api/performance` — Article performance tags
- `GET /api/topics` — Topic coverage tracking

## SSH Access & Tunneling

**VPS SSH:**
```bash
ssh root@100.87.191.9  # Direct (Tailscale)
```

**Query PostgreSQL via SSH:**
```bash
ssh root@100.87.191.9 \
  'psql -U postgres -d openseo -c "SELECT * FROM pages LIMIT 1;"'
```

**Query Redis via SSH:**
```bash
ssh root@100.87.191.9 \
  'redis-cli -p 6379 KEYS "*" | head -20'
```

**Forward ports locally (if needed):**
```bash
ssh -L 5432:localhost:5432 root@100.87.191.9  # PostgreSQL
ssh -L 6379:localhost:6379 root@100.87.191.9  # Redis
```

## Data Freshness & Retention

| Source | Freshness | Retention | Lag |
|--------|-----------|-----------|-----|
| GSC | Daily | 16 months | 2-3 days |
| GA4 | Hourly (real-time) / Daily (standard) | Configurable | 24-48 hours |
| PostgreSQL | Real-time snapshots | 24+ months | 0 hours |
| Redis | Real-time | 7 days | 0 hours |
| WordPress | Real-time | Unlimited | 0 hours |
| VPS APIs | Real-time | Varies by service | 0 hours |

**Best practice:** Always query most recent data first (`ORDER BY timestamp DESC`).
