# Report Templates

## Weekly Traffic Report (Standard)

```
WEEKLY TRAFFIC REPORT
Week of: Jan 15-21, 2026
Data sources: GSC, GA4, WordPress
Confidence: 94%

EXECUTIVE SUMMARY
Organic traffic grew 12% week-over-week to 2,847 sessions. Driven by 3 new
publications and continued ranking improvement on "AI SEO" cluster (avg +2.3 positions).
No anomalies detected. Conversion rate stable at 3.2%.

KEY METRICS
┌─────────────────┬──────┬──────────┬─────────┐
│ Metric          │ This │ vs Week  │ vs Year │
├─────────────────┼──────┼──────────┼─────────┤
│ Sessions        │ 2847 │ +12% ↑   │ +34% ↑  │
│ Users           │ 2103 │ +8% ↑    │ +28% ↑  │
│ Conversions     │  91  │ +11% ↑   │ +40% ↑  │
│ GSC Impressions │8294  │ +4% ↑    │ +18% ↑  │
│ GSC Avg CTR     │ 3.4% │ -0.1%    │ +0.2%   │
│ Avg Position    │ 7.2  │ -0.8 ↑   │ -2.1 ↑  │
└─────────────────┴──────┴──────────┴─────────┘

TOP PERFORMING CONTENT
1. "AI SEO Audit Tools" — 847 sessions, 156 conversions (18.4% CVR)
   Position: 4.2 (was 6.8) | Impressions: 1,304 | Backlinks: +5 new

2. "Content Gap Analysis" — 523 sessions, 89 conversions (17.0% CVR)
   Position: 5.1 | Impressions: 892 | Organic growth (no promo)

3. "Backlink Intelligence Guide" — 312 sessions, 41 conversions (13.1% CVR)
   Position: 8.3 | Impressions: 678 | Ranking stable

TRAFFIC SOURCES
├─ Organic Search:  2,103 sessions (73.9%)
├─ Referral:         421 sessions (14.8%) [LinkedIn, Reddit]
├─ Direct:          234 sessions (8.2%)
└─ Other:            89 sessions (3.1%)

CONVERSION FUNNEL
Landing: 2,847 users
→ Scroll 50%: 2,309 (81%)
→ Click CTA: 1,203 (42%)
→ View offer: 412 (14%)
→ Conversion: 91 (3.2%)

ROOT CAUSE ANALYSIS
Up 12% primarily due to:
  • 3 new "AI" articles published (driving 486 new sessions)
  • Rank improvement on existing cluster (avg +2.3 positions = +8% impressions)
  • Continued LinkedIn sharing momentum (+4 sponsored posts)
  • No technical issues or platform changes detected

RECOMMENDATIONS
1. Expand "AI" cluster — 3 follow-up articles ready (est. +20% traffic)
2. Optimize "Backlink Intelligence" for position 5 (currently 8.3)
   → Add expert comparison, update backlinks section
3. Test LinkedIn thread format for "Content Gap" article
   → Current LinkedIn share: 31 clicks → explore weekly threads
4. Monitor "AI SEO Audit" rank position — pushing for #3 with 1-2 backlinks

DATA QUALITY NOTES
• GSC: Final data (3+ days old), 100% confidence
• GA4: Standard reporting (24h lag), sampling inactive
• Conversion tracking: Custom event verified
• External traffic: Reddit r/SEO (organic 150 clicks)
```

## Monthly Deep-Dive Report

```
MONTHLY ANALYTICS DEEP DIVE
Period: January 1-31, 2026
Data sources: GSC, GA4, PostgreSQL, WordPress, RankMath
Confidence: 96%

[Similar structure to weekly, but with:]
- 30-day MoM and YoY comparisons
- Trend analysis (linear regression slope)
- Seasonality notes
- New content impact analysis
- Backlink velocity
- Rank progression on target keywords
- Competitor benchmarking (if available)
- Cohort analysis (new users vs returning)
- Device/location breakdown
- Attribution modeling (multi-touch if available)
```

## Traffic Drop Investigation Report

```
TRAFFIC DROP INVESTIGATION
Issue: Organic sessions dropped 28% on Jan 19 (vs daily baseline)
Report date: Jan 20, 2026
Investigator: Analytics Skill

CRISIS SUMMARY
Date detected: Jan 19, 2026
Severity: HIGH (28% drop, 847 sessions lost)
Duration: 1 day (recovered to 94% baseline by Jan 20)
Status: RESOLVED

TIMELINE
├─ Jan 18 baseline: 3,039 sessions (normal)
├─ Jan 19 drop: 2,192 sessions (-728, -28%)
├─ Investigation started: Jan 19 14:00 UTC
├─ Root cause identified: Jan 19 16:30 UTC (Google Core Update)
├─ Recovery began: Jan 20 09:00 UTC
└─ Full recovery: Jan 20 23:00 UTC

ROOT CAUSE ANALYSIS
Primary cause: Google Search Algorithm Update (likely Core Update)
  • Evidence 1: Search Status Dashboard flagged update on Jan 19 00:00 UTC
  • Evidence 2: ALL keywords dropped together (not page-specific)
  • Evidence 3: Impressions stable (-2%), but CTR collapsed (-82%)
    → Indicates algorithm rewrote SERPs, not ranking drop
  • Evidence 4: Competitor sites also fluctuated (no site-specific issue)
  • Confidence: 98% (matches historical Core Update pattern)

Detailed metrics:
├─ Impressions: 8,234 (was 8,427, -2%) ← Stable
├─ Clicks: 282 (was 934, -70%) ← Dropped
├─ CTR: 3.4% (was 11.1%, -69%) ← MAJOR DROP
├─ Avg Position: 8.2 (was 7.1, -1.1 positions)
└─ Traffic: 2,192 sessions (was 3,039, -28%)

Pages impacted:
  • All pages >1,000 monthly impressions affected
  • Largest drop: "AI SEO Audit" (-34% traffic)
  • Smallest drop: "Niche page" (-12% traffic)
  → Confirms broad algorithm impact, not content quality

SECONDARY FACTORS RULED OUT
✓ No site indexing issues (GSC coverage 100%)
✓ No Core Web Vitals degradation (CLS=0.05, LCP=1.8s, FID=45ms)
✓ No crawl errors (0 errors in GSC)
✓ No penalties (no security warnings, no manual actions)
✓ No DDOS or downtime (uptime 99.9%)
✓ No WordPress plugin conflict (all plugins verified)

RECOMMENDATIONS
1. WAIT for algorithm to stabilize (typically 2-4 weeks post-update)
   → Recommended: Monitor rank trends daily, don't make reactionary changes
2. Audit content against Google's Core Update checklist
   → Focus: E-E-A-T, content depth, user intent alignment
3. Accelerate high-priority backlink outreach
   → Target: 5 new backlinks on top 10 keywords within 30 days
4. Publish 2 new articles to signal freshness
   → Topics: Trending queries still showing opportunity

RECOVERY FORECAST
Historically, sites recover 60-90% within 2-4 weeks of Core Update.
Our recovery profile:
  • Jan 20: 94% recovered
  • Jan 21-25: Expect 97% recovery (stabilization)
  • Feb 1: Expect 100%+ recovery (new content + backlinks)
  • Feb 15: Potential +15% improvement (algorithm stabilization)

Monitor daily and report next week.
```

## Page-Level Performance Report

```
PAGE-LEVEL PERFORMANCE BREAKDOWN
Dimension: Individual blog posts
Period: Jan 1-31, 2026
Data sources: GSC, GA4, WordPress, RankMath

[Table format]
Page Title | Sessions | Conv | Conv% | Rank | Pos | CTR | Impressions | Backlinks
"AI Audit" | 847 | 156 | 18.4% | 4 | 4.2 | 11.2% | 1,304 | 23
"Gap Tool" | 523 | 89 | 17.0% | 12 | 5.1 | 9.8% | 892 | 8
"Backlink" | 312 | 41 | 13.1% | 34 | 8.3 | 7.2% | 678 | 5

[Analysis includes]
- Correlation between rank, backlinks, and traffic
- Content update recommendations
- Internal link opportunities
- Topic expansion gaps
```

## Audit Report (RankMath/SEO Compliance)

```
SEO AUDIT REPORT
Date: Jan 31, 2026
Tool: RankMath + custom audits

OVERALL SCORE: 87/100 (Excellent)

GREEN FLAGS (✓)
✓ All pages have unique meta descriptions
✓ Mobile usability: No errors
✓ Core Web Vitals: All green
✓ SSL certificate: Valid & current
✓ XML sitemap: Submitted & indexed
✓ Robots.txt: Correct
✓ Schema markup: 98% pages

ISSUES (⚠ yellow, 🔴 red)
⚠ 12 pages missing H1 tag (recommended: 1 per page)
⚠ 3 pages with <250 words (content depth)
🔴 1 page with broken internal link (fix required)
🔴 2 images missing alt text (accessibility)

RECOMMENDATIONS (Priority order)
1. Add H1 tags to 12 pages (30 minutes)
2. Expand 3 short pages (+500 words each = 3 hours)
3. Fix 1 broken link (5 minutes)
4. Add alt text to 2 images (5 minutes)

Est. impact: +3-5 points in audit score, +5-8% organic traffic
```
