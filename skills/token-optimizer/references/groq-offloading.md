# Groq Offloading Guide

Groq provides free-tier access to open-source LLMs (Llama 3.1 8B, 70B) with
the fastest inference available. Use it to offload bulk tasks that would waste
Anthropic tokens.

## Setup

```bash
# 1. Get free API key at console.groq.com
# 2. Install SDK
pip install groq

# 3. Set environment variable
export GROQ_API_KEY="gsk_your_key_here"
```

## Available Models (Free Tier)

| Model | Tokens/min | RPM | Best for |
|---|---|---|---|
| llama-3.1-8b-instant | 131,072 | 30 | Classification, tagging, simple transforms |
| llama-3.1-70b-versatile | 131,072 | 30 | Content generation, summarisation, analysis |
| llama-3.3-70b-versatile | 131,072 | 30 | Latest Llama, best quality on free tier |
| mixtral-8x7b-32768 | 32,768 | 30 | Code generation, multilingual |

## Integration Patterns

### Pattern 1: Bulk Classification

Classify hundreds of items for $0 instead of $2+ on Claude.

```python
import os, json, time
from groq import Groq

client = Groq(api_key=os.environ["GROQ_API_KEY"])

def classify_batch(items: list[str], categories: list[str],
                   model: str = "llama-3.1-8b-instant") -> list[str]:
    """Classify items into categories using Groq free tier."""
    results = []
    cats = ", ".join(categories)
    for i, item in enumerate(items):
        try:
            resp = client.chat.completions.create(
                model=model,
                messages=[{
                    "role": "user",
                    "content": f"Classify into exactly one category ({cats}): {item}\n\nCategory:"
                }],
                temperature=0,
                max_tokens=50
            )
            results.append(resp.choices[0].message.content.strip())
        except Exception as e:
            if "rate_limit" in str(e).lower():
                time.sleep(2)  # Back off on rate limit
                continue
            results.append("UNKNOWN")
        # Respect rate limits: 30 RPM = 1 every 2 seconds
        if (i + 1) % 25 == 0:
            time.sleep(5)
    return results
```

### Pattern 2: Bulk Summarisation

Summarise 100 articles/pages before feeding to Claude for analysis.

```python
def summarise_batch(texts: list[str], max_words: int = 100,
                    model: str = "llama-3.1-70b-versatile") -> list[str]:
    """Summarise texts on Groq, feed summaries to Claude for analysis."""
    summaries = []
    for text in texts:
        resp = client.chat.completions.create(
            model=model,
            messages=[{
                "role": "user",
                "content": f"Summarise in {max_words} words or less:\n\n{text[:4000]}"
            }],
            temperature=0.1,
            max_tokens=200
        )
        summaries.append(resp.choices[0].message.content.strip())
        time.sleep(2)
    return summaries

# Then feed concise summaries to Claude instead of full texts
# 100 full articles = ~500k tokens on Claude (~$1.50)
# 100 summaries = ~10k tokens on Claude (~$0.03)
```

### Pattern 3: Memory Scoring and Compaction

Score hundreds of JSONL memories on Groq, feed only top-N to Claude.

```python
def score_memories_with_groq(memories: list[dict], context: str,
                              model: str = "llama-3.1-8b-instant") -> list[tuple]:
    """Use Groq to score memory relevance instead of Claude tokens."""
    scored = []
    batch_size = 10  # Send 10 memories per request to reduce RPM

    for i in range(0, len(memories), batch_size):
        batch = memories[i:i+batch_size]
        batch_text = "\n".join([
            f"{j+1}. [{m['type']}] {m['key']}: {m['value']}"
            for j, m in enumerate(batch)
        ])
        resp = client.chat.completions.create(
            model=model,
            messages=[{
                "role": "user",
                "content": f"Rate each memory's relevance to this context (0.0-1.0):\n\nContext: {context}\n\nMemories:\n{batch_text}\n\nReturn JSON array of scores, e.g. [0.8, 0.3, ...]"
            }],
            temperature=0,
            max_tokens=200,
            response_format={"type": "json_object"}
        )
        try:
            scores = json.loads(resp.choices[0].message.content)
            if isinstance(scores, dict):
                scores = list(scores.values())[0]
            for m, s in zip(batch, scores):
                scored.append((m, float(s)))
        except:
            for m in batch:
                scored.append((m, 0.5))
        time.sleep(2)

    scored.sort(key=lambda x: x[1], reverse=True)
    return scored

# Score 500 memories for free, feed top 20 to Claude
scored = score_memories_with_groq(all_memories, "authentication debugging")
top_memories = [m for m, s in scored[:20]]
```

### Pattern 4: SEO Meta Description Generation (Bulk)

Generate meta descriptions for 100 pages on Groq, review on Claude.

```python
def generate_meta_descriptions(pages: list[dict],
                                model: str = "llama-3.1-70b-versatile") -> list[str]:
    """Generate SEO meta descriptions using Groq free tier."""
    descriptions = []
    for page in pages:
        resp = client.chat.completions.create(
            model=model,
            messages=[{
                "role": "user",
                "content": f"Write a 155-character SEO meta description for:\nTitle: {page['title']}\nContent: {page['content'][:1000]}\n\nMeta description:"
            }],
            temperature=0.3,
            max_tokens=100
        )
        descriptions.append(resp.choices[0].message.content.strip())
        time.sleep(2)
    return descriptions
```

### Pattern 5: Draft-Then-Polish Workflow

Generate first drafts on Groq (free), polish on Claude (quality).

```python
def draft_on_groq_polish_on_claude(topic: str, outline: str) -> str:
    """
    Step 1: Generate rough draft on Groq (free)
    Step 2: Feed draft to Claude for polishing (cheaper than writing from scratch)
    """
    # Step 1: Free draft on Groq
    draft_resp = client.chat.completions.create(
        model="llama-3.1-70b-versatile",
        messages=[{
            "role": "user",
            "content": f"Write a blog post draft about: {topic}\n\nOutline:\n{outline}\n\nWrite 800-1000 words."
        }],
        temperature=0.7,
        max_tokens=2000
    )
    draft = draft_resp.choices[0].message.content

    # Step 2: Polish on Claude (editing is cheaper than writing from scratch)
    # Feed 1000 tokens of draft vs generating 1000 tokens of original content
    # Editing uses ~30% fewer output tokens than generation
    return draft  # Pass this to Claude with "Polish this draft: ..."
```

## Batch Processing Script

For large batch jobs, use this rate-limit-aware processor:

```python
import time
from collections import deque

class GroqBatchProcessor:
    """Rate-limit-aware batch processor for Groq free tier."""

    def __init__(self, client, model="llama-3.1-8b-instant", rpm=28):
        self.client = client
        self.model = model
        self.rpm = rpm
        self.request_times = deque()

    def _wait_if_needed(self):
        now = time.time()
        # Remove requests older than 60 seconds
        while self.request_times and now - self.request_times[0] > 60:
            self.request_times.popleft()
        # If at limit, wait
        if len(self.request_times) >= self.rpm:
            sleep_time = 60 - (now - self.request_times[0]) + 0.5
            time.sleep(max(0, sleep_time))
        self.request_times.append(time.time())

    def process(self, prompt: str, max_tokens: int = 500) -> str:
        self._wait_if_needed()
        resp = self.client.chat.completions.create(
            model=self.model,
            messages=[{"role": "user", "content": prompt}],
            temperature=0.1,
            max_tokens=max_tokens
        )
        return resp.choices[0].message.content.strip()

    def process_batch(self, prompts: list[str], max_tokens: int = 500) -> list[str]:
        results = []
        for i, prompt in enumerate(prompts):
            result = self.process(prompt, max_tokens)
            results.append(result)
            if (i + 1) % 10 == 0:
                print(f"  Processed {i+1}/{len(prompts)}")
        return results
```

## When NOT to Use Groq

| Scenario | Why not | Use instead |
|---|---|---|
| Complex reasoning | Llama 8B can't match Claude | Sonnet or Opus |
| Code that must be correct | Higher error rate | Sonnet |
| Sensitive data | Data leaves your control | Claude (Anthropic ToS) |
| Real-time conversation | Different API format | Stay in Claude Code |
| Tasks requiring tool use | No MCP/tool support | Claude Code |

## Cost Savings Calculator

```
Monthly Claude spend:                    $400
Tasks offloadable to Groq:              40%
Groq cost for those tasks:              $0
Remaining Claude tasks:                  $240
Tasks routeable to Haiku:               30% of remaining
Haiku cost (vs Sonnet):                 $7 (was $72)
Final monthly cost:                      $175

Savings: $225/month (56%)
```
