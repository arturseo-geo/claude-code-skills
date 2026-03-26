# Content Freshness as a GEO Signal

Freshness determines consistency in AI search results. Stale data creates retrieval conflicts where different retrieval methods (semantic search, dense passage retrieval, hybrid) agree on irrelevant pages.

## Why Freshness Affects Retrieval

**Consistency conflict scenario:**
- Page A: "Google's algorithm update in 2022 ranked links as 2nd most important factor" (published 2022, now 2026)
- Page B: "Links remain the #1 ranking factor" (published 2020, never updated)
- Page C: "Links are increasingly marginal compared to semantic relevance" (published 2025, freshly updated)

When an LLM retrieves results for "how important are links in ranking," all three pages rank (semantic match on "links" + "ranking"). But the **consistency score** drops because:
- Pages contradict on fact (2nd vs 1st vs marginal)
- Dates are misaligned (2022, 2020, 2025)
- Reader cannot determine which is authoritative

**Result:** LLM hedges ("Links remain important, though their role is evolving...") instead of making a clear claim.

---

## 4-Priority Update Framework

Use this to triage content for re-testing and refresh:

### P1: High Citation Count + Old Stats
**Trigger:** Page cited ≥5 times in past 3 months + has undated stats >2 years old

**Why risky:** High citation = high visibility in retrieval. Old stats = credibility risk if contradicted.

**Action:** Full stat audit + refresh. Verify every %/number/year. Add citations.

**Example:**
- Page "SEO Citation Authority" cited in 8 recent blog posts
- Contains: "75% of businesses use backlinks" (from 2023 report, now 2026)
- **Action:** Re-run study or source 2025+ data. If unavailable, add caveat: "As of 2023, [stat]. [Current status unknown]."

### P2: Declining Citation Rate + Time-Sensitive Topic
**Trigger:** Page cited ≥3x last quarter, <1x this quarter + topic is time-sensitive (AI, tools, algorithms, policies)

**Why risky:** Declining citations suggest content is losing relevance. Time-sensitive topics compound this.

**Action:** Investigate + update. Check if newer competitor content exists.

**Example:**
- Page "How to Optimize for AI Overviews" (published Oct 2024)
- Cited 5x in Q1 2025, 1x in Q2 2025
- Google released 3 algorithm updates since pub date
- **Action:** Audit changes since pub. Add "Latest update: [date]". Compare vs competitor versions.

### P3: Time-Sensitive Topic + No Stats
**Trigger:** Topic is AI/tools/policy + no publication or update date + zero stats

**Why risky:** Readers can't assess age. No data = no credibility.

**Action:** Add publication date + update date. Add ≥1 stat or recent example.

**Example:**
- Page "Best Schema Markup Tools" (no publication date, never updated)
- Lists 5 tools, zero stats on usage/market share
- **Action:** Add "Published [date]. Last updated [date]." Add "Used by X% of sites" or "Y million downloads" for each tool.

### P4: Ranked + Never Cited
**Trigger:** Page ranks for ≥1 keyword + zero citations in past 6 months

**Why risky:** Content exists but isn't valued by peers. May indicate gap with community consensus.

**Action:** Audit vs competitors. Consider deprecation if outranked.

**Example:**
- Page "Stop Word Myth in SEO" (ranks #3 for "stop words SEO")
- Zero recent citations. Competitor pages have 3+ citations each.
- **Action:** Compare positioning. If competitor page is fresher, consider: (a) refresh + improve evidence, or (b) add internal link to competitor's page, or (c) sunset page.

---

## 7-Step Update Protocol

Follow this when refreshing stale content:

### Step 1: Audit Current Stats
Pull all statistics, dates, percentages, study citations from the page.

```bash
# Extract all potential stats
curl -s https://URL | grep -oE '[0-9]+%|[0-9]{4}|"[^"]*"' > /tmp/stats.txt
```

**Document:**
- Stat text
- Source cited (if any)
- Publication year of stat
- Whether stat is still valid (check original source)

### Step 2: Search for Newer Data
For each stat, search for updated versions:

```bash
# Example: "links ranking factor" + current year
google_search "links ranking factor 2025 2026 study" | head -5
```

**Decision matrix:**
| Source Status | Action |
|---|---|
| Original source updated | Use new stat + link to update |
| Original source retired | Search for replacement study |
| No newer data exists | Keep old stat + add caveat: "As of [year]..." |
| Contradicted by new research | Add both: "[Stat], though recent evidence suggests [new finding]" |

### Step 3: Set Publication/Update Dates
Add or confirm two dates on the page:

**Published:** Original publication date (unchanging)
**Updated:** Most recent update date (changes with each refresh)

**Visibility requirement:** Both dates must appear near the opening (within first 200 words or in byline/metadata).

Example format:
```html
<div class="article-meta">
  <span>Published: October 15, 2022</span>
  <span>Last updated: March 26, 2026</span>
</div>
```

### Step 4: Update Stat with Source
For each stat being refreshed:

```markdown
BEFORE:
"Links remain the most important ranking factor at 75%."

AFTER:
"According to a 2025 SEO study by [Company], links influence 62% of ranking changes, down from 75% in 2023 due to increased weight on semantic relevance."
```

**Pattern:** [Claim] [Year] [Source] [Context/Change if applicable]

### Step 5: Add Freshness Signal
Insert one of these near the opening:

- ✓ "This post was refreshed in [month] [year] to include [X new finding/data/tool]."
- ✓ "Latest data as of [month][year]."
- ✓ "Updated [date] following [specific event: algorithm change, new study, etc.]."

### Step 6: Add Version History Block
At the end of the post, add a "Version History" section:

```markdown
## Version History

- **v3** (March 26, 2026): Updated stats on link importance (2025 data), added section on AI Overview impacts
- **v2** (June 12, 2024): Expanded section on schema markup, new tool comparison
- **v1** (October 15, 2022): Original publication
```

### Step 7: Re-test Citation Readiness
Run the quality gate on updated sections. Priority: opening sentence + any stat-heavy sections.

**Retest checklist:**
- [ ] Opening sentence is a direct claim (not hedged)
- [ ] All stats have sources cited
- [ ] Dates are present and visible
- [ ] No pronouns with unclear antecedents
- [ ] Each stat supports a single claim (topic isolation)

---

## What NOT to Do: Anti-Patterns

### ✗ Bulk Date Refresh
**Mistake:** Updating the "Last Updated" date without changing content.

**Why it's wrong:** Creates consistency conflict. Readers see "Updated 2026" but data still says "2023 study." Signals falsified freshness to AI readers.

**Fix:** Only update date if you've actually updated the content.

---

### ✗ Add Date Without Context
**Mistake:** Adding "Last updated March 2026" but not indicating what changed.

**Why it's wrong:** Readers don't know if the entire page is fresh or just a minor typo fix. Creates ambiguity.

**Fix:** Add a freshness signal: "Updated March 2026 to include 2025 ranking factor study."

---

### ✗ Replace Old Stats Without Caveats
**Mistake:** Old stat says "75%", new stat says "62%". Replace 75% with 62% without explanation.

**Why it's wrong:** Readers miss the narrative. Change appears as error or contradiction. Citation-ready claims require explaining the shift.

**Fix:** "Originally 75% (2023), now 62% (2025), reflecting increased weight on semantic relevance."

---

### ✗ Update Only One Mention
**Mistake:** Page mentions "links are #1 factor" in intro (updated) but also in conclusion (old).

**Why it's wrong:** Consistency conflict. Reader sees contradictory claims within same page.

**Fix:** Search entire page for all versions of the same claim. Update all.

---

### ✗ Add Date But Not Visible
**Mistake:** Add `post_modified` date to WordPress but don't display it.

**Why it's wrong:** LLM retrieval doesn't see it. Readers don't see it. Freshness signal is wasted.

**Fix:** Display near opening and in byline.

---

## Statistical Currency Definition

A stat is **current** if:

1. **Age + context permit:** Published within 3 years for evergreen topics, within 1 year for time-sensitive (AI, tools, policy)
2. **Original source validates:** The study/report still exists and original publisher stands by it
3. **No contradicting consensus:** Newer research doesn't directly contradict it (allowed: refinement, not contradiction)
4. **Attribution present:** Source and date are cited

A stat is **stale** if:

- Published >3 years ago + topic is evergreen
- Published >1 year ago + topic is time-sensitive
- Original source is no longer available
- Newer data directly contradicts it
- No source attribution

---

## Freshness Signal Impact on Consistency Score

| Page State | Consistency Issue | Retrieval Impact | Action |
|---|---|---|---|
| Multiple versions with different stats + no dates | 3 pages contradict each other | High (LLM hedges, low confidence) | Refresh all 3 + add dates |
| One page fresh + others stale + dates visible | Reader can rank by date; LLM chooses newest | Medium (LLM picks fresh) | Archive or merge old pages |
| All pages dated + consistent stats | No conflict; same claim across sources | Low (LLM confident) | Maintain; refresh on P1/P2 triggers |
| Page undated + high citation count | Can't assess age; conflict hidden | High (LLM confused) | Add dates immediately |
