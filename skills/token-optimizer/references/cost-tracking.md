# Cost Tracking Guide

## Per-Session Tracking

Claude Code displays a cost summary at the end of each session. Key fields:

| Metric | What it tells you |
|---|---|
| Total input tokens | System overhead + your content + conversation history |
| Total output tokens | Claude's responses + thinking tokens |
| Cache read tokens | Tokens served from cache (cheap) |
| Cache write tokens | Tokens written to cache (one-time premium) |
| Total cost | Sum of all token costs at model rates |

### Reading the summary
- **High input, low output**: you are sending too much context per message
  (bloated CLAUDE.md, too many MCPs, long conversation history)
- **High output, low input**: extended thinking is dominating cost (cap
  MAX_THINKING_TOKENS)
- **Low cache reads**: your prefix is unstable or sessions are too short
- **Cost spikes in specific messages**: one message triggered a large
  thinking or tool-use chain

## Per-Task Cost Awareness

Track cost by task type to find optimisation opportunities:

### Logging approach
After each session, note:
```
Date: 2026-03-22
Task: Implement user settings page
Model: Sonnet
Duration: 45 min
Messages: 23
Total cost: $0.45
Notes: Could have batched the 3 form components into one request
```

### Patterns to watch for
| Pattern | Symptom | Fix |
|---|---|---|
| Expensive simple tasks | $0.50+ for basic edits | Switch to Haiku or cap thinking |
| Grinding (many retries) | 10+ messages for one change | Escalate to Opus, clearer instructions |
| Cost creep in long sessions | First 30 min = $0.20, last 30 min = $2.00 | Compact earlier, start fresh sessions |
| Subagent explosion | Many parallel tasks at Opus cost | Set CLAUDE_CODE_SUBAGENT_MODEL=haiku |

## Per-Project Budget Management

### Setting a budget
Estimate based on project phase:

| Phase | Typical daily cost (Sonnet) | Weekly budget |
|---|---|---|
| Exploration / prototyping | $2-5 | $15-35 |
| Active development | $5-15 | $35-100 |
| Code review / maintenance | $1-3 | $7-20 |
| Documentation sprint | $2-5 | $15-35 |

### Budget alerts
Anthropic's API dashboard shows usage and spend. Set up alerts:
1. Go to console.anthropic.com > Settings > Usage
2. Set a monthly spend limit
3. Configure email alerts at 50%, 80%, 100% of limit
4. Review weekly to catch unexpected spikes

### Team cost allocation
For teams sharing an API key:
- Use separate API keys per developer or per project
- Review the usage dashboard weekly
- Compare cost-per-feature across team members to share optimisation tips

## Cost Reduction Checklist

Ordered by impact (apply top-to-bottom):

### Tier 1: High impact (60-80% savings potential)
- [ ] Use Sonnet instead of Opus for standard work
- [ ] Set `MAX_THINKING_TOKENS=10000`
- [ ] Set `CLAUDE_CODE_SUBAGENT_MODEL=haiku`

### Tier 2: Medium impact (20-40% savings)
- [ ] Trim CLAUDE.md to under 1,500 tokens
- [ ] Disable unused MCP servers
- [ ] Batch related requests into single messages
- [ ] Use `/compact` before switching topics

### Tier 3: Incremental gains (5-15% savings)
- [ ] Set `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70`
- [ ] Avoid mid-session CLAUDE.md edits (breaks cache)
- [ ] Use file path references instead of pasting content
- [ ] Start new sessions for unrelated tasks (keeps cache effective)

### Tier 4: Workflow habits
- [ ] Start on Sonnet, escalate to Opus only when stuck
- [ ] Write clear, specific prompts (fewer correction rounds)
- [ ] Provide examples when the expected format is non-obvious
- [ ] Use `/compact` after completing a major subtask

## Cost Comparison Calculator

Quick formula for estimating session cost:

```
Total cost = (input_tokens * input_price) + (output_tokens * output_price)

Where output_tokens includes thinking tokens.

Example (Sonnet, 30-message session):
- Input per message: ~25k tokens (system + CLAUDE.md + history)
- Output per message: ~2k tokens (response) + ~8k (thinking)
- Total input: 30 * 25,000 = 750,000 tokens = $2.25
- Total output: 30 * 10,000 = 300,000 tokens = $4.50
- Cache discount: ~60% of input cached = -$1.35
- Estimated total: $5.40

With MAX_THINKING_TOKENS=5000:
- Total output: 30 * 7,000 = 210,000 tokens = $3.15
- Estimated total: $4.05 (25% savings)
```

## Monthly Cost Benchmarks

For a solo developer using Claude Code as primary IDE assistant:

| Usage level | Model mix | Monthly cost |
|---|---|---|
| Light (1-2 hours/day) | Sonnet default | $50-150 |
| Moderate (4-6 hours/day) | Sonnet + Opus for hard tasks | $150-400 |
| Heavy (8+ hours/day) | Sonnet + Opus + Haiku subagents | $300-800 |
| Optimised heavy (8+ hours/day) | Sonnet default, Haiku subagents, 10k thinking cap | $150-400 |

The difference between "heavy" and "optimised heavy" is the settings in this
skill — same productivity, 50% lower cost.
