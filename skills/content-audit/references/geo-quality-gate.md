# GEO Quality Gate (5-Question Per-Section Test)

Per-section quality checkpoint. Each section (intro, body paragraphs, subsections, closing) must pass all 5 questions. Failure on any question triggers targeted rejection.

## The 5 Questions

### 1. Direct Declarative Opener
**What to check:** Does the first sentence state a single, testable fact or claim without hedging?

**What failure looks like:**
- "In today's digital landscape, many businesses struggle with..." (filler + multi-part claim)
- "It could be argued that..." (hedging)
- "This article explores the question of..." (meta-reference instead of claim)
- "There are several ways to..." (vague plural)

**Exact rejection message:**
```
REJECT: Opener lacks directness
Reason: Section opens with [quote]. This is hedging/filler, not a claim.
Fix: Replace with a single declarative sentence: "CLAIM: [X] because [Y]."
Example: "Google's AI Overview uses 3-4 sentence summaries, not full passage extraction."
```

**Verification bash:**
```bash
# Check for hedging patterns in first sentence
FIRST_SENT=$(curl -s "$URL" | sed -n '/<p>/,/<\/p>/p' | head -1)
echo "$FIRST_SENT" | grep -iE '(in today|could be|seems|appears|many|several|there are|argues that|tends to)' && echo "FAIL: Hedging detected" || echo "PASS"
```

---

### 2. Pronoun Resolution
**What to check:** Every pronoun (it, they, this, that) has a clear antecedent within 2 sentences back.

**What failure looks like:**
- "The algorithm was trained on X. It then applied Y. This resulted in Z." (What is "it"? Algorithm? Training? Y?)
- "Ranking factors include links, content, and RankBrain. They work together." (What is "they"? All three? Just links?)
- "Google updated Helpful Content. That change affected..." (What is "that"? The update? Its timing?)

**Exact rejection message:**
```
REJECT: Pronoun resolution failure in section [SECTION]
Reason: Pronoun "[PRONOUN]" at line [N] has unclear antecedent.
Context: "[PREVIOUS_SENT]"
Fix: Replace "[PRONOUN]" with explicit noun: "[NOUN]".
Current: "...They resulted in..."
Revised: "...The three factors resulted in..."
```

**Verification bash:**
```bash
# Extract section and flag pronouns without clear antecedent
SECTION=$(curl -s "$URL" | sed -n '/<h2>.*<\/h2>/,/<h2>/p')
echo "$SECTION" | grep -n -E '\b(it|they|this|that)\b' | while read LINE; do
  LINENUM=$(echo "$LINE" | cut -d: -f1)
  PREV_SENT=$(echo "$SECTION" | sed -n "$((LINENUM-2)),$((LINENUM-1))p")
  PRONOUN=$(echo "$LINE" | grep -o 'it\|they\|this\|that' | head -1)
  echo "Line $LINENUM: pronoun '$PRONOUN' - check antecedent in: $PREV_SENT"
done
```

---

### 3. Single Topic Isolation
**What to check:** Does the section cover one discrete topic, not a hybrid or list?

**What failure looks like:**
- Section on "Citation Authority & Backlink Placement" (two unrelated concepts)
- "How to fix broken links, manage redirects, and set up monitoring" (three separate tasks)
- "Understanding AI Overviews: Google Search, Bard, and competitor systems" (five topics)
- A subsection that mixes "Why this matters" with "How to implement it" (conflates motivation with action)

**Exact rejection message:**
```
REJECT: Section lacks topic isolation
Reason: "[SECTION_TITLE]" covers [N] distinct topics: [TOPIC1], [TOPIC2], [TOPIC3]...
Rule: One section = one discrete topic. Use subsections for related concepts.
Fix: Rename to focus on ONE topic, or split into subsections:
  - Subsection 1: Citation Authority
  - Subsection 2: Backlink Placement
```

**Verification bash:**
```bash
# Extract section heading and count distinct clauses/topics
SECTION=$(curl -s "$URL" | sed -n '/<h2>.*<\/h2>/p' | head -1 | grep -o '[^:]*</h2>')
CLAUSE_COUNT=$(echo "$SECTION" | grep -o ' and \| or \|, ' | wc -l)
[ "$CLAUSE_COUNT" -gt 1 ] && echo "WARNING: Section title has $(($CLAUSE_COUNT+1)) topics" || echo "PASS"
```

---

### 4. Verifiable Data Point
**What to check:** Does the section contain ≥1 specific, measurable claim (stat, quote, timestamp, example)?

**What failure looks like:**
- Section with only abstract explanations ("This is important because...") and zero numbers/quotes
- "Content helps with SEO" (true but unverifiable without supporting data)
- "Our clients saw better results" (no magnitude, no baseline, no citation)
- All claims are hedged ("may help", "could improve")

**Exact rejection message:**
```
REJECT: No verifiable data in section
Reason: Section "[TITLE]" contains only abstract claims with zero supporting data.
Required: ≥1 verifiable data point: stat (%), quote, study, example, or timestamp.
Example claims that PASS:
- "Google's AI Overview limits to 3-4 sentences."
- "According to Moz, domain authority correlates 0.36 with rankings."
- "A/B test on 500 pages showed 12% CTR improvement."
Current section: [QUOTE]
Fix: Add ≥1 data point with source attribution.
```

**Verification bash:**
```bash
# Count verifiable elements: percentages, years, quotes, studies
SECTION=$(curl -s "$URL" | sed -n '/<h2>.*<\/h2>/,/<h2>/p' | head -1)
DATA_POINTS=$(echo "$SECTION" | grep -oE '(\d+%|[0-9]{4}|"[^"]*"|study|research|found)' | wc -l)
[ "$DATA_POINTS" -ge 1 ] && echo "PASS: $DATA_POINTS data points found" || echo "FAIL: No data points"
```

---

### 5. Citation-Ready Opening Sentence
**What to check:** Does the opening sentence of the section contain one claim that could stand alone as a thesis with a source cited?

**What failure looks like:**
- "There are many ways to improve SEO." (vague, non-citeable)
- "Let me explain why this matters." (meta-narrative, not a claim)
- "In this section, we'll cover three strategies." (structural, not substantive)
- "The algorithm works like this:" (framing for a list, not a claim)

**Citation-ready examples (PASS):**
- "Google's AI Overviews prioritize concise answers over long-form content." (single claim, specific)
- "Schema markup increases CTR by an average of 30% according to Semrush data." (claim + source ready)

**Exact rejection message:**
```
REJECT: Opening sentence not citation-ready
Reason: First sentence of "[SECTION]" is not a standalone, citable claim.
Current: "[FIRST_SENT]"
Issue: [ISSUE TYPE - e.g., vague, meta-narrative, structural]
Fix: Replace with single declarative claim that could appear in a bibliography.
Model: "[TOPIC] [RELATIONSHIP] [OBJECT] [because/since/which] [basis]."
Example: "Internal links increase page authority when they use contextual anchor text."
```

**Verification bash:**
```bash
# Extract opening sentence and check for citation-ready structure
SECTION=$(curl -s "$URL" | sed -n '/<h2>/,/<\/h2>/p' | head -1)
FIRST_SENT=$(echo "$SECTION" | sed -n 's/.*<\/h2>\s*<p>\([^<]*\).*/\1/p')
echo "$FIRST_SENT" | grep -iE '(there are|let me|in this section|the algorithm|ways to|strategies|methods)' && echo "FAIL: Not citation-ready" || echo "PASS"
```

---

## Approval Flow

```
FOR EACH SECTION:
  RUN all 5 questions
  IF all 5 pass:
    Mark section as APPROVED
  ELSE:
    COLLECT all failing questions
    RETURN rejection message with:
      - Which question(s) failed
      - Exact quote from text showing the failure
      - Model/example of corrected text
      - Author must revise section and resubmit
```

## Reporting Table Format

Use this table to log audit results:

| Section | Q1 Direct | Q2 Pronoun | Q3 Topic | Q4 Data | Q5 Citation | Overall | Notes |
|---------|-----------|-----------|---------|---------|------------|---------|-------|
| Intro (H2) | ✓ | ✓ | ✓ | ✗ | ✓ | FAIL | Add 1 stat to Q4 |
| Body Para 1 | ✓ | ✓ | ✓ | ✓ | ✓ | PASS | - |
| Subsection 2.1 | ✗ | ✓ | ✓ | ✓ | ✗ | FAIL | Remove hedging in Q1; rewrite opening |
| Subsection 2.2 | ✓ | ✗ | ✓ | ✓ | ✓ | FAIL | Clarify pronoun "it" in para 2 of Q2 |
| Closing | ✓ | ✓ | ✓ | ✓ | ✓ | PASS | - |
| **TOTAL** | 4/5 | 4/5 | 5/5 | 4/5 | 4/5 | **17/25 FAIL** | Revise 4 sections |

**Pass threshold:** 5/5 questions across ≥80% of sections.
**Fail = any section scores <3/5 on questions.**
