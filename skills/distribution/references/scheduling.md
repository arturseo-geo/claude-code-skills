# Upload Scheduling & Publishing Cadence

## Platform Rate Limits

| Platform | Upload Limit | Review Time | Time to Live |
|---|---|---|---|
| Gumroad | Unlimited | None (instant) | Minutes |
| Payhip | Unlimited | None (instant) | Minutes |
| D2D / Writing Life | **3 per 24 hours** | 24-48h for retailer distribution | 1-2 weeks to all retailers |
| Amazon KDP | Unlimited | Up to 72 hours | 24-72 hours |
| Google Play Books | Unlimited | Up to 48 hours | 24-48 hours |

---

## D2D 3/Day Limit — Planning Guide

The D2D limit is the primary constraint when publishing multiple titles. Plan uploads across multiple days.

### Example: 9-book library via D2D

| Day | Upload Slot 1 | Upload Slot 2 | Upload Slot 3 |
|---|---|---|---|
| Day 1 | Book 1 | Book 2 | Book 3 |
| Day 2 | Book 4 | Book 5 | Book 6 |
| Day 3 | Book 7 | Book 8 | Book 9 |

- Upload all 3 early in the day (morning) to reset the 24h window cleanly
- The 24-hour window is rolling, not calendar-day based
- If you upload Book 3 at 2:00 PM, you cannot upload Book 4 until 2:00 PM the next day
- **Violation = account termination** — do not create secondary accounts to bypass

### Tip: Upload D2D titles in priority order
1. Lead magnet / permafree title first (needs to be live everywhere for KDP price-match)
2. Flagship / highest-value title second
3. Supporting titles last

---

## Optimal Publishing Sequence

### For a single title (publish everywhere in one session):

```
Hour 0:   Gumroad — publish (instant, live immediately)
Hour 0:   Payhip — publish (instant, live immediately)
Hour 0:   KDP — submit (72h review queue starts)
Hour 1:   D2D — submit (if under 3/day limit)
Hour 72:  KDP likely live — verify
Week 1-2: D2D retailers (Apple, Kobo, B&N) go live
Week 2:   Update Gumroad description with cross-links
Week 2:   If permafree: report lower price to KDP
```

### For a library launch (multiple titles):

```
Day 1:
  - All titles → Gumroad (no limit)
  - All titles → Payhip (no limit)
  - All titles → KDP (no limit, 72h review each)
  - 3 titles → D2D (daily limit)

Day 2:
  - 3 more titles → D2D

Day 3:
  - 3 more titles → D2D (9-book library complete)

Day 4-5:
  - KDP titles should be live — verify each
  - Begin cross-linking on Gumroad

Week 2-3:
  - D2D retailers go live — check distribution status dashboard
  - Update cross-links as each platform goes live
  - Begin permafree price-match requests to KDP
```

---

## Post-Publish Monitoring Schedule

| Timeframe | Action |
|---|---|
| Day 0 | Verify Gumroad + Payhip listings are correct |
| Day 1-3 | Check KDP review status daily |
| Day 3-5 | KDP should be live — verify listing, categories, keywords |
| Week 1 | Check D2D distribution dashboard — some retailers are fast (Kobo), others slow (Apple) |
| Week 2 | Most D2D retailers should be live — verify links work |
| Week 2 | If permafree: submit price-match request to KDP |
| Week 3 | KDP may or may not have price-matched — re-report if needed |
| Month 1 | Full audit: all platforms live, descriptions correct, cross-links working |

---

## Scheduling Around Updates

When updating an existing title (new edition, description change, cover update):

### Metadata-only changes (description, keywords, categories):
- **Gumroad/Payhip**: Instant, no review
- **KDP**: 24-48h review after save
- **D2D**: Pushes to retailers, 1-2 weeks to propagate

### File updates (new EPUB, new cover):
- **Gumroad/Payhip**: Instant replace
- **KDP**: Full review cycle (up to 72h), book may go temporarily unavailable
- **D2D**: New file pushed to retailers, 1-2 weeks, does NOT count against 3/day limit (updates are free)

### Best practice for coordinated updates:
1. Update direct platforms first (Gumroad, Payhip) — instant
2. Update KDP — triggers review
3. Update D2D last — slowest propagation
4. Allow 2-3 weeks for all platforms to reflect changes

---

## Launch Day Timing Tips

- **Weekday launches** (Tue-Thu) get faster KDP review than weekends
- **Avoid major Amazon events** (Prime Day, Black Friday) — review queues are longer
- **D2D uploads**: Morning uploads give a clean 24h reset for next day
- **Gumroad Discover**: Products appear in Discover after first paid sale + risk review — plan a small paid launch to trigger eligibility
- **Cross-link updates**: Wait until ALL platforms are live before adding cross-links to Gumroad descriptions — broken links hurt credibility
