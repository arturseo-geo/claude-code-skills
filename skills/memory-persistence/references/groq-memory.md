# Groq-Powered Memory Operations

Use Groq's free Llama 3.1 models to handle memory operations that would
otherwise consume expensive Claude tokens.

## Setup

```bash
pip install groq
export GROQ_API_KEY="gsk_..."  # Free at console.groq.com
```

## Complete Memory Management Script

Save as `.claude/scripts/memory-manager.py`:

```python
#!/usr/bin/env python3
"""
Memory manager using Groq free tier for scoring, compaction, and dedup.
Saves Claude tokens by offloading bulk memory operations.
"""
import os
import json
import math
import time
import shutil
from datetime import datetime, timezone
from groq import Groq

MEMORY_PATH = ".claude/memory.jsonl"
GROQ_MODEL = "llama-3.1-8b-instant"

client = Groq(api_key=os.environ["GROQ_API_KEY"])


def load_all(path: str = MEMORY_PATH) -> list[dict]:
    """Load all memories from JSONL file."""
    memories = []
    with open(path) as f:
        next(f)  # skip schema
        for line in f:
            if line.strip():
                memories.append(json.loads(line))
    return memories


def local_score(memory: dict) -> float:
    """Score locally using importance + recency decay (no API call)."""
    now = datetime.now(timezone.utc)
    ts = datetime.fromisoformat(memory["ts"].replace("Z", "+00:00"))
    age_days = (now - ts).total_seconds() / 86400
    recency = math.exp(-0.01 * age_days)
    importance = memory.get("importance", 0.7)
    return importance * recency


def groq_score(memories: list[dict], context: str,
               top_n: int = 20) -> list[dict]:
    """Score memories against a context using Groq (free)."""
    scored = []
    batch_size = 15

    for i in range(0, len(memories), batch_size):
        batch = memories[i:i + batch_size]
        batch_text = "\n".join([
            f"{j+1}. [{m['type']}] {m['key']}: {m['value']}"
            for j, m in enumerate(batch)
        ])

        try:
            resp = client.chat.completions.create(
                model=GROQ_MODEL,
                messages=[{
                    "role": "user",
                    "content": f"Rate relevance (0.0-1.0) to: {context}\n\n{batch_text}\n\nJSON array:"
                }],
                temperature=0,
                max_tokens=200
            )
            scores = json.loads(resp.choices[0].message.content)
            if isinstance(scores, dict):
                scores = list(scores.values())[0]
            for m, s in zip(batch, scores):
                scored.append((m, float(s)))
        except Exception:
            for m in batch:
                scored.append((m, local_score(m)))
        time.sleep(2)

    scored.sort(key=lambda x: x[1], reverse=True)
    return [m for m, s in scored[:top_n]]


def find_duplicates(memories: list[dict]) -> list[list[int]]:
    """Find duplicate memories using Groq."""
    # Process in chunks of 50
    all_dupes = []
    for i in range(0, len(memories), 50):
        chunk = memories[i:i + 50]
        chunk_text = "\n".join([
            f"{i+j+1}. [{m['type']}] {m['key']}: {m['value']}"
            for j, m in enumerate(chunk)
        ])

        try:
            resp = client.chat.completions.create(
                model=GROQ_MODEL,
                messages=[{
                    "role": "user",
                    "content": f"Find duplicate/near-duplicate memories. Return JSON array of arrays (1-indexed), e.g. [[1,5],[3,7]]. If none, return [].\n\n{chunk_text[:4000]}"
                }],
                temperature=0,
                max_tokens=500
            )
            dupes = json.loads(resp.choices[0].message.content)
            if isinstance(dupes, list) and all(isinstance(d, list) for d in dupes):
                all_dupes.extend(dupes)
        except Exception:
            pass
        time.sleep(2)

    return all_dupes


def compact(path: str = MEMORY_PATH, keep_top: int = 200):
    """Compact memory file using local scoring. No API calls needed."""
    memories = load_all(path)
    if len(memories) <= keep_top:
        print(f"Only {len(memories)} memories, no compaction needed.")
        return

    scored = [(m, local_score(m)) for m in memories]
    scored.sort(key=lambda x: x[1], reverse=True)
    keepers = [m for m, s in scored[:keep_top]]
    keepers.sort(key=lambda x: x["ts"])

    # Backup
    shutil.copy2(path, path + ".bak")

    # Write
    with open(path) as f:
        schema = f.readline()

    with open(path, "w") as f:
        f.write(schema)
        for m in keepers:
            f.write(json.dumps(m) + "\n")

    print(f"Compacted: {len(memories)} -> {len(keepers)} memories")


def deduplicate(path: str = MEMORY_PATH):
    """Remove duplicate memories using Groq."""
    memories = load_all(path)
    dupes = find_duplicates(memories)

    if not dupes:
        print("No duplicates found.")
        return

    # Collect indices to remove (keep first in each group)
    remove_indices = set()
    for group in dupes:
        for idx in group[1:]:  # Keep first, remove rest
            remove_indices.add(idx - 1)  # Convert to 0-indexed

    keepers = [m for i, m in enumerate(memories) if i not in remove_indices]

    # Backup and write
    shutil.copy2(path, path + ".bak")
    with open(path) as f:
        schema = f.readline()

    with open(path, "w") as f:
        f.write(schema)
        for m in keepers:
            f.write(json.dumps(m) + "\n")

    print(f"Removed {len(remove_indices)} duplicates. {len(keepers)} memories remain.")


def summarise_old(path: str = MEMORY_PATH, age_days: int = 90):
    """Summarise memories older than N days using Groq."""
    memories = load_all(path)
    now = datetime.now(timezone.utc)

    old = []
    recent = []
    for m in memories:
        ts = datetime.fromisoformat(m["ts"].replace("Z", "+00:00"))
        age = (now - ts).total_seconds() / 86400
        if age > age_days:
            old.append(m)
        else:
            recent.append(m)

    if not old:
        print("No old memories to summarise.")
        return

    # Summarise in groups
    summaries = []
    for i in range(0, len(old), 20):
        group = old[i:i + 20]
        group_text = "\n".join([
            f"- [{m['type']}] {m['key']}: {m['value']}"
            for m in group
        ])

        try:
            resp = client.chat.completions.create(
                model=GROQ_MODEL,
                messages=[{
                    "role": "user",
                    "content": f"Summarise into 3-5 key facts:\n\n{group_text}"
                }],
                temperature=0.1,
                max_tokens=300
            )
            summaries.append({
                "ts": datetime.now(timezone.utc).isoformat().replace("+00:00", "Z"),
                "type": "summary",
                "key": f"summary_{i//20}",
                "value": resp.choices[0].message.content.strip(),
                "context": f"Summarised {len(group)} memories older than {age_days} days",
                "importance": 0.8
            })
        except Exception:
            pass
        time.sleep(2)

    # Write: schema + summaries + recent
    shutil.copy2(path, path + ".bak")
    with open(path) as f:
        schema = f.readline()

    with open(path, "w") as f:
        f.write(schema)
        for s in summaries:
            f.write(json.dumps(s) + "\n")
        for m in recent:
            f.write(json.dumps(m) + "\n")

    print(f"Replaced {len(old)} old memories with {len(summaries)} summaries. {len(recent)} recent kept.")


if __name__ == "__main__":
    import sys

    commands = {
        "score": lambda: print(json.dumps(
            groq_score(load_all(), sys.argv[2] if len(sys.argv) > 2 else "general"),
            indent=2
        )),
        "compact": lambda: compact(),
        "dedup": lambda: deduplicate(),
        "summarise": lambda: summarise_old(),
        "stats": lambda: print(f"Total memories: {len(load_all())}"),
    }

    if len(sys.argv) < 2 or sys.argv[1] not in commands:
        print(f"Usage: {sys.argv[0]} <{'|'.join(commands.keys())}> [context]")
        sys.exit(1)

    commands[sys.argv[1]]()
```

## Usage

```bash
# Score memories against a context (for feeding to Claude)
python .claude/scripts/memory-manager.py score "debugging auth errors"

# Compact to top 200 memories (local scoring, no API)
python .claude/scripts/memory-manager.py compact

# Find and remove duplicates (uses Groq)
python .claude/scripts/memory-manager.py dedup

# Summarise memories older than 90 days (uses Groq)
python .claude/scripts/memory-manager.py summarise

# Check memory count
python .claude/scripts/memory-manager.py stats
```

## Automating with Cron

Run weekly maintenance for free:

```bash
# Add to crontab (crontab -e)
# Every Sunday at 3am: compact + dedup
0 3 * * 0 cd /path/to/project && python .claude/scripts/memory-manager.py dedup && python .claude/scripts/memory-manager.py compact
```

## Integration with Stop Hook

Auto-score and prune after each session:

```bash
#!/bin/bash
# .claude/hooks/stop-memory-groq.sh
# After session ends, score memories and compact if needed

MEMORY=".claude/memory.jsonl"
COUNT=$(wc -l < "$MEMORY" 2>/dev/null || echo 0)

if [ "$COUNT" -gt 300 ]; then
    echo "Memory has $COUNT entries, compacting..."
    python .claude/scripts/memory-manager.py compact
fi
```

## Cost Impact

| Operation | On Claude Sonnet | On Groq | Savings |
|---|---|---|---|
| Score 500 memories | ~$2.00 | $0.00 | 100% |
| Summarise 100 old memories | ~$1.50 | $0.00 | 100% |
| Dedup 300 memories | ~$1.00 | $0.00 | 100% |
| Weekly maintenance | ~$4.50/week | $0.00/week | 100% |
| Monthly total | ~$18.00 | $0.00 | 100% |
