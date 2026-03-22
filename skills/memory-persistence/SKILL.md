---
name: memory-persistence
description: >
  Implement cross-session memory, persistent agent knowledge, and session
  continuity for Claude Code. Use this skill whenever the user wants Claude
  to remember things between sessions, asks how to avoid re-explaining context
  each day, wants to build a memory system for an agent, mentions CLAUDE.md
  memory, session files, append-only memory, JSONL memory, graph-based memory,
  knowledge persistence, continuous learning, saving what Claude discovered
  during a session, or resuming work where they left off. Also trigger when
  the user is frustrated that Claude forgot something from a previous session,
  wants to build a personal knowledge base inside Claude Code, or needs
  project-specific facts to survive across compaction events.
---

# Memory Persistence Skill

## Memory Architecture Overview

Claude Code has no built-in cross-session memory — every new session starts blank.
There are six patterns to solve this, ranging from simple to powerful:

| Pattern | Best for | Complexity |
|---|---|---|
| **CLAUDE.md injection** | Small, stable facts | Low |
| **Session summary files** | Resuming complex work | Low |
| **Append-only JSONL** | Evolving agent knowledge | Medium |
| **Graph-based memory** | Relationship-rich knowledge | Medium |
| **Hooks (stop/pre/post)** | Auto-saving and auto-loading | Medium |
| **MCP memory servers** | Production multi-user agents | High |

---

## Pattern 1: CLAUDE.md Injection (Always-On Facts)

CLAUDE.md is Claude Code's built-in memory layer. It loads automatically at session
start, making it the simplest persistence mechanism. There are three levels:

| File | Scope | Loads when |
|---|---|---|
| `~/.claude/CLAUDE.md` | All projects (global) | Every session |
| `CLAUDE.md` (project root) | This project only | When in project dir |
| `.claude/CLAUDE.md` | This project (hidden) | When in project dir |

All three merge at session start. Use the right level for each fact.

**Example CLAUDE.md structure**:
```markdown
## Project Context
- Stack: Next.js 14, Supabase, Tailwind, Vercel
- Auth: Clerk (not NextAuth)
- Primary branch: main (never commit directly)
- Test command: pnpm test:unit
- Deployment: auto on push to main via Vercel

## Key Decisions
- We use Zod for all validation, not Yup
- All API routes live in /app/api, never /pages/api
- Error handling pattern: see /lib/errors.ts
```

### CLAUDE.md Size Limits

**Keep it under 150 lines** — it loads every session, consuming tokens permanently.
Move verbose details to reference files and @-import them only when needed.

| Content type | Put in CLAUDE.md? | Alternative |
|---|---|---|
| Stack, tools, test commands | Yes | — |
| Architectural decisions (< 10) | Yes | — |
| Long reference tables | No | `references/*.md` + @-import |
| API docs | No | Separate file, load on demand |
| Session state | No | Session files (Pattern 2) |

### Router Pattern for Large Projects

When CLAUDE.md would exceed 150 lines, use a router that points to detail files:

```markdown
## Memory Index
- [Architecture](references/architecture.md) — system design, service map
- [Conventions](references/conventions.md) — naming, patterns, anti-patterns
- [API Reference](references/api.md) — endpoints, auth, rate limits
- [Decisions Log](references/decisions.md) — ADRs with rationale
```

Claude loads only the index (4 lines). When it needs architecture details, it
reads the referenced file. This scales to thousands of lines of memory without
bloating the context window.

---

## Pattern 2: Session Summary Files (Resumable Work)

At the end of a working session, have Claude write a handoff file:

**Prompt to use at session end:**
```
Summarise this session for tomorrow. Include:
- What we were building and current state
- Exact file paths changed and what changed
- Decisions made and why
- Next steps in order
- Any blockers or open questions
Save to .claude/sessions/YYYY-MM-DD.md
```

**To resume next session:**
```
@.claude/sessions/2026-03-20.md — pick up where we left off
```

**Session file template** (`.claude/sessions/template.md`):
```markdown
# Session: [DATE]
## Goal
[What we set out to do]

## State
[Current state — what's working, what's not]

## Files Changed
- path/to/file.ts — [what changed and why]

## Decisions
- [Decision] because [reason]

## Next Steps
1. [First thing to do]
2. [Second thing]

## Blockers
- [Any open question or blocker]
```

### Session Continuity Across Compaction

When a session is long, Claude Code compacts context to free tokens. Session files
survive compaction because they are on disk, not in context. To protect critical
state during long sessions:

1. Write state to disk early — do not keep critical decisions only in chat
2. Re-read the session file after compaction (`@.claude/sessions/today.md`)
3. Use incremental writes — append to the file as the session progresses

---

## Pattern 3: Append-Only JSONL Memory (Evolving Knowledge)

For agents that learn over time — each discovered insight gets appended as a JSON line.
Never overwrite; always append. This preserves history and is agent-friendly.

**Schema line 1** (always first — defines the structure):
```json
{"schema": "v1", "fields": ["ts", "type", "key", "value", "context", "importance"]}
```

**Append a memory**:
```json
{"ts": "2026-03-20T14:00:00Z", "type": "pattern", "key": "auth_error_fix", "value": "Always check Clerk webhook secret matches env var CLERK_WEBHOOK_SECRET", "context": "Spent 2h debugging this in session 2026-03-20", "importance": 0.9}
{"ts": "2026-03-20T15:30:00Z", "type": "preference", "key": "test_style", "value": "User prefers Vitest over Jest; never suggest Jest", "context": "Corrected by user", "importance": 0.8}
{"ts": "2026-03-21T09:00:00Z", "type": "decision", "key": "db_client", "value": "Using Supabase JS v2 client, not direct Postgres", "context": "Architectural decision", "importance": 1.0}
```

**Reading memory in a new session**:
```python
import json

def load_memories(path: str, types: list[str] = None, min_importance: float = 0.0) -> list[dict]:
    memories = []
    with open(path) as f:
        next(f)  # skip schema line
        for line in f:
            m = json.loads(line)
            if types and m["type"] not in types:
                continue
            if m.get("importance", 1.0) < min_importance:
                continue
            memories.append(m)
    return memories

# Load only high-importance patterns and decisions
relevant = load_memories(".claude/memory.jsonl", types=["pattern", "decision"], min_importance=0.7)
```

### Memory Types

| Type | What to store | Typical importance |
|---|---|---|
| `fact` | Project-specific facts (env names, URLs, keys) | 0.7 - 1.0 |
| `decision` | Architectural decisions and rationale | 0.8 - 1.0 |
| `pattern` | Debugging techniques, workarounds discovered | 0.6 - 0.9 |
| `preference` | User preferences on style, tools, approach | 0.7 - 0.9 |
| `error` | Recurring errors and their fixes | 0.5 - 0.8 |
| `relationship` | How entities relate (X depends on Y) | 0.5 - 0.8 |

### Memory Decay and Relevance Scoring

Not all memories stay relevant forever. Apply decay to prioritize recent knowledge:

```python
import math
from datetime import datetime, timezone

def score_memory(memory: dict, now: datetime = None) -> float:
    """Score = importance * recency_decay"""
    now = now or datetime.now(timezone.utc)
    ts = datetime.fromisoformat(memory["ts"].replace("Z", "+00:00"))
    age_days = (now - ts).total_seconds() / 86400
    recency = math.exp(-0.01 * age_days)  # half-life ~69 days
    importance = memory.get("importance", 0.7)
    return importance * recency
```

**Retrieval strategy**: load all memories, score each, return top-N. This keeps
context lean while surfacing the most relevant knowledge.

---

## Pattern 4: Graph-Based Memory (Relationship-Rich Knowledge)

For complex projects where entities relate to each other — use a graph structure
stored as a JSON file. Faster context loading, fewer tokens, safer for refactoring.

```json
{
  "nodes": {
    "UserModel": {"type": "entity", "file": "lib/models/user.ts", "deps": ["AuthService", "DB"]},
    "AuthService": {"type": "service", "file": "lib/auth.ts", "deps": ["Clerk", "UserModel"]},
    "DB": {"type": "external", "url": "Supabase"}
  },
  "edges": [
    {"from": "UserModel", "to": "AuthService", "rel": "used-by"},
    {"from": "AuthService", "to": "Clerk", "rel": "delegates-auth-to"}
  ]
}
```

**When to use**: projects with 20+ modules, microservices, or complex dependency chains.
Claude can query: "what depends on AuthService?" without loading all files.

### Graph Queries

Common queries an agent can run against the graph:

| Query | How |
|---|---|
| What depends on X? | Filter edges where `from == X` |
| What does X depend on? | Read `nodes[X].deps` |
| Impact of changing X? | BFS from X along `used-by` edges |
| Which services touch the DB? | Filter nodes where `"DB"` in deps |

---

## Pattern 5: Hooks for Auto-Save and Auto-Load

Hooks let you automate memory operations so the user never has to remember to save.

### Stop Hook — Auto-Save Session Learnings

A `Stop` hook runs at session end. It captures what Claude discovered and appends
to memory automatically. Use `Stop` not `UserPromptSubmit` — runs once per session,
not on every message, so no latency impact.

**`.claude/hooks/stop-memory.sh`**:
```bash
#!/bin/bash
# Runs at end of every session
# Asks Claude to extract learnings and append to JSONL

claude -p "Review this session. Extract any:
- Debugging patterns discovered
- Project-specific facts learned
- User preferences observed
- Decisions made

Format each as a JSON line:
{\"ts\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\", \"type\": \"[type]\", \"key\": \"[short_key]\", \"value\": \"[what to remember]\", \"context\": \"[brief context]\", \"importance\": [0.0-1.0]}

Append to .claude/memory.jsonl — do not overwrite."
```

Register in `.claude/settings.json`:
```json
{
  "hooks": {
    "Stop": [{"command": "bash .claude/hooks/stop-memory.sh"}]
  }
}
```

### PreToolUse Hook — Auto-Load Relevant Memory

Load memory before specific tool calls to give Claude context exactly when needed:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "bash .claude/hooks/load-memory.sh"
      }
    ]
  }
}
```

**`.claude/hooks/load-memory.sh`**:
```bash
#!/bin/bash
# Before editing files, surface relevant memories
# Read last 20 memories and output as context
tail -20 .claude/memory.jsonl 2>/dev/null || true
```

### PostToolUse Hook — Capture Outcomes

After a tool runs, record what happened for future sessions:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Bash",
        "command": "bash .claude/hooks/record-outcome.sh"
      }
    ]
  }
}
```

---

## Pattern 6: MCP Memory Servers

For production memory, connect a memory MCP server instead of flat files:

| Server | Storage | Best for |
|---|---|---|
| `@modelcontextprotocol/memory` | In-process key-value | Simple persistence |
| Chroma MCP | Vector DB | Semantic search over memories |
| SQLite MCP | Relational | Structured queries over memory |

**Important**: if a memory MCP is configured but no skill/hook actually writes to it,
disable it — it consumes context window tokens with no benefit.

---

## Memory Retrieval Strategies

When memory grows large, you need strategies to load only what matters:

| Strategy | How it works | Best for |
|---|---|---|
| **Recency** | Load last N entries | Fast-moving projects |
| **Type filter** | Load only `decision` + `pattern` | Focused recall |
| **Importance threshold** | Load entries with importance >= 0.7 | Mature memory stores |
| **Decay scoring** | Score = importance * recency_decay | Balanced retrieval |
| **Semantic search** | Vector similarity via MCP | Large memory stores (1000+) |
| **Key prefix** | Filter by key prefix (`auth_*`) | Domain-specific queries |

### Combining Strategies

For best results, combine multiple strategies:

```python
def retrieve_relevant(path: str, query_types: list[str], top_n: int = 20) -> list[dict]:
    all_memories = load_memories(path, types=query_types)
    scored = [(m, score_memory(m)) for m in all_memories]
    scored.sort(key=lambda x: x[1], reverse=True)
    return [m for m, s in scored[:top_n]]
```

---

## Quick Reference — Which Pattern to Use?

```
Need to remember forever?          -> CLAUDE.md (if < 20 lines) or JSONL memory
Resuming complex work tomorrow?    -> Session summary file
Agent that learns over time?       -> Append-only JSONL + Stop hook
Complex codebase relationships?    -> Graph-based JSON
Auto-save without user action?     -> Stop hook + JSONL
Production multi-user agent?       -> Memory MCP server
Surviving context compaction?      -> Write to disk early and often
```

---

## Anti-Patterns to Avoid

| Anti-pattern | Why it fails | Fix |
|---|---|---|
| CLAUDE.md over 300 lines | Wastes tokens every session | Router pattern + reference files |
| Overwriting memory files | Loses history, breaks diffs | Append-only JSONL |
| Memory in chat only | Lost on compaction or new session | Write to disk |
| No importance scoring | All memories treated equal | Add importance field |
| Hook on every message | Latency on each prompt | Use Stop hook (once per session) |
| MCP server with no writes | Token cost, zero benefit | Disable unused MCP tools |
