# Caching Strategies

## How Prompt Caching Works

Anthropic automatically caches prompt prefixes. When consecutive API calls
share the same prefix (system prompt, tools, CLAUDE.md content), the cached
portion is served at a discount.

### Pricing impact
| Token type | Cost relative to standard input |
|---|---|
| Cache write (first time) | 1.25x (25% premium) |
| Cache read (subsequent) | 0.10x (90% discount) |
| Standard input (no cache) | 1.00x |

**Breakeven**: a cached prefix pays for itself after 2 cache reads. By the
third message in a session, caching is saving money.

### Cache lifetime
- Cached prefixes persist for **5 minutes** of inactivity
- Each cache hit resets the 5-minute timer
- Active sessions keep the cache warm automatically
- Long pauses (coffee break, meeting) may invalidate the cache

## Maximising Cache Hits

### Keep instruction files stable
Every edit to CLAUDE.md, skill files, or settings invalidates the cache for
everything after the edit point. Strategies:

1. **Batch edits**: make all CLAUDE.md changes at once, not incrementally
2. **Separate stable from volatile**: put rarely-changing rules in CLAUDE.md,
   put frequently-changing context in conversation messages
3. **Avoid whitespace-only changes**: even trailing spaces invalidate cache

### Front-load static content
The cache works on prefixes — the beginning of the prompt. Structure matters:

```
[System prompt]       <- cached automatically
[Tool definitions]    <- cached automatically
[CLAUDE.md]           <- cached if unchanged
[Skills]              <- cached if unchanged
[Conversation history] <- changes every message (not cached)
```

You cannot control the order of system prompt and tools, but you can ensure
CLAUDE.md and skills are stable so they stay in the cached prefix.

### Session-level optimisation
- **Do similar tasks in the same session**: the shared prefix stays cached
- **Use /compact to reset volatile context**: compaction preserves the prefix
  while clearing conversation history
- **Avoid switching contexts mid-session**: changing CLAUDE.md or enabling new
  MCPs invalidates the cache

## Response Reuse Patterns

### Template-then-apply pattern
Instead of asking Claude to reason about each file independently:

```
Step 1: "Review src/components/Button.tsx and create a test. Show me the
pattern you used."

Step 2: "Apply the same test pattern to all components in src/components/.
Use the Button test as the template."
```

The second step reuses the established pattern without re-reasoning.

### Checklist pattern
For repetitive reviews:

```
"Create a review checklist for React components based on our style guide at
docs/style-guide.md. Then apply it to each file in src/components/, outputting
a pass/fail per item."
```

The checklist is created once, applied many times — no repeated reasoning.

### Summary-and-reference pattern
For long sessions:

```
"Summarise all the changes we've made so far into a bullet list. I'll use
this as context for the next session."
```

Starting the next session with a concise summary instead of replaying the
full conversation saves tokens on input.

## Anti-Patterns That Break Caching

| Anti-pattern | Why it hurts | Fix |
|---|---|---|
| Editing CLAUDE.md mid-session | Invalidates prefix cache | Edit between sessions |
| Toggling MCP servers | Changes tool definitions | Decide before starting |
| Installing skills mid-session | Adds to prefix | Install between sessions |
| Pasting large file contents in messages | Bloats conversation, pushes past cache | Use file paths, let Claude read |

## Measuring Cache Effectiveness

In Claude Code's session summary, look for:
- **Cache read tokens**: higher = better caching
- **Cache write tokens**: one-time cost, amortised over reads
- **Cache read ratio**: `cache_read / (cache_read + standard_input)`

Target: cache read ratio above 60% for sessions longer than 5 messages.

A low cache read ratio means:
- Your prefix is changing too often (CLAUDE.md edits, MCP changes)
- Sessions are too short for caching to pay off
- You are switching between very different tasks within one session
