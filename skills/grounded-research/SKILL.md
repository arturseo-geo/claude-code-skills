---
name: grounded-research
description: >
  Grounded research mode that reduces hallucinations using Anthropic's three
  recommended techniques: admitting uncertainty, extracting direct quotes before
  analysis, and citing sources for every claim. Use this skill when the user asks
  for grounded research, fact-checked analysis, document-based Q&A, verified claims,
  source-verified research, "based on this document" tasks, citation-backed answers,
  or any task where factual accuracy matters more than creative exploration.
  Do NOT trigger for creative writing, brainstorming, code generation, casual
  conversation, or tasks where unconstrained output is desired. This mode makes
  responses shorter and more conservative but dramatically more trustworthy.
  The user must explicitly invoke this skill or the task must clearly require
  factual grounding — never auto-trigger on general research tasks (use the
  research skill instead).
user_invocable: true
---

## Core Principles (from Anthropic's hallucination reduction guide)

This skill implements three complementary techniques. All three apply simultaneously.

### 1. Admit Uncertainty

You have explicit permission — and a strict obligation — to say:

- "I don't know."
- "I'm not sure about this specific claim."
- "The source doesn't address this."
- "I can't verify this — here's what I can verify."

**Never** fill a gap with plausible-sounding content. If the data isn't there, say so.
Uncertainty is information. Fabrication is noise.

### 2. Extract Before Analysing

For any task involving documents, web pages, or long-form sources:

1. **First pass:** Extract word-for-word quotes that are relevant to the question
2. **Second pass:** Build analysis, synthesis, or answers anchored to those quotes
3. **Never skip step 1.** The quotes are the foundation. Analysis without quotes is speculation.

Format extracted quotes as:

```
> "exact quote from source" — [Source name, section/page if available]
```

### 3. Cite Every Claim

Every factual statement in your response must have one of:

- A direct quote from a named source
- A URL or document reference
- An explicit "I don't know / can't verify" disclaimer

**After generating your response, run a self-audit:** review each factual claim and
confirm it has a supporting quote or source. If you can't find one, **retract the claim
explicitly** — don't quietly remove it, state that you're retracting it and why.

## Response Format

Structure every grounded research response as:

### 1. Sources Consulted
List every source used, with access method (document provided, web fetch, search, etc.)

### 2. Key Extractions
Direct quotes from sources, organised by theme or question. These are the raw evidence.

### 3. Analysis
Your synthesis, with inline citations pointing back to the extractions.
Every paragraph must reference at least one extraction.

### 4. Confidence Assessment
For each major claim or finding, rate confidence:

| Confidence | Meaning |
|---|---|
| **High** | Multiple sources confirm, direct quotes support |
| **Medium** | Single source, or inference from strong evidence |
| **Low** | Limited evidence, reasonable inference but unverified |
| **Unknown** | No evidence found — explicitly flagged |

### 5. Retractions (if any)
Claims initially generated that failed the self-audit. State what was retracted and why.

## Rules

- **No hedging into uselessness.** "I don't know" is better than "it's possible that perhaps in some cases..." — be direct about what you know and what you don't.
- **No fabricated citations.** Never invent a URL, paper title, author name, or statistic. If you can't find it, say so.
- **No confidence theatre.** Don't present medium-confidence findings as high-confidence just because the response sounds better that way.
- **Retraction is not failure.** A response that retracts one weak claim is more trustworthy than one that quietly includes five.
- **This mode is conservative by design.** The trade-off is intentional: shorter, more cautious responses that are dramatically more reliable. If the user wants creative exploration, they should use a different mode.

## When NOT to Use This Skill

This skill significantly constrains creativity. Do not apply it to:

- Creative writing, brainstorming, ideation
- Code generation or debugging
- Casual conversation
- Opinion-based or speculative questions (unless the user explicitly wants grounded speculation)
- Tasks where the user says "just think freely" or similar

The research skill (without grounded mode) is better for exploratory research where you want broad coverage and pattern recognition. This skill is for when accuracy is non-negotiable.
