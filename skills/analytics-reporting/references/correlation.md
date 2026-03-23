# Multi-Source Data Correlation

## GSC → GA4 Join Strategy

**Problem:** GSC reports 1-3 days delayed; GA4 reports ~24h delayed; different date cutoffs

**Solution: Lag-Aware Join**

```sql
-- Pseudo-code (PostgreSQL)
SELECT 
  gsc.date,
  gsc.page,
  gsc.impressions,
  gsc.clicks,
  gsc.ctr,
  gsc.position,
  ga4.sessions,
  ga4.users,
  ga4.conversions,
  CASE WHEN gsc.date < (TODAY() - INTERVAL '3 days') THEN 'final'
       WHEN gsc.date < (TODAY() - INTERVAL '1 day') THEN 'preliminary'
       ELSE 'real-time' END as data_quality
FROM gsc_impressions gsc
LEFT JOIN ga4_pages ga4 
  ON gsc.page = ga4.page 
  AND gsc.date = ga4.date  -- exact match preferred
  OR gsc.date BETWEEN ga4.date - INTERVAL '2 days' AND ga4.date
WHERE gsc.date >= (TODAY() - INTERVAL '90 days')
ORDER BY gsc.date DESC;
```

**Key points:**
- Only join 3+ days in past for "final" data
- Use 2-day window if less than 3 days old
- Always note data_quality level in report

## GA4 → PostgreSQL (WordPress) Join

**Problem:** GA4 stores URLs; PostgreSQL stores post_id; need to match

**Solution: URL → Post ID Resolution**

```sql
-- Step 1: Extract slug from GA4 URL
WITH ga4_slugs AS (
  SELECT 
    ga4.date,
    regexp_replace(ga4.page_path, '^/([a-z0-9-]+)/?$', '1') as slug,
    ga4.sessions,
    ga4.users
  FROM ga4_pages
  WHERE ga4.page_path ~ '^/[a-z0-9-]+/?$'  -- blog post pattern
)
-- Step 2: Join to WordPress via post_name
SELECT 
  gs.date,
  wp.ID as post_id,
  wp.post_title,
  wp.post_date,
  gs.slug,
  gs.sessions,
  gs.users
FROM ga4_slugs gs
LEFT JOIN wp_posts wp 
  ON LOWER(wp.post_name) = LOWER(gs.slug)
  AND wp.post_type = 'post'
  AND wp.post_status = 'publish'
WHERE wp.ID IS NOT NULL;
```

**Confidence:** 95% if exact match; 85% if fuzzy match; 0% if no match

## WordPress → VPS Service Enrichment

**Query post metadata from WordPress REST API:**

```bash
curl -s "https://thegeolab.net/wp-json/wp/v2/posts/{post_id}" | jq '.'
```

**Extract:**
- `id` — WordPress post ID
- `title` — Post title
- `link` — Canonical URL
- `date` — Publish date
- `yoast_head` — RankMath/Yoast SEO meta (if available)
- `meta.seo_score` — Internal SEO score (custom field)
- `_links.about.href` — User/author info

**Confidence:** 100% (direct source)

## PostgreSQL → VPS Service APIs

**Query internal databases directly:**

```bash
# SSH to VPS
ssh root@100.87.191.9

# Check PostgreSQL for geolab-links data
psql -U postgres -d openseo -c \
  "SELECT page_url, link_count, domain_authority, last_updated 
   FROM pages WHERE domain = 'thegeolab.net' 
   ORDER BY last_updated DESC LIMIT 10;"

# Check Redis for queue state
redis-cli -h 100.87.191.9 -p 6379 INFO stats
```

**Confidence:** 100% (real-time)

## Confidence Scoring Model

```python
def calculate_confidence(sources: List[str], data_age_days: int, sample_size: int) -> float:
    """
    Composite confidence score (0-100%)
    sources: ['gsc', 'ga4', 'postgresql', 'wordpress', 'redis']
    data_age_days: days since last update
    sample_size: number of data points
    """
    # Source weight
    source_weight = {
        'gsc': 0.25,          # 25% — official Google, but 2-3 day lag
        'ga4': 0.25,          # 25% — official Google, but sampling
        'postgresql': 0.25,   # 25% — internal, real-time
        'wordpress': 0.15,    # 15% — secondary source
        'redis': 0.10         # 10% — ephemeral queue state
    }
    
    score = sum(source_weight.get(s, 0) for s in sources) * 100
    
    # Age penalty
    if data_age_days > 7:
        score *= 0.85  # 15% deduction for >7 days old
    elif data_age_days > 3:
        score *= 0.95  # 5% deduction for 3-7 days
    
    # Sample size penalty (GA4 sampling)
    if sample_size < 100:
        score *= 0.80  # 20% deduction for low sample
    
    return max(0, min(100, score))  # Clamp 0-100
```

## Example Multi-Source Report

**Report: Blog post traffic surge on Jan 15**

| Source | Data | Confidence | Notes |
|--------|------|------------|-------|
| GSC | 150 clicks (up 45% from 103) | 95% | Final data (3+ days old) |
| GA4 | 187 sessions (up 52% from 123) | 90% | Sampling margin ±15% |
| WordPress | Published Jan 12, title "AI SEO Audit", updated Jan 14 | 100% | Direct query |
| PostgreSQL | Link anchors: 42 backlinks, 3 new referring domains | 100% | Real-time |
| RankMath | SEO score 81/100, green lights on all pillars | 95% | Snapshot data |

**Conclusion:** High confidence (94% composite) that content quality + backlinks drove traffic spike. Recommend: Monitor position trending; consider expanding this topic.
