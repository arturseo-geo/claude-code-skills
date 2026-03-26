---
name: content-audit
description: >
  Post-publication content maintenance and compliance skill. Audit published
  content for GEO compliance, detect freshness decay, test LLM readability,
  check citation-readiness, and manage re-test cadence. Use this skill whenever
  the user mentions content audit, compliance check, content freshness, stale
  content, citation decay, LLM readability test, re-test cadence, content
  maintenance, statistical currency, pronoun resolution, declarative structure
  check, GEO compliance, post-publication QC, or any task involving reviewing
  published content for AI search quality. This skill covers the maintenance
  lifecycle stage — after seo-geo (plan) and content-pipeline (create).
---

# Content Audit Skill

Post-publication maintenance for AI search visibility. This skill audits
published content for GEO compliance, detects freshness decay, tests LLM
readability at the section level, and manages re-test schedules.

**Lifecycle position:** seo-geo (plan) → content-pipeline (create) → **content-audit (maintain)**

## Reference Files — Load When Relevant

| Topic | File | Load when... |
|---|---|---|
| GEO compliance checklist | `references/compliance-checklist.md` | Auditing pages for schema, structure, voice, CTAs |
| GEO Quality Gate | `references/geo-quality-gate.md` | Per-section quality review (5 questions) |
| Content freshness | `references/content-freshness.md` | Detecting stale statistics, planning updates |
| LLM readability | `references/llm-readability.md` | Testing sections for passage-level AI processability |
| Re-test cadence | `references/retest-cadence.md` | Scheduling and triggering citation rate re-tests |

## When This Skill Triggers

- "audit this page for GEO compliance"
- "check if the content is still fresh"
- "test the LLM readability of this section"
- "when should I re-test citation rates?"
- "is this section citation-ready?"
- "run a content maintenance check"
- "check for stale statistics"
- "are the opening sentences declarative enough?"

## Core Workflow

### 1. Full Page Audit
When asked to audit a page, run all checks from `references/compliance-checklist.md`:
- Schema (Article + FAQPage + Organization + Review JSON-LD)
- Structure (single H1, TL;DR, FAQ, TOC, figure, key takeaways)
- Voice (first-person ≥2, "since 2004" in bio, no self-review)
- Links (internal ≥4, external ≥2, no homepage links)
- CTAs (closing CTA + ebook callout)
- Report PASS/FAIL per check with counts

### 2. Section-Level Quality Gate
When asked to review content quality, run all 5 questions from
`references/geo-quality-gate.md` on every H2 section:
1. Direct declarative opener?
2. All pronouns resolved?
3. Single topic per section?
4. Verifiable data point present?
5. Citation-ready opening sentence?

### 3. Freshness Assessment
When asked about content freshness, use `references/content-freshness.md`:
- Identify every statistic and its source date
- Flag statistics >12 months old
- Apply the 4-priority framework (P1-P4)
- Generate a freshness update plan

### 4. LLM Readability Test
When asked about AI readability, use `references/llm-readability.md`:
- Run the 7-component self-test on each H2 section
- Report which components fail per section
- Identify the binding constraint on retrieval

### 5. Re-Test Cadence
When asked about re-testing, use `references/retest-cadence.md`:
- Check the 5 mandatory re-test triggers
- Recommend re-test schedule based on page age and content type
- Track citation rate baselines

## Output Format

When auditing, produce a structured report:

```markdown
# Content Audit: [Page Title]
**URL:** [url]
**Date:** [date]

## Compliance Score: X/25

| Category | Checks | Pass | Fail |
|----------|--------|------|------|
| Schema | 5 | X | X |
| Structure | 6 | X | X |
| Voice | 4 | X | X |
| Links | 3 | X | X |
| CTAs | 2 | X | X |
| New checks | 5 | X | X |

## Section Quality Gate

| Section (H2) | Opener | Pronouns | Topic | Data | Citation-ready | Result |
|---|---|---|---|---|---|---|
| [heading] | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ | ✅/❌ |

## Freshness Assessment
[Priority level + specific statistics flagged]

## LLM Readability
[Component failures per section]

## Re-Test Status
[Current cadence + next scheduled re-test]

## Recommended Actions
1. [Highest priority fix]
2. [Second priority]
3. [etc.]
```

## Rules

- Always check the RENDERED page (via curl or browser), not just post_content
- WordPress KSES may strip microdata — verify JSON-LD in page source
- Clear nginx cache before auditing: `rm -rf /var/cache/nginx/fastcgi/* && systemctl reload nginx`
- Never mark a page as compliant if any mandatory check fails
- Freshness checks are most critical for pages with time-sensitive queries
