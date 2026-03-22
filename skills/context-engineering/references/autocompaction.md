# Autocompaction Deep Dive

## How Autocompaction Works

When Claude Code's conversation token count crosses the compaction threshold, the system automatically triggers a summarization pass. This is **lossy** — exact variable names, error messages, file paths, and nuanced decisions may be reduced to high-level summaries.

### The Compaction Pipeline

1. **Threshold check** — After each assistant response, total token usage is compared against the trigger point
2. **Summary generation** — Claude produces a compressed representation of the conversation so far
3. **Context replacement** — The original messages are replaced with the summary
4. **Continuation** — The session resumes with the summary as "memory" plus any new messages

### What Gets Lost

Compaction is a summarization, not a compression algorithm. It preserves:
- High-level goals and decisions
- File paths mentioned recently
- Current task state

It tends to lose:
- Exact error messages from early in the session
- Variable names and function signatures discussed but not recently referenced
- Rejected alternatives and the reasoning behind rejecting them
- Specific numeric values (line numbers, counts, thresholds)
- Context about *why* a particular approach was chosen over another

## Trigger Mechanics

### Default Behavior

```
Total capacity:        200k tokens
Autocompact buffer:     33k tokens (hardcoded, cannot be changed)
Default trigger:       ~167k tokens (~83.5% of 200k)
```

The 33k buffer exists to ensure there is always room for the compaction summary itself plus the next user message and assistant response.

### CLAUDE_AUTOCOMPACT_PCT_OVERRIDE

This environment variable controls **when** compaction fires as a percentage of total capacity:

```bash
# Fire earlier — more frequent compactions, less data lost each time
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=60

# Fire later — longer runs between compactions, more data lost each time
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=90
```

**Persistence options:**

```bash
# Option 1: Shell profile (~/.bashrc or ~/.zshrc)
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70

# Option 2: Claude Code settings (project or global)
# .claude/settings.json
{
  "env": {
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "70"
  }
}
```

### Choosing a Threshold

| Value | Behavior | Best for |
|---|---|---|
| 50–60% | Very frequent compaction | Long exploratory sessions, multi-file refactors |
| 65–75% | Balanced | General development work |
| 80–85% | Default range | Short-to-medium sessions |
| 85–95% | Infrequent, large compactions | Quick tasks where you want max context |

**Rule of thumb:** If you regularly lose critical details after compaction, lower the threshold. If your sessions feel choppy from too-frequent compactions, raise it.

## Manual Compaction with /compact

The `/compact` command triggers compaction immediately regardless of token usage. Use it at **logical breakpoints**:

- After completing a feature or fixing a bug
- Before switching to a different area of the codebase
- After a long debugging session where many dead-end paths were explored
- Before asking Claude to do a large new task

### Compaction with Focus

You can guide what the compaction preserves:

```
/compact focus on the auth refactor decisions and the new middleware API
```

This tells the summarizer to prioritize retaining specific information.

## Anti-Patterns

### Relying on Compaction to Remember Everything

Compaction is lossy. If something is critical, write it to a file:
```
Save the API contract we agreed on to .claude/decisions/auth-api.md
```

### Setting the Threshold Too High (>90%)

The compaction summary itself needs room. If you set the threshold to 95%, the system has very little space to work with and the resulting summary may be excessively compressed.

### Never Compacting Manually

Automatic compaction fires at arbitrary points in your conversation. Manual compaction at logical breakpoints produces better summaries because the context has natural structure at those moments.

## Monitoring Compaction

After compaction fires (automatically or manually), verify what was retained:

```
Summarize what you remember about our session so far
```

If critical details are missing, re-inject them:
```
For context: we decided to use JWT with refresh tokens, the middleware
lives in src/middleware/auth.ts, and the failing test was in
tests/auth.test.ts line 47 — a race condition on token refresh.
```
