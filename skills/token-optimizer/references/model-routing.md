# Model Routing Guide

## Available Models and Pricing (2026)

| Model | Input (per 1M tokens) | Output (per 1M tokens) | Context window | Best for |
|---|---|---|---|---|
| Opus | $15.00 | $75.00 | 200k | Complex reasoning, architecture, novel problems |
| Sonnet | $3.00 | $15.00 | 200k | Standard coding, reviews, feature work |
| Haiku | $0.25 | $1.25 | 200k | File reading, grep, simple transforms, subagents |

Note: pricing may change. Check Anthropic's pricing page for current rates.

## Cost Comparison by Task

| Task | Opus cost | Sonnet cost | Haiku cost | Recommended |
|---|---|---|---|---|
| Simple Q&A (3k tokens) | $0.05 | $0.01 | $0.001 | Haiku |
| Code review (10k tokens) | $0.15 | $0.03 | $0.003 | Sonnet |
| Feature implementation (40k) | $0.60 | $0.12 | $0.01 | Sonnet |
| Architecture design (80k) | $1.20 | $0.24 | N/A | Opus |
| Multi-file refactor (100k) | $1.50 | $0.30 | N/A | Sonnet or Opus |
| Codebase audit (200k) | $3.00 | $0.60 | N/A | Sonnet |

## Decision Framework

### Use Opus when:
- Designing a new system from scratch (architecture, data models, API design)
- Debugging subtle issues (race conditions, memory leaks, state inconsistencies)
- Cross-system reasoning (understanding how 5+ services interact)
- Ambiguous requirements that need interpretation and judgement
- Security audits where missing a vulnerability has high cost
- The task has failed on Sonnet after 2+ attempts

### Use Sonnet when (default choice):
- Implementing a feature from a clear spec
- Writing or updating tests
- Refactoring code with well-defined patterns
- Code review and PR feedback
- Documentation writing
- Most day-to-day development work

### Use Haiku when:
- File reading and content extraction
- Running grep/search operations
- Simple text transformations (renaming, formatting)
- Subagent tasks (parallel file processing)
- Quick factual lookups
- Generating boilerplate from templates

## Subagent Routing

Set `CLAUDE_CODE_SUBAGENT_MODEL=haiku` as default. This routes all Task tool
invocations to Haiku, which handles most subagent work (file reading, search,
simple analysis) at 4% of Opus cost.

For subagents that need more reasoning:
- Explicitly state the complexity in the task description
- The main agent can note "this subtask needs Sonnet-level reasoning" but
  the env var applies globally per session
- If many subtasks need Sonnet, temporarily change the env var

## Session-Level Switching

```
/model sonnet    # Start here for most work
/model opus      # Escalate when stuck or for complex architecture
/model haiku     # Downgrade for quick lookups or simple edits
```

**Pattern for cost-efficient sessions**:
1. Start on Sonnet
2. Do the bulk of implementation work
3. Switch to Opus only for the hard parts (design decisions, debugging)
4. Switch back to Sonnet for follow-up work (tests, docs, cleanup)
5. Use Haiku for any batch file processing

## Quality Tradeoffs

| Dimension | Opus | Sonnet | Haiku |
|---|---|---|---|
| Code correctness | Highest | High | Good for simple tasks |
| Architecture quality | Best | Good | Not suitable |
| Following complex instructions | Best | Good | Struggles with nuance |
| Speed (tokens/second) | Slowest | Fast | Fastest |
| Multi-step reasoning | Best | Good | Limited |
| Sticking to constraints | Best | Good | May miss edge cases |

The most common mistake is using Opus for everything. Sonnet handles 80% of
coding tasks at 20% of the cost. Reserve Opus for the 20% where its extra
reasoning depth actually changes the outcome.
