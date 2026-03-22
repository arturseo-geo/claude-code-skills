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
There are four patterns to solve this, ranging from simple to powerful:

| Pattern | Best for | Complexity |
|---|---|---|
| **CLAUDE.md injection** | Small, stable facts | Low |
| **Session summary files** | Resuming complex work | Low |
| **Append-only JSONL** | Evolving agent knowledge | Medium |
| **Graph-based memory** | Relationship-rich knowledge | High |

---

## Pattern 1: CLAUDE.md Injection (Always-On Facts)

Put facts that never change or change rarely directly in `.claude/CLAUDE.md`:

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

**Keep it under 150 lines** — it loads every session, consuming tokens permanently.
Move verbose details to reference files and @-import them only when needed.

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

---

## Pattern 3: Append-Only JSONL Memory (Evolving Knowledge)

For agents that learn over time — each discovered insight gets appended as a JSON line.
Never overwrite; always append. This preserves history and is agent-friendly.

**Schema line 1** (always first — defines the structure):
```json
{"schema": "v1", "fields": ["ts", "type", "key", "value", "context"]}
```

**Append a memory**:
```json
{"ts": "2026-03-20T14:00:00Z", "type": "pattern", "key": "auth_error_fix", "value": "Always check Clerk webhook secret matches env var CLERK_WEBHOOK_SECRET", "context": "Spent 2h debugging this in session 2026-03-20"}
{"ts": "2026-03-20T15:30:00Z", "type": "preference", "key": "test_style", "value": "User prefers Vitest over Jest; never suggest Jest", "context": "Corrected by user"}
{"ts": "2026-03-21T09:00:00Z", "type": "decision", "key": "db_client", "value": "Using Supabase JS v2 client, not direct Postgres", "context": "Architectural decision"}
```

**Reading memory in a new session**:
```python
import json

def load_memories(path: str, types: list[str] = None) -> list[dict]:
    memories = []
    with open(path) as f:
        next(f)  # skip schema line
        for line in f:
            m = json.loads(line)
            if types is None or m["type"] in types:
                memories.append(m)
    return memories

# Load only patterns and decisions (skip preferences)
relevant = load_memories(".claude/memory.jsonl", types=["pattern", "decision"])
```

**Memory types to use**:
| Type | What to store |
|---|---|
| `pattern` | Debugging techniques, workarounds discovered |
| `decision` | Architectural decisions and rationale |
| `preference` | User preferences on style, tools, approach |
| `error` | Recurring errors and their fixes |
| `fact` | Project-specific facts (env names, URLs, keys) |

---

## Pattern 4: Continuous Learning Hook

A `Stop` hook that runs at session end — captures what Claude discovered and appends
to memory automatically. Use `Stop` not `UserPromptSubmit` (runs once per session,
not on every message — no latency impact):

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
{\"ts\": \"$(date -u +%Y-%m-%dT%H:%M:%SZ)\", \"type\": \"[type]\", \"key\": \"[short_key]\", \"value\": \"[what to remember]\", \"context\": \"[brief context]\"}

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

---

## Pattern 5: Graph-Based Memory (Relationship-Rich Knowledge)

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

---

## MCP Memory Servers

For production memory, connect a memory MCP server instead of flat files:

| Server | Storage | Best for |
|---|---|---|
| `@modelcontextprotocol/memory` | In-process key-value | Simple persistence |
| Chroma MCP | Vector DB | Semantic search over memories |
| SQLite MCP | Relational | Structured queries over memory |

**Important**: if a memory MCP is configured but no skill/hook actually writes to it,
disable it — it consumes context window tokens with no benefit.

---

## Quick Reference — Which Pattern to Use?

```
Need to remember forever? → CLAUDE.md (if < 20 lines) or JSONL memory
Resuming complex work tomorrow? → Session summary file
Agent that learns over time? → Append-only JSONL + Stop hook
Complex codebase relationships? → Graph-based JSON
Production multi-user agent? → Memory MCP server
```
