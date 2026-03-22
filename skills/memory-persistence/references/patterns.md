# Memory Architecture Patterns

## Append-Only Log

The simplest persistent memory. Each entry is a JSON line appended to a file.
Never edit or delete — only append. This makes the file safe for concurrent writes,
easy to back up, and trivial to replay.

**File format**: `.claude/memory.jsonl`

```json
{"schema": "v1", "fields": ["ts", "type", "key", "value", "context", "importance"]}
{"ts": "2026-03-20T14:00:00Z", "type": "pattern", "key": "auth_fix", "value": "Check CLERK_WEBHOOK_SECRET env var", "context": "session 2026-03-20", "importance": 0.9}
```

**Pros**: simple, append-safe, git-friendly diffs, works offline
**Cons**: linear scan for reads, no relationships, grows unbounded

**When to compact**: if the file exceeds 500 lines, run a compaction pass that:
1. Loads all entries
2. Groups by key — keep only the latest entry per key
3. Drops entries with importance < 0.3
4. Writes a fresh file (keep the old one as `.jsonl.bak`)

---

## Key-Value Store

A JSON object where each key maps to a value. Fast lookup by key, but no history.

**File format**: `.claude/memory.json`

```json
{
  "stack": "Next.js 14 + Supabase + Tailwind",
  "auth_provider": "Clerk",
  "test_command": "pnpm test:unit",
  "deploy_target": "Vercel",
  "last_session": "2026-03-21"
}
```

**Pros**: O(1) lookup, compact, easy to read
**Cons**: no history (overwrites), no relationships, merge conflicts in git

**Best for**: small projects with < 30 stable facts.

---

## Graph-Based Memory

Entities as nodes, relationships as edges. Supports dependency analysis, impact
assessment, and structured queries without loading entire files.

**File format**: `.claude/memory-graph.json`

```json
{
  "nodes": {
    "UserService": {
      "type": "service",
      "file": "src/services/user.ts",
      "description": "Handles user CRUD and profile management",
      "deps": ["Database", "AuthService", "EmailService"]
    },
    "AuthService": {
      "type": "service",
      "file": "src/services/auth.ts",
      "deps": ["Clerk", "UserService"]
    },
    "Database": {
      "type": "external",
      "provider": "Supabase",
      "deps": []
    }
  },
  "edges": [
    {"from": "UserService", "to": "Database", "rel": "reads-from"},
    {"from": "UserService", "to": "Database", "rel": "writes-to"},
    {"from": "AuthService", "to": "UserService", "rel": "creates-users-via"},
    {"from": "AuthService", "to": "Clerk", "rel": "delegates-auth-to"}
  ]
}
```

**Graph queries**:

| Query | Implementation |
|---|---|
| What depends on X? | `[e for e in edges if e["to"] == X]` |
| What does X depend on? | `nodes[X]["deps"]` |
| Impact of changing X? | BFS/DFS from X along reverse edges |
| Orphan detection | Nodes with no incoming or outgoing edges |
| Circular dependency check | DFS cycle detection |

**Pros**: relationship-aware, scales to large projects, enables impact analysis
**Cons**: more complex to maintain, harder to append incrementally

---

## Hybrid Pattern (Recommended for Production)

Combine multiple patterns for different memory needs:

```
.claude/
  CLAUDE.md              # Level 1: always-loaded facts (< 150 lines)
  memory.jsonl           # Level 2: append-only learnings
  memory-graph.json      # Level 3: entity relationships
  sessions/
    2026-03-20.md        # Level 4: session handoffs
    2026-03-21.md
```

**Routing logic**:

| Memory type | Where it goes |
|---|---|
| Stable project facts | CLAUDE.md |
| Debugging patterns learned | memory.jsonl |
| User preferences discovered | memory.jsonl |
| Architectural decisions | CLAUDE.md (brief) + memory.jsonl (detailed) |
| Entity relationships | memory-graph.json |
| Session state / WIP | sessions/YYYY-MM-DD.md |

This hybrid scales from small personal projects to large multi-agent systems.

---

## Pattern Selection Guide

| Project size | Team | Memory pattern |
|---|---|---|
| Small (< 10 files) | Solo | CLAUDE.md only |
| Medium (10-50 files) | Solo | CLAUDE.md + session files |
| Large (50+ files) | Solo | Hybrid (all four) |
| Any size | Team | Hybrid + MCP server for shared memory |
| Multi-agent | Any | JSONL per agent + shared graph |
