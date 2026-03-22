# Memory Retrieval Strategies

As memory grows, loading everything becomes impractical. This reference covers
strategies for retrieving only the memories that matter.

## Strategy Overview

| Strategy | Complexity | Best for | Memory size |
|---|---|---|---|
| Load all | None | < 50 entries | Small |
| Recency (tail -N) | Low | Fast-moving projects | < 200 |
| Type filter | Low | Role-specific queries | < 500 |
| Key prefix | Low | Domain-scoped lookups | < 500 |
| Importance threshold | Low | Mature memory stores | < 1000 |
| Decay scoring | Medium | Balanced retrieval | < 1000 |
| Combined scoring | Medium | Production agents | < 5000 |
| Semantic search (MCP) | High | Large knowledge bases | 5000+ |

---

## Recency — Load Last N

The simplest strategy. Load the most recent N entries, assuming newer = more relevant.

```bash
# Load last 30 memories (skip schema line)
tail -30 .claude/memory.jsonl
```

```python
def load_recent(path: str, n: int = 30) -> list[dict]:
    import json
    with open(path) as f:
        lines = f.readlines()[1:]  # skip schema
    return [json.loads(line) for line in lines[-n:]]
```

**When to use**: projects where context changes rapidly (early development, prototyping).
**Limitation**: misses important old memories (e.g., a critical decision from week 1).

---

## Type Filter — Load by Category

Load only specific memory types relevant to the current task.

```python
def load_by_type(path: str, types: list[str]) -> list[dict]:
    import json
    memories = []
    with open(path) as f:
        next(f)  # skip schema
        for line in f:
            m = json.loads(line)
            if m["type"] in types:
                memories.append(m)
    return memories

# Before making architectural decisions
decisions = load_by_type(".claude/memory.jsonl", ["decision", "fact"])

# Before debugging
debug_help = load_by_type(".claude/memory.jsonl", ["error", "pattern"])

# Before writing code
preferences = load_by_type(".claude/memory.jsonl", ["preference", "decision"])
```

**When to use**: when the task type is clear and you want focused recall.

---

## Key Prefix — Domain-Scoped Lookup

Filter by key prefix to retrieve memories about a specific subsystem.

```python
def load_by_prefix(path: str, prefix: str) -> list[dict]:
    import json
    memories = []
    with open(path) as f:
        next(f)
        for line in f:
            m = json.loads(line)
            if m["key"].startswith(prefix):
                memories.append(m)
    return memories

# Everything about authentication
auth_memories = load_by_prefix(".claude/memory.jsonl", "auth_")

# Everything about the database
db_memories = load_by_prefix(".claude/memory.jsonl", "db_")
```

**Naming convention**: use prefixes consistently when writing memories:
- `auth_` — authentication and authorization
- `db_` — database and queries
- `api_` — API endpoints and integrations
- `ui_` — frontend and styling
- `deploy_` — deployment and infrastructure
- `test_` — testing patterns and fixtures

---

## Importance Threshold — Load Only High-Value Memories

Skip low-importance entries to reduce context size.

```python
def load_important(path: str, min_importance: float = 0.7) -> list[dict]:
    import json
    memories = []
    with open(path) as f:
        next(f)
        for line in f:
            m = json.loads(line)
            if m.get("importance", 0.7) >= min_importance:
                memories.append(m)
    return memories
```

**Importance guidelines when writing memories**:

| Score | Meaning | Example |
|---|---|---|
| 1.0 | Critical — always load | "Never use force push on main" |
| 0.8 - 0.9 | Important — load by default | "Auth uses Clerk, not NextAuth" |
| 0.5 - 0.7 | Useful — load when relevant | "User prefers tabs over spaces" |
| 0.3 - 0.4 | Minor — load only in deep searches | "Tried X, it didn't work" |
| < 0.3 | Ephemeral — candidate for compaction | "Temp fix applied today" |

---

## Decay Scoring — Time-Weighted Relevance

Older memories lose relevance unless they are high-importance. Decay scoring
balances recency and importance.

```python
import json
import math
from datetime import datetime, timezone

def score_memory(memory: dict, now: datetime = None) -> float:
    """Score = importance * recency_decay

    Half-life of ~69 days means a memory loses half its recency
    score after ~10 weeks. High importance offsets decay.
    """
    now = now or datetime.now(timezone.utc)
    ts = datetime.fromisoformat(memory["ts"].replace("Z", "+00:00"))
    age_days = (now - ts).total_seconds() / 86400
    recency = math.exp(-0.01 * age_days)
    importance = memory.get("importance", 0.7)
    return importance * recency

def load_top_scored(path: str, top_n: int = 30) -> list[dict]:
    memories = []
    with open(path) as f:
        next(f)
        for line in f:
            memories.append(json.loads(line))
    scored = [(m, score_memory(m)) for m in memories]
    scored.sort(key=lambda x: x[1], reverse=True)
    return [m for m, s in scored[:top_n]]
```

**Tuning the decay rate**:

| Decay constant | Half-life | Good for |
|---|---|---|
| 0.001 | ~693 days | Long-running projects |
| 0.005 | ~139 days | Seasonal projects |
| 0.01 | ~69 days | Active development (default) |
| 0.02 | ~35 days | Fast iteration |
| 0.05 | ~14 days | Sprint-based work |

---

## Combined Scoring — Production Strategy

For production agents, combine multiple signals:

```python
def combined_score(memory: dict, query_types: list[str] = None,
                   query_prefix: str = None) -> float:
    """Multi-factor scoring:
    - Base: importance * recency_decay
    - Bonus: +0.2 if type matches query
    - Bonus: +0.3 if key prefix matches query
    """
    base = score_memory(memory)

    type_bonus = 0.2 if query_types and memory["type"] in query_types else 0.0
    prefix_bonus = 0.3 if query_prefix and memory["key"].startswith(query_prefix) else 0.0

    return base + type_bonus + prefix_bonus

def retrieve_relevant(path: str, query_types: list[str] = None,
                      query_prefix: str = None, top_n: int = 30) -> list[dict]:
    memories = []
    with open(path) as f:
        next(f)
        for line in f:
            memories.append(json.loads(line))

    scored = [(m, combined_score(m, query_types, query_prefix)) for m in memories]
    scored.sort(key=lambda x: x[1], reverse=True)
    return [m for m, s in scored[:top_n]]
```

---

## Semantic Search via MCP

For very large memory stores (1000+ entries), use a vector database MCP server
for semantic similarity search.

**Setup with Chroma MCP**:
```json
{
  "mcpServers": {
    "memory": {
      "command": "npx",
      "args": ["-y", "@anthropic/chroma-mcp"],
      "env": {
        "CHROMA_PERSIST_DIR": ".claude/chroma_db"
      }
    }
  }
}
```

**Writing to vector memory**:
```
Store this memory: "Clerk webhook secret must match CLERK_WEBHOOK_SECRET env var"
Tags: auth, debugging, clerk
```

**Querying vector memory**:
```
Search memories related to "authentication errors"
```

**When to use**: when keyword/prefix matching is insufficient and you need
"find memories similar to this concept" capability.

---

## Memory Compaction

When the JSONL file grows too large (500+ lines), compact it:

```python
def compact_memory(path: str, keep_top_n: int = 200, min_importance: float = 0.3):
    """Compact memory file by keeping only the most relevant entries."""
    import json, shutil

    # Load all
    with open(path) as f:
        schema_line = f.readline()
        memories = [json.loads(line) for line in f]

    # Score and sort
    scored = [(m, score_memory(m)) for m in memories]
    scored.sort(key=lambda x: x[1], reverse=True)

    # Keep top N above minimum importance
    keepers = [m for m, s in scored[:keep_top_n] if m.get("importance", 0.7) >= min_importance]

    # Backup original
    shutil.copy2(path, path + ".bak")

    # Write compacted file
    with open(path, "w") as f:
        f.write(schema_line)
        for m in sorted(keepers, key=lambda x: x["ts"]):
            f.write(json.dumps(m) + "\n")

    return len(memories), len(keepers)
```

Run compaction monthly or when the file exceeds 500 lines.
