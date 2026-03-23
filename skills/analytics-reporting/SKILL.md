# Analytics & Reporting Skill

## When This Skill Activates

This skill loads automatically when you mention:
- analytics, reporting, traffic metrics, performance reports
- weekly reports, monthly reports, dashboards, data analysis
- traffic drops, traffic spikes, content performance, page performance
- user behavior, conversion analysis, cross-source data correlation
- why did traffic drop, what content is performing, performance anomaly detection
- trend analysis, root cause analysis, traffic investigation
- combined reporting, GSC + GA4 analysis, search traffic, organic traffic trends
- bounce rate analysis, engagement metrics, landing page analysis
- session analysis, revenue correlation, goal conversion
- heat maps, user journeys, content audit performance, traffic forecasting
- any multi-source data analysis and reporting task

## Data Sources

**Real-time APIs:**
- Google Search Console API (impressions, clicks, CTR, position, queries)
- Google Analytics 4 API (property 528383456) — sessions, users, conversions, events
- PostgreSQL OpenSEO DB (100.87.191.9:5432) — internal metrics, backlinks, keywords
- Redis cache (100.87.191.9:6379) — geolab-links queue state, job timing

**WordPress & VPS:**
- WordPress REST API (thegeolab.net) — post metadata, publish dates, author
- VPS Service APIs (100.87.191.9):
  - seo-intelligence (:3001) — rank tracking data
  - geolab-links (:3002) — link profile, authority scores
  - geolab-backlinks (:4000) — backlink snapshots, competitor data
  - geolab-keywords (:4001) — keyword clustering, SERP features
  - geolab-writer (:4002) — published content, performance tags

## Core Responsibilities

1. **Multi-Source Correlation** — Join data from GSC, GA4, PostgreSQL, WordPress, Redis using lag-aware joins and confidence scoring
2. **Anomaly Detection** — Z-score, moving averages, seasonality adjustment, algorithm change detection, composite anomaly scoring
3. **Report Generation** — Weekly summaries, monthly deep-dives, investigation reports, audit snapshots, page-level breakdowns
4. **Actionable Insights** — Root cause analysis, content recommendations, traffic recovery strategies, opportunity detection
5. **Data Validation** — Check data completeness, lag detection, source freshness, accuracy within confidence bounds

## Key Patterns

### Data Joining Strategy
- **GSC → GA4**: Join on page URL, account for 1-3 day GSC reporting lag
- **GA4 → PostgreSQL**: Join on post_id via WordPress URL parsing
- **WordPress → VPS Services**: Query via REST API for metadata enrichment
- **Confidence Scoring**: Apply weights based on data freshness and source reliability

### Anomaly Detection Method
1. Calculate baseline (30-90 day rolling average)
2. Adjust for seasonality (day-of-week, holidays, known events)
3. Compute z-score and isolation forest scores
4. Combine into composite anomaly score
5. Flag if score > threshold + explanation

### Report Template Structure
- Executive summary (1 paragraph)
- Key metrics table (YoY/MoM % change)
- Anomaly findings (if any)
- Content performance ranking
- Traffic sources breakdown
- Conversion funnel analysis
- Root cause diagnosis (for drops)
- Actionable recommendations (3-5 items)
- Data quality notes + sources cited

## Output Standards

**Always include:**
- Data date range ("Jan 1-31, 2026")
- All sources used (GSC API, GA4, PostgreSQL, etc.)
- Confidence levels (95%, 85%, unvalidated)
- Actionable recommendations (not just observations)
- Any caveats or data gaps

**Never:**
- Fabricate data or fill gaps with estimates
- Claim confidence >95% without 3+ sources
- Recommend without understanding intent
- Ignore seasonal patterns without mentioning them
- Skip data freshness checks

## See Also

- **anomaly-detection.md** — Z-score formulas, moving averages, seasonality math
- **correlation.md** — Multi-source joining strategy, lag-aware logic, confidence scoring
- **data-sources.md** — API endpoints, query patterns, connection strings
- **report-templates.md** — Weekly, monthly, investigation, audit, page-level templates
