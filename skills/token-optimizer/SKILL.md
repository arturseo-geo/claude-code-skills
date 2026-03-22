---
name: token-optimizer
description: >
  Reduce Claude Code token consumption and API costs without sacrificing
  output quality. Use this skill whenever the user mentions high costs,
  expensive sessions, token waste, burning through credits, wanting to use
  Haiku instead of Opus, model routing, MAX_THINKING_TOKENS, extended thinking
  cost, CLAUDE_CODE_SUBAGENT_MODEL, smart model selection, /model switching,
  cost per session, prompt efficiency, reducing repetition in prompts, batching
  requests, caching strategies, prompt caching, response caching, cache hits,
  cost-per-session analysis, budget alerts, or wanting to get the same results
  for less. Also trigger when reviewing CLAUDE.md or settings.json for cost
  optimization, or when the user wants to understand what is actually costing
  tokens.
---

# Token Optimizer Skill

## The Token Cost Breakdown

Understanding where tokens actually go in a typical Claude Code session:

| Source | Typical tokens | Controllable? |
|---|---|---|
| System tools (built-in) | ~16.8k | No — Fixed |
| System prompt | ~2.7k | No — Fixed |
| MCP tool descriptions | 200-500 per tool | Yes — Disable unused |
| CLAUDE.md / memory files | 1k-20k+ | Yes — Trim aggressively |
| Agent persona files | 200-1k each | Yes — Use functional roles |
| Skill descriptions | ~100 per skill | Yes — Keep library lean |
| Extended thinking (hidden) | Up to 31,999/request | Yes — Cap with env var |
| Prompt caching overhead | 25% write premium | Yes — Maximise cache hits |
| Subagent model choice | 5-15x cost difference | Yes — Route to Haiku |
| Conversation history | Grows until compaction | Yes — Compact strategically |

---

## Settings — The High-Impact Tuning Knobs

Add to `.claude/settings.json`:

```json
{
  "model": "sonnet",
  "env": {
    "MAX_THINKING_TOKENS": "10000",
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "70",
    "CLAUDE_CODE_SUBAGENT_MODEL": "haiku"
  }
}
```

### MAX_THINKING_TOKENS
Extended thinking reserves up to **31,999 tokens per request** for internal reasoning.
- Default: 31,999 (very expensive for simple tasks)
- `10000`: ~70% cost reduction on thinking, sufficient for most coding tasks
- `0`: disables extended thinking entirely (use for trivial/repetitive tasks)
- `20000`: good balance for complex architecture work
- See `references/thinking-tokens.md` for detailed tuning guidance

### CLAUDE_CODE_SUBAGENT_MODEL
Subagents (spawned via the Task tool) default to the same model as the main session.
- Set to `haiku` for cheap parallel work: file reading, grep, simple transforms
- Subagents doing complex reasoning? Override per-task to `sonnet`
- Never route subagents to Opus — the reasoning overhead is wasted on narrow tasks
- Typical savings: **60-80% reduction** on subagent costs

### CLAUDE_AUTOCOMPACT_PCT_OVERRIDE
Controls when auto-compaction fires (default ~83.5% = 167k tokens used).
- `70`: compacts earlier, more frequently, less data lost per event
- `50`: very aggressive — good for extremely long sessions
- Lower = more compaction events but each loses less precision

---

## Model Routing Strategy

Not every task needs Opus. Route by complexity:

| Task type | Model | Why | Cost relative to Opus |
|---|---|---|---|
| Complex architecture, novel problems | Opus | Full reasoning depth | 1x |
| Standard coding, refactoring, reviews | Sonnet | Handles 80% of work | ~0.2x |
| File reading, grep, simple transforms | Haiku | Fast, parallel-friendly | ~0.04x |
| Subagents doing routine tasks | Haiku | Bulk work, cost sensitive | ~0.04x |

**Manual switching in session**:
```
/model sonnet   # switch for this session
/model opus     # escalate for hard problem
/model haiku    # downgrade for quick task
```

**Rule of thumb**: start on Sonnet, escalate to Opus only when stuck or when the task involves novel architecture, cross-system reasoning, or ambiguous requirements.

**When Opus pays for itself**: multi-file refactors touching 10+ files, security audits, designing new system architectures, debugging subtle race conditions. The extra reasoning often completes the task in fewer turns, offsetting the per-token premium.

See `references/model-routing.md` for detailed cost comparison and decision framework.

---

## Prompt Efficiency Patterns

### Use trigger tables instead of prose descriptions
Bad (loads verbosely):
```markdown
When you need to work with the database, you should use our custom DB
utility which wraps Supabase. This utility handles connection pooling,
error handling, and type safety. It is located at lib/db.ts and...
```

Good (pointer pattern):
```markdown
DB work -> see lib/db.ts
```

### Reference files by path instead of pasting content
Bad:
```
Here is the full content of our config file: [1000 lines pasted]
```

Good:
```
Config is at .env.example -- read it if needed
```

### Batch related requests in one message
Bad (3 separate messages = 3x system overhead):
```
Message 1: "Fix the lint errors"
Message 2: "Now add tests"
Message 3: "Update the README"
```

Good (one message):
```
Fix lint errors, add tests for the changed functions, and update README. Do them in order.
```

### Use `/compact` before switching topics
Compacting at a clean boundary removes resolved thread history
and keeps the fresh context lean for the next task.

### Eliminate redundant instructions
Review your CLAUDE.md for instructions that repeat default Claude Code behaviour.
Claude already knows how to write tests, use git, and format code — only add
rules that override defaults or encode project-specific knowledge.

---

## Caching Strategies

### Prompt caching mechanics
Anthropic caches prompt prefixes automatically. Tokens served from cache cost
**90% less** than fresh input tokens, but the initial write to cache costs
**25% more**. The breakpoint: if a prompt prefix is reused 2+ times, caching
saves money.

### Maximise cache hits
- Keep CLAUDE.md and skill files **stable** — every edit invalidates the cache
  for that prefix
- Front-load static context (project rules, style guides) before dynamic
  context (conversation history, file contents)
- Avoid unnecessary whitespace changes in instruction files
- Batch similar tasks in the same session — the shared prefix stays cached

### Response reuse
- For repeated operations (e.g., reviewing 20 files with the same checklist),
  ask Claude to produce a reusable template in the first pass, then apply it
- Use `/compact` between batches to keep the prefix stable while clearing
  variable content

See `references/caching.md` for detailed mechanics and optimisation patterns.

---

## CLAUDE.md Audit Checklist

Run this audit on your CLAUDE.md to find token waste:

- [ ] **Remove examples** — move to reference files, load on demand
- [ ] **Replace prose with tables** — same info, 60% fewer tokens
- [ ] **Delete outdated rules** — old instructions ghost-load every session
- [ ] **Use trigger tables** for skill routing instead of descriptions
- [ ] **Move verbose context** to `.claude/project-context.md`, import with `@`
- [ ] **Remove default behaviour restating** — do not tell Claude to "write clean code"
- [ ] **Target**: CLAUDE.md under 100 lines / 1,500 tokens

### Before vs After example:
```markdown
# BEFORE (bloated -- ~800 tokens)
## Authentication
We use Clerk for authentication. Clerk is a complete user management solution
that provides sign-in, sign-up, user profiles... [200 words]
Always make sure to check CLERK_WEBHOOK_SECRET environment variable...
The Clerk middleware should be in middleware.ts, not in individual routes...

# AFTER (lean -- ~50 tokens)
## Stack
Auth: Clerk | DB: Supabase | Deploy: Vercel | Tests: Vitest
Auth details -> see .claude/refs/auth.md
```

---

## MCP Token Audit

Each MCP tool description consumes tokens from your context window on every session.

```bash
# Check how many tools are active
# In Claude Code: /mcp

# Disable unused servers in .claude/settings.json
{
  "disabledMcpServers": [
    "memory",
    "supabase",
    "vercel",
    "railway"
  ]
}
```

**Rule**: if you haven't used an MCP in the last 5 sessions, disable it.
Re-enable when needed — takes 30 seconds.

**Quantify the cost**: each MCP server adds 200-500 tokens per tool. A server
with 10 tools = 2,000-5,000 tokens loaded every message. Multiply by messages
per session to see the true cost.

---

## Extended Thinking — When to Enable/Disable

Extended thinking burns tokens on internal reasoning that you never see.

| Task | Thinking tokens needed | Setting |
|---|---|---|
| Architectural decisions, novel algorithms | High | `20000-31999` |
| Standard feature implementation | Medium | `10000` |
| Refactoring, bug fixes, test writing | Low | `5000` |
| File reading, search, formatting | None | `0` |

Override for a specific session task:
```
Disable extended thinking for this task -- it's straightforward refactoring.
```

See `references/thinking-tokens.md` for detailed analysis of when thinking
helps vs hurts and how to tune the budget.

---

## Batch Request Optimisation

### When to batch
- Processing multiple files with the same operation
- Running the same check across a codebase
- Generating similar outputs (tests, docs, translations)

### How to batch effectively
1. **Group by operation type** — all lint fixes together, all test additions together
2. **Provide the pattern once** — "For each file in src/components/, add a test
   file following the pattern in tests/Button.test.ts"
3. **Use subagents for parallelism** — with `CLAUDE_CODE_SUBAGENT_MODEL=haiku`,
   parallel subagents handle bulk work at minimal cost
4. **Set explicit completion criteria** — "Done when every component has a
   corresponding test file"

### Avoid these batch anti-patterns
- Sending one file at a time in separate messages (N x system overhead)
- Asking for confirmation between each item (breaks the batch)
- Mixing unrelated tasks in one batch (confuses prioritisation)

---

## Cost-Per-Session Analysis

### Track what you spend
After each session, check the cost summary displayed by Claude Code. Key metrics:
- **Total input tokens**: dominated by system prompt + CLAUDE.md + conversation
- **Total output tokens**: your actual useful output
- **Cache read vs cache write**: high read ratio = good caching
- **Thinking tokens**: often the largest hidden cost

### Budget rules of thumb
| Session type | Typical cost (Sonnet) | With optimisation |
|---|---|---|
| Quick question | $0.01-0.05 | $0.01-0.02 |
| Single feature | $0.10-0.50 | $0.05-0.20 |
| Multi-file refactor | $0.50-2.00 | $0.20-0.80 |
| Full codebase audit | $2.00-10.00 | $1.00-4.00 |
| Long session (2+ hours) | $5.00-20.00 | $2.00-8.00 |

### Cost reduction checklist (ordered by impact)
1. Switch model: Opus -> Sonnet = ~80% reduction
2. Cap thinking: `MAX_THINKING_TOKENS=10000` = ~70% thinking reduction
3. Route subagents: `CLAUDE_CODE_SUBAGENT_MODEL=haiku` = ~80% subagent reduction
4. Trim CLAUDE.md: remove 1,000 tokens = saves that per message
5. Disable unused MCPs: each server = 2-5k tokens per message saved
6. Compact earlier: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70`
7. Batch requests: 3 tasks in 1 message vs 3 messages = ~60% overhead reduction

See `references/cost-tracking.md` for project-level tracking and budget alerts.

---

## Groq Offloading — Free/Near-Free Processing

Groq provides **free-tier access** to Llama 3.1 8B and other open models with
extremely fast inference. Offload bulk processing tasks to Groq to avoid burning
Anthropic tokens entirely.

### When to offload to Groq

| Task | Groq model | Why not Claude |
|---|---|---|
| Keyword clustering | llama-3.1-8b | Bulk classification, no reasoning needed |
| Text summarisation (bulk) | llama-3.1-8b | Summarise 100 articles = massive token cost on Claude |
| JSON/data transformation | llama-3.1-8b | Reformatting, no intelligence needed |
| Memory compaction/scoring | llama-3.1-8b | Score 500 memories offline, feed top-N to Claude |
| Content tagging/classification | llama-3.1-8b | Label 200 posts by category |
| Embeddings generation | — | Use dedicated embedding APIs instead |
| SEO meta description generation (bulk) | llama-3.1-70b | Higher quality for content, still free tier |
| Draft generation for review | llama-3.1-70b | Generate drafts on Groq, review/polish on Claude |

### Groq integration pattern

```python
import os
from groq import Groq

client = Groq(api_key=os.environ.get("GROQ_API_KEY"))

def offload_to_groq(prompt: str, model: str = "llama-3.1-8b-instant") -> str:
    """Offload a task to Groq's free tier instead of using Claude tokens."""
    response = client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
        temperature=0.1,
        max_tokens=2048
    )
    return response.choices[0].message.content

# Example: bulk classify 100 items for $0 instead of $2+ on Claude
categories = []
for item in items:
    cat = offload_to_groq(f"Classify this into one category (tech/business/science): {item['title']}")
    categories.append(cat)
```

### Groq free tier limits (2026)
- **llama-3.1-8b-instant**: 30 RPM, 14,400 requests/day, 131,072 tokens/min
- **llama-3.1-70b-versatile**: 30 RPM, 14,400 requests/day, 131,072 tokens/min
- **No cost** — completely free for these limits
- Rate limited, not usage limited — spread requests over time for bulk work

### Cost comparison: Claude vs Groq offloading

| Task (100 items) | Claude Sonnet cost | Groq cost | Savings |
|---|---|---|---|
| Classify 100 articles | ~$0.60 | $0.00 | 100% |
| Summarise 100 pages | ~$3.00 | $0.00 | 100% |
| Generate 100 meta descriptions | ~$1.50 | $0.00 | 100% |
| Score 500 memories | ~$2.00 | $0.00 | 100% |
| Cluster 200 keywords | ~$1.00 | $0.00 | 100% |

### Setup
```bash
# Get free API key at console.groq.com
pip install groq

# Add to environment
export GROQ_API_KEY="gsk_..."
```

See `references/groq-offloading.md` for complete integration patterns and batch scripts.

---

## Haiku as Default — The 80/20 Rule

Most Claude Code users leave the model on Opus or Sonnet for everything.
**Switch the default to Haiku** and escalate only when needed:

### Haiku-first workflow
```
1. Start session on Haiku (/model haiku)
2. Do file reading, search, simple edits, boilerplate — Haiku handles all of this
3. Hit a complex problem? /model sonnet — solve it
4. Back to routine work? /model haiku
5. Stuck on architecture? /model opus — for that one decision
6. Back to /model haiku for implementation
```

### What Haiku handles well (at 4% of Opus cost)
- Reading and summarising files
- Simple code edits (rename, add parameter, fix typo)
- Running grep/search and reporting results
- Generating boilerplate (tests, interfaces, types)
- Formatting and linting fixes
- Git operations (commit messages, status checks)
- Simple Q&A about code

### What needs escalation
- Multi-file refactors touching complex logic → Sonnet
- Architectural decisions → Opus
- Debugging subtle race conditions → Opus
- Security review → Opus
- Ambiguous requirements → Sonnet or Opus

### Combined Haiku + Groq strategy

For maximum savings, split work three ways:

| Work type | Route to | Cost |
|---|---|---|
| Bulk processing (classify, summarise, tag) | **Groq** (free) | $0.00 |
| Routine Claude Code work (read, edit, search) | **Haiku** | ~$0.001/task |
| Standard development (features, tests, reviews) | **Sonnet** | ~$0.01-0.10/task |
| Complex reasoning (architecture, debugging) | **Opus** | ~$0.10-1.00/task |

**Realistic daily cost** with this routing:
- Before: $15-20/day (Opus/Sonnet for everything)
- After: $3-8/day (Groq + Haiku + Sonnet, Opus only when stuck)
- **Savings: 50-80%**

---

## Cost Estimation Quick Reference

Approximate token costs per action (Sonnet pricing as baseline):

| Action | Approx tokens | Relative cost |
|---|---|---|
| Simple question + answer | 2-5k | 1x |
| Code review of a file | 5-15k | 3-7x |
| Implement a feature | 20-50k | 10-25x |
| Full codebase audit | 100-300k | 50-150x |
| Multi-file refactor with tests | 50-150k | 25-75x |

**Cost levers in order of impact**:
1. Model choice (Haiku vs Opus = 20x difference)
2. MAX_THINKING_TOKENS reduction (~70% hidden cost)
3. Subagent model routing
4. Prompt caching optimisation
5. CLAUDE.md/MCP trimming
6. Strategic compaction timing
7. Batch request consolidation
