# LLM Readability Testing

LLM readability measures passage-level processability by AI systems. This differs fundamentally from human readability (legibility, flow) and impacts whether retrievers can extract, summarize, or cite a passage correctly.

## Definition: Passage-Level AI Processability

A passage is LLM-readable if:

1. **Semantic coherence**: Passage can be understood in isolation without external context
2. **Information density**: Claim-to-filler ratio supports extraction (not padded with transitions)
3. **Explicit relationships**: Entities and claims are connected via direct language, not inference
4. **Minimal ambiguity**: Pronouns, references, and scope are unambiguous
5. **Verifiable structure**: Claims are bound to evidence within or immediately adjacent to the passage

**Result:** LLM can extract a fact, cite a source, or generate a summary that reflects the passage accurately.

**Non-result:** LLM hedges, invents context, or misrepresents the passage because it cannot anchor understanding.

---

## 7 Components Mapped to GEO Stack Layers

| Component | Definition | Layer(s) | Failure Impact |
|---|---|---|---|
| **Natural Language Quality** | Sentences are grammatically correct, avoid jargon overload, use active voice | L2 (Language Semantics) | LLM misinterprets claim or invents interpretation |
| **Structuring (Hierarchy)** | Headings, subheadings, paragraphs create logical nesting; information flows depth-first | L1 (Syntax) + L2 (Semantics) | LLM misaligns scope; treats sub-claim as main claim |
| **Chunk Relevance** | Sentence/passage contains only info relevant to its section heading; no tangents | L1 (Structure) | LLM extracts irrelevant info as supporting evidence |
| **Intent Match** | Passage directly addresses the query intent it appears under (not tangential) | L1 (Syntax - question framing) | LLM retrieves passage but marks it low-relevance; reader confusion |
| **Information Hierarchy** | Main claim stated first; evidence/nuance follow; not buried | L2 (Semantic Ordering) | LLM misidentifies main point; cites supporting detail as primary claim |
| **Context Management** | Passage contains sufficient context for standalone understanding; minimal forward/backward references | L2 (Reference Resolution) | LLM cannot resolve pronouns/references; hallucinates context |
| **Consistency & Specificity** | Claims are specific (not vague), internally consistent, and consistent with adjacent passages | L3 (Knowledge Integrity) | LLM detects contradiction; hedges or chooses arbitrarily |

---

## 7-Question LLM Readability Self-Test

Run this test on each passage (50-300 words) that you want LLM to extract or cite.

### Q1: Natural Language Quality
**Does the passage use grammatically correct, active-voice sentences without jargon overload?**

**Failure indicators:**
- Passive voice dominates ("was found to be", "has been shown")
- Technical jargon without definition ("ontological commitment")
- Sentence fragments or run-ons

**Test:** Read the passage aloud. Can you say it naturally in 1 breath per sentence?

**PASS example:**
"Internal links distribute page authority across the site. Google's crawler follows these links to discover new pages and assess their importance relative to the homepage."

**FAIL example:**
"The utilization of internal linking architectures as a mechanism for authority distribution has been identified as a critical factor in the indexation priority determination process by search algorithms."

---

### Q2: Structuring (Hierarchy)
**Is the passage nested under a clear, single heading? Does information flow logically?**

**Failure indicators:**
- No heading or heading is vague ("Overview", "Details")
- Sentences jump between topics (para 1: "Links matter", para 2: "Images matter too")
- Information appears out of logical order (evidence before claim)

**Test:** Can you summarize the passage in one sentence using the heading as context?

**PASS example:**
**Heading:** "How Internal Links Affect Page Authority"
**Para 1:** "Internal links pass authority from one page to another."
**Para 2:** "The anchor text of the link signals what the linked page is about."
**Para 3:** "More internal links to a page increase its authority, up to a limit."

**FAIL example:**
**Heading:** "Links and SEO"
**Para 1:** "Internal links pass authority. External links are bought. Anchor text matters."
**Para 2:** "Nofollow links don't pass authority, but they can still drive traffic."

---

### Q3: Chunk Relevance
**Does every sentence in the passage address the section heading? Are there tangents?**

**Failure indicators:**
- Sentence that could belong in a different section
- "By the way..." clauses that distract
- Examples that illustrate a different point than the heading

**Test:** For each sentence, ask: "Does this sentence directly answer the question in the heading?"

**PASS example:**
**Heading:** "When to Use Anchor Text for Internal Links"
**Content:**
- "Use descriptive anchor text that matches the linked page's main topic."
- "Avoid generic anchor text like 'click here' or 'read more'."
- "Match the anchor text to the H1 of the linked page when possible."

(All three sentences address "when/why to use specific anchor text")

**FAIL example:**
**Heading:** "When to Use Anchor Text for Internal Links"
**Content:**
- "Use descriptive anchor text that matches the linked page's main topic."
- "External links can also use anchor text, but they're less effective."
- "Match the anchor text to the H1 of the linked page when possible."

(Sentence 2 introduces external links — off-topic tangent)

---

### Q4: Intent Match
**Does the passage directly address the query or section intent it appears under?**

**Failure indicators:**
- Passage is tangentially related but doesn't answer the implied question
- Reader would need to infer a connection
- Passage answers a different, unstated question

**Test:** State the implied question in the heading. Does the passage answer that specific question?

**PASS example:**
**Heading:** "How to Find Broken Internal Links"
**Implied question:** "What methods can I use to identify broken links on my site?"
**Content:** "Run a site crawler like Screaming Frog. Set it to crawl your domain. Flag all 404 responses. Prioritize fixing 404s on high-authority pages (homepage, pillar pages)."

(Answer directly matches question)

**FAIL example:**
**Heading:** "How to Find Broken Internal Links"
**Implied question:** "What methods can I use to identify broken links on my site?"
**Content:** "Broken links harm user experience. They also signal poor site maintenance to search engines. You should maintain a link health monitoring system."

(Discusses WHY broken links matter, not HOW to find them)

---

### Q5: Information Hierarchy
**Is the main claim stated first, with evidence/nuance following?**

**Failure indicators:**
- Main claim is buried in the second or third paragraph
- Supporting details are introduced before the central point
- Nuance or caveats dominate the opening

**Test:** Read only the first sentence. Does it tell you what the section is about?

**PASS example:**
"Internal links increase page authority when they use contextual anchor text. For example, linking from an article about 'link building' to a page titled 'How to Get Backlinks' signals to Google that the pages are related. Authority flows along the anchor text's semantic direction."

(Main claim stated first; evidence follows)

**FAIL example:**
"It's complicated. There are many factors. Some say internal links don't matter anymore. But in certain contexts, under specific conditions, with the right anchor text, they can increase page authority."

(Main claim is buried; nuance dominates opening)

---

### Q6: Context Management
**Can the passage be understood in isolation, or does it rely on external context?**

**Failure indicators:**
- Pronouns without clear antecedents ("It increases authority" — what is "it"?)
- Forward references ("As we'll see later...")
- Undefined abbreviations ("The EEAT standard")
- Assumes reader knowledge of prior sections

**Test:** Read the passage to someone unfamiliar with the post. Can they understand it without asking questions?

**PASS example:**
"Google's algorithm considers three factors when ranking pages: backlinks, content quality, and user signals. Backlinks are external links pointing to your page. Content quality measures whether your page answers the user's query. User signals include CTR, dwell time, and return rate."

(Each term is defined within or immediately after introduction)

**FAIL example:**
"The algorithm also weights these. As mentioned earlier, they're critical. This mechanism is what allows the system to work. Without it, the whole thing fails."

(Pronouns "these", "it", "the system" lack clear antecedents; reader must infer)

---

### Q7: Consistency & Specificity
**Are claims specific (not vague) and consistent with adjacent passages?**

**Failure indicators:**
- Vague claims ("helps with SEO", "improves rankings")
- Contradictory claims between passages ("links are #1 factor" vs "links are declining")
- Hedging that obscures specificity ("may help", "could improve")
- Statistics without dates or context

**Test:** Highlight every claim. Is each claim testable and specific? Do all claims about the same topic align?

**PASS example:**
"According to a 2025 Moz study, domain authority correlates 0.36 with rankings. This correlation is weaker than in 2020 (0.42), suggesting that content quality and user signals are increasingly important."

(Specific: correlation value, date, trend direction)

**FAIL example:**
"Links help with rankings. Studies show they're important. Some people think they're less important now. It depends."

(Vague: no numbers, no dates, no direction, no specificity)

---

## Interpretation: Binding Constraint Model

**Binding Constraint Rule:** The weakest component determines LLM readability. A passage that scores 6/7 is only as readable as its single failing component.

### Readability Score Interpretation

| Score | LLM Behavior | Action |
|---|---|---|
| 7/7 | Extracts, cites, summarizes accurately. High confidence in retrieval. | Deploy. Refresh on cadence trigger only. |
| 6/7 | Extracts main claim correctly but may misinterpret nuance or misattribute context. Medium-high confidence. | Deploy with caveat in adjacent passage explaining the failing component. Retest after refresh. |
| 5/7 or lower | Hedges, hallucinates context, or cites incorrectly. Low confidence. LLM may prefer competitor's passage. | Do not deploy. Rewrite failing components. Retest before publishing. |

### Identifying the Binding Constraint

1. Run all 7 questions
2. Mark each as PASS (✓) or FAIL (✗)
3. Identify failed question(s)
4. Rewrite ONLY the failed component
5. Retest that component only
6. Re-run all 7 questions on the rewritten section

**Example:**

```
Initial test:
Q1 Natural Language Quality: ✓
Q2 Structuring: ✓
Q3 Chunk Relevance: ✓
Q4 Intent Match: ✗ (passage discusses WHY, not HOW)
Q5 Information Hierarchy: ✓
Q6 Context Management: ✓
Q7 Consistency: ✓

Binding constraint: Q4 (Intent Match)

Action: Rewrite passage to directly answer the HOW question, not the WHY.
Retest Q4 only first, then retest all 7.
```

---

## When to Apply LLM Readability Testing

| Scenario | Trigger | Action |
|---|---|---|
| New passage created | Before publishing | Run 7-question test. If <6/7, rewrite before publishing. |
| Passage refreshed (stats updated) | Before republishing | Run full 7-question test. Changes may have broken context or hierarchy. |
| Passage cited in another article | Before external link | Confirm 7/7 score. Readers may extract and share; readability failure = citation failure. |
| High-traffic passage | Monthly cadence | Random sampling: test 2-3 passages per audit. Catch drift. |
| After LLM retrieval failure | Reactive | If an LLM misinterprets or hallucinates context, test that passage. Likely failure in Q2, Q5, or Q6. |

---

## LLM Readability vs. Human Readability

| Dimension | Human Readability | LLM Readability |
|---|---|---|
| **Scope** | Ease of understanding for a human reader | Processability for an AI system |
| **Key metric** | Reading time, comprehension score (Flesch-Kincaid) | Extraction accuracy, citation confidence, hedge rate |
| **Favors** | Varied sentence length, narrative flow, analogies | Direct language, explicit relationships, no inference required |
| **Penalizes** | Long sentences, jargon (without definition) | Pronouns without antecedents, vague claims, tangents |
| **Test method** | Readability formula, user testing | 7-question self-test, AI extraction test |
| **Improvement strategy** | Simplify vocabulary, shorten sentences, add transitions | Clarify pronouns, remove tangents, add specificity |

**Note:** A passage can be human-readable but LLM-unreadable (too much implied context) or LLM-readable but human-unfriendly (overly explicit, choppy).

---

## Passage-Level AI Processability: Testing Workflow

```bash
#!/bin/bash
# Batch test LLM readability of passages on a page

URL=$1
SECTION_ID=$2

echo "=== LLM READABILITY TEST: $URL#$SECTION_ID ==="

# Extract section
SECTION=$(curl -s "$URL" | sed -n "/$SECTION_ID/,/<h[2-3]/p" | head -n -1)

# Test each question
echo "Q1 Natural Language Quality:"
echo "$SECTION" | grep -iE '(was found to be|has been shown|the utilization of|the mechanism of)' && echo "  ✗ Passive voice detected" || echo "  ✓ Active voice present"

echo "Q2 Structuring (Hierarchy):"
echo "$SECTION" | grep -E '<h[2-3]' | head -1

echo "Q3 Chunk Relevance:"
echo "$SECTION" | grep -iE '(by the way|however, unrelated|tangent:|off-topic)' && echo "  ✗ Tangent detected" || echo "  ✓ On-topic"

echo "Q4 Intent Match:"
echo "$SECTION" | head -20 | grep -iE '(^.*[A-Z][a-z]+.*[?:].*$)' || echo "  ? Manual review: does opening address section intent?"

echo "Q5 Information Hierarchy:"
FIRST_SENT=$(echo "$SECTION" | sed -n '/<p>/,/<\/p>/p' | head -1 | grep -o '[^<>]*' | head -1)
[ -z "$FIRST_SENT" ] && echo "  ? Could not extract opening sentence" || echo "  Opening: $FIRST_SENT"

echo "Q6 Context Management:"
PRONOUNS=$(echo "$SECTION" | grep -o -E '\b(it|they|this|that)\b' | wc -l)
echo "  Pronoun count: $PRONOUNS (high = risk of ambiguity)"

echo "Q7 Consistency & Specificity:"
STATS=$(echo "$SECTION" | grep -o -E '[0-9]+%|[0-9]{4}' | wc -l)
echo "  Data points found: $STATS (high = specific)"
```
