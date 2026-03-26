# Re-Test Cadence: Citation Decay & Freshness Triggers

Citation rate decay signals when content is losing retrieval momentum. This guide defines mandatory re-test triggers and establishes a baseline 6-month cadence for monitoring citation freshness and consistency.

## Why Citation Rate Decays

Citation rate measures how often a page is referenced by other content (internal links, external backlinks, or LLM retrieval citations). Decay occurs when:

1. **Content ages relative to topic**: Fresh content ranks higher. After 6 months, new competitors may have published fresher versions.
2. **Stats become outdated**: If a passage cites "2024 data," it becomes less credible in 2026. LLM retrieval penalizes stale statistics.
3. **Topic developments**: Algorithm changes, tool launches, or policy shifts invalidate older claims. Competitors update; you don't.
4. **Searcher behavior shifts**: Query patterns change seasonally or due to product launches. Content misses new intent.
5. **Consistency conflicts**: Multiple versions of your content contradict each other. Readers (and LLMs) lose confidence.

**Result:** Citation rate drops from 5+ per month to <1 per month. If not reversed, page becomes retrieval-invisible.

---

## 5 Mandatory Re-Test Triggers

Re-test a page immediately if ANY of these conditions occur:

### Trigger 1: Citation Age ≥6 Months Without Update
**Condition:** Page last updated >6 months ago AND citation count has not changed in last month.

**Why:** Evergreen topics decay naturally after 6 months. Time-sensitive topics (AI, tools, policy) decay after 3 months.

**Action:**
- Audit current stats (use content-freshness.md Step 1)
- Search for newer data (Step 2)
- Refresh ≥1 stat or add freshness signal (Steps 3-5)
- Re-test via quality gate (Step 7)

**Example:**
```
Page: "How to Optimize for AI Overviews"
Last updated: Sept 2024
Current date: March 2026
Topic type: Time-sensitive (AI)
Citation count this month: 0

ACTION: Retest required. Audit vs 3+ algorithm updates since pub.
```

---

### Trigger 2: Citation Count Declining >50% Quarter-Over-Quarter
**Condition:** Citations in Q2 < 50% of citations in Q1 (e.g., 10 citations → 4 citations).

**Why:** Rapid decline (not gradual decay) signals a specific event: competitor launched fresher content, topic lost relevance, or your content has an error.

**Action:**
- Identify the competitor or trigger event (use SEO Intelligence tool)
- Compare your content vs competitor's (structure, freshness, evidence)
- If competitor has newer data: update your stats + add source comparison
- If competitor has better structure: reorganize your content
- If topic changed: add "As of [date]" caveat or deprecate page

**Example:**
```
Page: "Best Schema Markup Tools for 2024"
Q1 2025: 8 citations
Q2 2025: 2 citations
Q3 2025: 1 citation
Decline rate: 75% per quarter

ACTION: Retest required. Competitor launched "Best Schema Tools 2025" with 12 citations.
Fix: Add "2026 tool comparison" subsection with current usage data.
```

---

### Trigger 3: Statistics >12 Months Old + No Source Attribution
**Condition:** Page contains stat with publication year ≤2024 (as of 2026) AND no citation/source link.

**Why:** Undated, unattributed stats fail LLM readability Q7 (Consistency & Specificity). LLMs hedges and rates page low-confidence.

**Action:**
- Extract all stats (grep -oE '[0-9]+%|[0-9]{4}' or manual review)
- For each stat: verify original source still exists
- If source retired: find replacement study or add caveat "As of [year]..."
- Add source attribution: "[Stat] according to [Company/Study], [Year]"
- Re-test via quality gate Q7

**Example:**
```
BEFORE:
"75% of marketers use internal linking."

AFTER:
"According to a 2025 HubSpot study, 73% of practitioners prioritize internal linking, down from 75% in 2023 due to increased focus on semantic relevance."
```

---

### Trigger 4: Topic Experienced Significant Development Since Publication
**Condition:** Topic has had ≥1 major event since page pub date: algorithm update, product launch, policy change, new study.

**Why:** Content becomes incomplete. LLMs detect missing context and assign lower authority to older versions.

**Action:**
- List all major events post-pub (e.g., Google Helpful Content Update March 2024, AI Overviews rollout May 2024)
- Audit page against each event: Does page mention it? Is coverage complete?
- If event not mentioned: add subsection or update existing section
- Add freshness signal: "Updated [date] to include [event]"
- Re-test via quality gate

**Example:**
```
Page: "How Google's Algorithm Works"
Published: June 2023
Major events post-pub:
  - March 2024: Helpful Content Update
  - May 2024: AI Overviews launch
  - Nov 2024: Ranking system update

Coverage audit:
  - Helpful Content: mentioned (old language)
  - AI Overviews: NOT mentioned
  - Ranking system: NOT mentioned

ACTION: Add section on AI Overviews + update Helpful Content language.
```

---

### Trigger 5: Traffic or Internal Links Declining >20% Month-Over-Month
**Condition:** Organic traffic OR internal link count down >20% from previous month.

**Why:** Decline signals retrieval loss. Either content is losing ranking (off-topic), or internal linking strategy changed (de-prioritized page).

**Action:**
- Check GSC: is page ranking but losing CTR? (fix: title/meta refresh)
- Check Analytics: is page losing traffic? (fix: topic relevance or freshness)
- Check internal links: are fewer pages linking to it? (fix: add internal link opportunities)
- Audit vs competitors: is your ranking dropping? (fix: evidence or structure)
- Re-test via quality gate

**Example:**
```
Page: "Internal Linking Best Practices"
Feb 2026 traffic: 450 visitors
March 2026 traffic: 325 visitors
Decline: 28%

GSC data:
- Ranking: still #2 for "internal linking"
- CTR: dropped from 8% to 5%

ACTION: Title/meta not resonating. Rewrite:
  FROM: "Internal Linking Best Practices"
  TO: "How Internal Links Pass Authority: 5 Proven Strategies"
```

---

## Baseline 6-Month Re-Test Cadence

Establish this schedule for all published content:

| Cadence | Action | Responsibility | Trigger |
|---------|--------|-----------------|---------|
| **Monthly** | Citation rate snapshot | Analytics/GSC tool | All pages |
| **Quarterly** | Decay trend analysis | Audit | Pages with declining rate >20% |
| **Semi-annual (6mo)** | Full re-test on 30% sample | Audit | All content (rotate 30% per cycle) |
| **Triggered** | Immediate re-test | Audit | Any of 5 mandatory triggers met |

### Implementation

**Month 1-2:** Audit 30% of content (alphabetical, oldest first)
- Run quality gate (geo-quality-gate.md)
- Check citation freshness (content-freshness.md)
- Check LLM readability (llm-readability.md)
- Mark pass/fail + assign to writer if fail

**Month 3-4:** Refresh all failed content from Month 1-2
- Writer revises sections per audit feedback
- Re-test before publishing

**Month 5-6:** Audit next 30% of content
- Repeat Month 1-2 process
- Track cumulative pass rate

**Between cycles:** Monitor for mandatory triggers
- Any trigger fire → stop cycle, retest that page immediately
- Update citation tracking log (see below)

---

## Citation Rate Tracking Log

Use this format to log citation rate data and distinguish decay from noise:

```markdown
## Citation Tracking Log

| Date | Page | Citations This Month | Citations Last Month | Change | Decay Type | Notes | Action |
|------|------|----------------------|----------------------|--------|-----------|-------|--------|
| 2026-03-26 | Internal Links Authority | 3 | 5 | -40% | Gradual | Consistent decline over 3mo | Monitor |
| 2026-03-26 | AI Overviews Guide | 0 | 2 | -100% | Steep | No citations in 4 weeks | RETEST TRIGGER 2 |
| 2026-03-26 | Schema Markup Tools | 1 | 1 | 0% | Stable | Flat for 2mo; topic seasonal | Monitor |
| 2026-03-26 | Backlink Analysis | 8 | 6 | +33% | Growing | Freshly updated Jan 2026 | Monitor |
| 2026-03-20 | SEO Metrics 2025 | 0 | 0 | 0% | None | Never cited; 2-month old | RETEST TRIGGER 1 |
```

**Column definitions:**
- **Date**: Log date
- **Page**: Page title or URL slug
- **Citations This Month**: Count from this period
- **Citations Last Month**: Count from previous period
- **Change**: % difference [(This - Last) / Last * 100]
- **Decay Type**: None, Stable, Gradual (5-20%), Steep (>50%), Growing
- **Notes**: Context (event, topic shift, seasonality)
- **Action**: Monitor, RETEST TRIGGER [1-5], or Retired

---

## How to Distinguish Decay from Noise

**Noise** = random fluctuation, not trend. Safe to ignore.
**Decay** = consistent downward trend. Requires action.

### Decision Matrix

| Data Points | Pattern | Verdict | Action |
|-------------|---------|---------|--------|
| 1 month | -30% | NOISE | Collect another month before deciding |
| 2 months | -30%, -20% | NOISE | Random variation, not trend |
| 2 months | -30%, -35% | DECAY | Trend confirmed, needs investigation |
| 3 months | -10%, -5%, -20% | NOISE | Fluctuating, not consistent |
| 3 months | -20%, -25%, -22% | DECAY | Consistent downward trend |
| 3 months | 0, 0, 0 | STALL | Flat line (no citations ever or recently stopped) |

**Interpretation:**
- **2 months steady decline** = Likely decay. Investigate immediately (Trigger 2).
- **1 month spike** = Likely noise. Wait for next month's data.
- **Flat for 3+ months after previous activity** = Stall (treated as decay). Investigate.

### Example Decision Process

```
Page: "Link Building 2025"

Week 1 (Jan): 6 citations
Week 2 (Jan): 5 citations (-17%)
Week 1 (Feb): 4 citations (-20%)
Week 2 (Feb): 3 citations (-25%)

Analysis:
- 3 consecutive periods down
- Trend: -20% to -25% per period (consistent)
- Verdict: DECAY (not noise)
- Action: TRIGGER 2 — investigate competitor, update content

Week 1 (Mar): 4 citations (+33%)
- Contradicts decay trend
- If next period also up: noise confirmed, no action needed
- If next period down: resume decay pattern, action taken
```

---

## Anti-Patterns: When NOT to Retest

### ✗ Retest on Every Citation Drop
**Mistake:** Citation drops from 5 to 4 one month. Immediately retest.

**Why it's wrong:** Single-month variations are noise. Creates thrash; wastes resources.

**Fix:** Wait for 2-month downward trend before retriggering. Log data but don't act on single month.

---

### ✗ Retest Without Knowing What Changed
**Mistake:** Citation rate down 30%. Refresh entire page without investigating cause.

**Why it's wrong:** Wastes effort. If competitor just published fresher content, refreshing one stat won't help. If topic shifted, structural rewrite needed, not just stat update.

**Fix:** Before retesting, identify the trigger (Trigger 1-5). Let that drive the fix scope.

---

### ✗ Ignore Stall (Zero Citations for 3+ Months)
**Mistake:** Page got 1 citation in Jan 2025, zero citations since. Don't retest because "no decay trend."

**Why it's wrong:** Zero activity = stall. Signals page lost authority or is off-topic. Not retesting allows irrelevance to compound.

**Fix:** Treat stall (flat zero for 3+ periods) same as decay. Investigate + retest.

---

## Citation Decay vs Natural Lifecycle

Not all citation decline is bad. Understand page lifecycle:

| Lifecycle Stage | Citation Rate | Action |
|---|---|---|
| **Launch (0-3mo)** | Rising | Monitor; don't retest unless Trigger fires |
| **Maturity (3-12mo)** | Stable or declining gradually | Retest if decline >20% OR stats >12mo |
| **Decline (12mo+)** | Falling >20%/quarter | Retest on Trigger 2 or deprecate |
| **Evergreen (12mo+)** | Flat/stable low (1-2/month) | Retest on 6-month cadence only |
| **Seasonal** | Peaks/troughs matching season | Track by season, not absolute numbers |

**Example: Seasonal Content**
```
Page: "Best Black Friday Tools 2024"
Sept: 0 citations
Oct: 12 citations (rising to event)
Nov: 25 citations (peak event month)
Dec: 8 citations (falling post-event)
Jan: 0 citations (off-season)

Verdict: SEASONAL, not decay.
Action: Retest in Sept 2025 for "2025" version, not quarterly.
```

---

## Retest Approval Flow

```
FOR EACH TRACKED PAGE:
  CHECK citation rate log for Trigger 1-5
  IF any trigger fires:
    ASSIGN page to writer
    NOTIFY: "Retest trigger [N] fired. Citation rate: [rate]. Last update: [date]."
    WRITER executes appropriate remediation (see content-freshness.md Steps 1-7)
    WRITER re-tests via quality gate
    IF pass:
      PUBLISH updated page
      LOG: "Retriggered [date]. Action taken: [description]."
    ELSE:
      RETURN to writer with quality gate feedback
  ELSE:
    CONTINUE monitoring
```

---

## Automated Citation Tracking Script

```bash
#!/bin/bash
# Track citation rate decay over time

LOG_FILE="citation_tracking.log"
TRACKING_DATE=$(date +%Y-%m-%d)

# Initialize log if not exists
if [ ! -f "$LOG_FILE" ]; then
  echo "Date,Page,This_Month,Last_Month,Change_Pct,Decay_Type,Notes" > "$LOG_FILE"
fi

# Example pages to track (customize per site)
PAGES=(
  "internal-links-authority"
  "ai-overviews-guide"
  "schema-markup-tools"
)

for PAGE in "${PAGES[@]}"; do
  # Get citation counts from GSC API or manual source
  # THIS EXAMPLE IS PSEUDOCODE — implement with actual API calls
  THIS_MONTH=$(get_citation_count "$PAGE" "month")
  LAST_MONTH=$(get_citation_count "$PAGE" "last_month")

  if [ "$LAST_MONTH" -eq 0 ]; then
    CHANGE_PCT=0
  else
    CHANGE_PCT=$(( (THIS_MONTH - LAST_MONTH) * 100 / LAST_MONTH ))
  fi

  # Determine decay type
  if [ "$CHANGE_PCT" -lt -50 ]; then
    DECAY_TYPE="Steep"
  elif [ "$CHANGE_PCT" -lt -20 ]; then
    DECAY_TYPE="Gradual"
  elif [ "$CHANGE_PCT" -eq 0 ]; then
    DECAY_TYPE="Stable"
  else
    DECAY_TYPE="Growing"
  fi

  # Log entry
  echo "$TRACKING_DATE,$PAGE,$THIS_MONTH,$LAST_MONTH,$CHANGE_PCT%,$DECAY_TYPE," >> "$LOG_FILE"

  # Alert on trigger
  if [ "$CHANGE_PCT" -lt -50 ]; then
    echo "ALERT: $PAGE citation rate dropped $CHANGE_PCT%. Trigger 2 fired."
  fi
done

echo "Tracking complete. Results logged to $LOG_FILE"
```

---

## Integration with Other Audit Tools

- **Trigger 1 or 3 fires:** Use content-freshness.md 7-Step Update Protocol
- **Any trigger fires:** Use geo-quality-gate.md 5-question test before republishing
- **New or updated stats:** Use llm-readability.md 7-question test on affected sections
- **Monthly monitoring:** Use compliance-checklist.md automated script to track schema health

---

## Quick Reference: Trigger Checklist

```markdown
## Monthly Re-Test Trigger Checklist

- [ ] Citation age ≥6 months + no update? → TRIGGER 1
- [ ] Citation count down >50% this quarter? → TRIGGER 2
- [ ] Stats >12 months old, no source? → TRIGGER 3
- [ ] Major topic event post-publication? → TRIGGER 4
- [ ] Traffic or internal links down >20%? → TRIGGER 5
- [ ] Any trigger fired? → Execute remediation + retest
- [ ] No triggers: Continue monitoring
```
