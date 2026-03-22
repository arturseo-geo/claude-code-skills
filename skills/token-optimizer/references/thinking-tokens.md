# Extended Thinking Token Guide

## What Extended Thinking Costs

Extended thinking tokens are **output tokens** billed at output rates, even
though you never see them. They are the single largest hidden cost in most
Claude Code sessions.

| Setting | Tokens per request | Cost per request (Sonnet) | Cost per request (Opus) |
|---|---|---|---|
| Default (31,999) | Up to 31,999 | Up to $0.48 | Up to $2.40 |
| 20,000 | Up to 20,000 | Up to $0.30 | Up to $1.50 |
| 10,000 | Up to 10,000 | Up to $0.15 | Up to $0.75 |
| 5,000 | Up to 5,000 | Up to $0.08 | Up to $0.38 |
| 0 (disabled) | 0 | $0.00 | $0.00 |

Over a 50-message session, the difference between default and 10,000 cap:
- Sonnet: ~$16.50 saved
- Opus: ~$82.50 saved

## How to Set MAX_THINKING_TOKENS

### Per-project (recommended)
In `.claude/settings.json`:
```json
{
  "env": {
    "MAX_THINKING_TOKENS": "10000"
  }
}
```

### Per-session (shell)
```bash
MAX_THINKING_TOKENS=10000 claude
```

### Per-task (in conversation)
```
Disable extended thinking for this task -- it's straightforward.
```
Note: this is a soft instruction, not a guaranteed override. The env var is
the reliable mechanism.

## When Thinking Helps

Extended thinking improves results when the task requires:
- **Multi-step planning**: designing an architecture before writing code
- **Constraint satisfaction**: balancing multiple requirements simultaneously
- **Novel problem solving**: tasks unlike common training examples
- **Error analysis**: debugging by reasoning through execution flow
- **Cross-file reasoning**: understanding how changes propagate across a system

## When Thinking Hurts (Wastes Tokens)

Extended thinking adds cost without improving results when:
- **Following a clear pattern**: "Add a test like the existing ones"
- **Simple edits**: renaming, formatting, adding imports
- **File reading and search**: grep, glob, reading file contents
- **Boilerplate generation**: creating files from templates
- **Repetitive operations**: applying the same change to many files

## Recommended Settings by Task Type

| Task category | MAX_THINKING_TOKENS | Rationale |
|---|---|---|
| System architecture design | 25000-31999 | Complex multi-factor reasoning |
| Debugging subtle issues | 20000 | Need to trace execution paths |
| New feature implementation | 10000 | Some planning, mostly execution |
| Code review | 10000 | Need to spot issues |
| Refactoring | 5000 | Pattern application, low novelty |
| Test writing | 5000 | Following existing patterns |
| Documentation | 5000 | Straightforward generation |
| File operations, search | 0 | No reasoning needed |
| Batch formatting | 0 | Mechanical task |

## Tuning Strategy

1. **Start at 10,000** for general development work
2. **Monitor quality**: if Claude's first attempts consistently fail, increase
3. **Track the pattern**: if thinking tokens are high but output is simple,
   decrease
4. **Task-switch**: bump to 20,000+ before architecture sessions, drop to
   5,000 for cleanup

## Thinking Tokens and Model Interaction

| Model | Default thinking | Recommended cap | Notes |
|---|---|---|---|
| Opus | 31,999 | 20,000 | Opus already reasons well; cap saves money |
| Sonnet | 31,999 | 10,000 | Sweet spot for most coding tasks |
| Haiku | 31,999 | 5,000 | Haiku's thinking is less valuable; keep low |

The key insight: Opus at 20,000 thinking tokens often produces better results
than Sonnet at 31,999 thinking tokens, because the base model quality matters
more than the thinking budget.

## Measuring Impact

After changing MAX_THINKING_TOKENS, compare:
- Task completion rate (did it finish correctly?)
- Number of turns (more turns = thinking cap too low)
- Total session cost (visible in Claude Code summary)

If lowering the cap causes more back-and-forth corrections, the net cost may
be higher. Find the minimum cap where first-attempt success stays high.
