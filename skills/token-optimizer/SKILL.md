---
name: token-optimizer
description: >
  Reduce Claude Code token consumption and API costs without sacrificing
  output quality. Use this skill whenever the user mentions high costs,
  expensive sessions, token waste, burning through credits, wanting to use
  Haiku instead of Opus, model routing, MAX_THINKING_TOKENS, extended thinking
  cost, CLAUDE_CODE_SUBAGENT_MODEL, smart model selection, /model switching,
  cost per session, prompt efficiency, reducing repetition in prompts, batching
  requests, caching strategies, or wanting to get the same results for less.
  Also trigger when reviewing CLAUDE.md or settings.json for cost optimization,
  or when the user wants to understand what is actually costing tokens.
---

# Token Optimizer Skill

## The Token Cost Breakdown

Understanding where tokens actually go in a typical Claude Code session:

| Source | Typical tokens | Controllable? |
|---|---|---|
| System tools (built-in) | ~16.8k | ❌ Fixed |
| System prompt | ~2.7k | ❌ Fixed |
| MCP tool descriptions | 200–500 per tool | ✅ Disable unused |
| CLAUDE.md / memory files | 1k–20k+ | ✅ Trim aggressively |
| Agent persona files | 200–1k each | ✅ Use functional roles |
| Skill descriptions | ~100 per skill | ✅ Keep library lean |
| Extended thinking (hidden) | Up to 31,999/request | ✅ Cap with env var |
| Subagent model choice | 5–15× cost difference | ✅ Route to Haiku |
| Conversation history | Grows until compaction | ✅ Compact strategically |

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

### CLAUDE_CODE_SUBAGENT_MODEL
Subagents (spawned via the Task tool) default to the same model as the main session.
- Set to `haiku` for cheap parallel work: file reading, grep, simple transforms
- Subagents doing complex reasoning? Override per-task to `sonnet`
- Typical savings: **60–80% reduction** on subagent costs

### CLAUDE_AUTOCOMPACT_PCT_OVERRIDE
Controls when auto-compaction fires (default ~83.5% = 167k tokens used).
- `70`: compacts earlier, more frequently, less data lost per event
- `50`: very aggressive — good for extremely long sessions
- Lower = more compaction events but each loses less precision

---

## Model Routing Strategy

Not every task needs Opus. Route by complexity:

| Task type | Model | Why |
|---|---|---|
| Complex architecture, novel problems | Opus | Needs full reasoning |
| Standard coding, refactoring, reviews | Sonnet | 60% cheaper, handles 80% of work |
| File reading, grep, simple transforms | Haiku | 90% cheaper, fast |
| Subagents doing routine tasks | Haiku | Bulk work, cost sensitive |

**Manual switching in session**:
```
/model sonnet   # switch for this session
/model opus     # escalate for hard problem
/model haiku    # downgrade for quick task
```

**Rule of thumb**: start on Sonnet, escalate to Opus only when stuck.

---

## Prompt Efficiency Patterns

### Use trigger tables instead of prose descriptions
❌ Expensive (loads verbosely):
```markdown
When you need to work with the database, you should use our custom DB
utility which wraps Supabase. This utility handles connection pooling,
error handling, and type safety. It is located at lib/db.ts and...
```

✅ Cheap (pointer pattern):
```markdown
DB work → see lib/db.ts
```

### Reference files by path instead of pasting content
❌ Expensive:
```
Here is the full content of our config file: [1000 lines pasted]
```

✅ Cheap:
```
Config is at .env.example — read it if needed
```

### Batch related requests in one message
❌ Expensive (3 separate messages = 3× system overhead):
```
Message 1: "Fix the lint errors"
Message 2: "Now add tests"
Message 3: "Update the README"
```

✅ Cheap (one message):
```
Fix lint errors, add tests for the changed functions, and update README. Do them in order.
```

### Use `/compact` before switching topics
Compacting at a clean boundary removes resolved thread history
and keeps the fresh context lean for the next task.

---

## CLAUDE.md Audit Checklist

Run this audit on your CLAUDE.md to find token waste:

- [ ] **Remove examples** — move to reference files, load on demand
- [ ] **Replace prose with tables** — same info, 60% fewer tokens
- [ ] **Delete outdated rules** — old instructions ghost-load every session
- [ ] **Use trigger tables** for skill routing instead of descriptions
- [ ] **Move verbose context** to `.claude/project-context.md`, import with `@`
- [ ] **Target**: CLAUDE.md under 100 lines / 1,500 tokens

### Before vs After example:
```markdown
# BEFORE (bloated — ~800 tokens)
## Authentication
We use Clerk for authentication. Clerk is a complete user management solution
that provides sign-in, sign-up, user profiles... [200 words]
Always make sure to check CLERK_WEBHOOK_SECRET environment variable...
The Clerk middleware should be in middleware.ts, not in individual routes...

# AFTER (lean — ~50 tokens)
## Stack
Auth: Clerk | DB: Supabase | Deploy: Vercel | Tests: Vitest
Auth details → see .claude/refs/auth.md
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
    "memory",        # if not using persistent memory
    "supabase",      # if not on a Supabase project
    "vercel",        # if not deploying
    "railway"        # if not using Railway
  ]
}
```

**Rule**: if you haven't used an MCP in the last 5 sessions, disable it.
Re-enable when needed — takes 30 seconds.

---

## Extended Thinking — When to Enable/Disable

Extended thinking burns tokens on internal reasoning that you never see.

| Task | Thinking tokens needed | Setting |
|---|---|---|
| Architectural decisions, novel algorithms | High | `20000–31999` |
| Standard feature implementation | Medium | `10000` |
| Refactoring, bug fixes, test writing | Low | `5000` |
| File reading, search, formatting | None | `0` |

Override for a specific session task:
```
Disable extended thinking for this task — it's straightforward refactoring.
```

---

## Cost Estimation Quick Reference

Approximate token costs per action (Sonnet pricing as baseline):

| Action | Approx tokens | Relative cost |
|---|---|---|
| Simple question + answer | 2–5k | 1× |
| Code review of a file | 5–15k | 3–7× |
| Implement a feature | 20–50k | 10–25× |
| Full codebase audit | 100–300k | 50–150× |
| Multi-file refactor with tests | 50–150k | 25–75× |

**Cost levers in order of impact**:
1. Model choice (Haiku vs Opus = 20× difference)
2. MAX_THINKING_TOKENS reduction (~70% hidden cost)
3. Subagent model routing
4. CLAUDE.md/MCP trimming
5. Strategic compaction timing
