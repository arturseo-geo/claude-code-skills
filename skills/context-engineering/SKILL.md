---
name: context-engineering
description: >
  Manage, optimize, and debug Claude Code context windows. Use this skill
  whenever the user mentions context limits, context window, autocompaction,
  compaction firing, context rot, losing context mid-session, session degradation,
  running out of tokens, /compact, CLAUDE_AUTOCOMPACT_PCT_OVERRIDE, context
  compression, masking, caching, progressive disclosure, CLAUDE.md being too
  large, skills eating tokens, tool descriptions bloating context, or wanting
  to extend how long a session can run without losing precision. Also trigger
  when designing agent architectures that need to maintain state across long
  tasks, when debugging why Claude forgot something earlier in a session, or
  when the user wants to understand how context budgeting works in Claude Code.
---

# Context Engineering Skill

## How Claude Code's Context Window Works

Claude Code's 200k context window is split across fixed overhead and your conversation:

| Slot | Typical size | Notes |
|---|---|---|
| System prompt | ~2.7k tokens | Fixed |
| System tools | ~16.8k tokens | Fixed — all built-in tools |
| Custom agents | ~1.3k tokens | Per agent file loaded |
| Memory files (CLAUDE.md etc.) | ~7.4k tokens | Grows with your config |
| Skills (metadata) | ~0.5–1k tokens | ~100 tokens per skill description |
| Messages (conversation) | Grows → triggers compaction | Your actual work |
| **Autocompact buffer** | **33k tokens** | Hardcoded — cannot be changed |

**Working limit**: compaction fires at ~167k tokens (200k minus 33k buffer).  
When it fires, your precise early context (variable names, exact errors, decisions) gets lossy-summarised.

---

## Autocompaction Control

```bash
# Default: fires at ~83.5% capacity
# Override to trigger earlier (more frequent, less data loss per event)
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=70

# Or later (longer sessions, more data lost when it fires)
export CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=90
```

Set in `.claude/settings.json` to persist:
```json
{
  "env": {
    "CLAUDE_AUTOCOMPACT_PCT_OVERRIDE": "70"
  }
}
```

**Strategic compaction**: run `/compact` manually at logical checkpoints
(end of a feature, after a big refactor) instead of waiting for auto-fire.
This gives you control over *what* gets summarised vs lost.

---

## Reducing Static Context Overhead

### CLAUDE.md diet
Every line in CLAUDE.md loads every session. Keep it under 100 lines.

❌ **Bad** — verbose explanation:
```markdown
## beads
Beads is a dependency-aware issue tracking system that integrates
with git and provides backlog management... [500 words]
```

✅ **Good** — trigger table only (skill loads the rest on demand):
```markdown
| Trigger words | Skill |
|---|---|
| task, track, backlog, blocked by, depends on | beads |
| deploy, release, ship | deploy |
```

### Skill description length
Each skill's description is loaded into every session (~100 tokens each).
- Keep descriptions focused on WHEN to trigger, not HOW it works
- The HOW lives in the SKILL.md body — loaded only when triggered
- 10 skills ≈ 1k tokens; 50 skills ≈ 5k tokens — keep your library lean

### MCP tool count
Each enabled MCP tool description consumes context.  
**Rule**: keep under 10 active MCPs and under 80 total tools.

```json
// .claude/settings.json — disable unused MCPs
{
  "disabledMcpServers": ["supabase", "railway", "vercel"]
}
```

---

## Context Compression Strategies

### 1. Compaction (built-in)
`/compact` — Claude summarises the conversation into a compressed representation.
Best used proactively at clean breakpoints.

### 2. Masking
Explicitly tell Claude to stop referencing certain content:
> "Ignore all prior discussion of the auth module — treat it as complete."

Reduces re-reading of resolved threads.

### 3. Caching via files
Instead of keeping large outputs in the conversation, write them to files:
> "Save the full analysis to `.claude/analysis.md` — we'll reference it by path."

File reads are on-demand; the content doesn't sit in the message thread.

### 4. Progressive disclosure
Structure reference material in layers:
```
SKILL.md (100 tokens — always loaded)
  └── references/detail.md (loaded only when needed)
        └── references/full-spec.md (loaded only for edge cases)
```

### 5. Recursive decomposition (for 100k+ token tasks)
Break large tasks into independent sub-tasks dispatched to subagents.
Each subagent gets a clean context with only what it needs.
```
Orchestrator (holds plan)
├── Subagent A: processes files 1–25 → writes output-a.md
├── Subagent B: processes files 26–50 → writes output-b.md
└── Orchestrator synthesises output-a.md + output-b.md
```

---

## Context Degradation Patterns to Avoid

| Pattern | Problem | Fix |
|---|---|---|
| Pasting large files inline | Eats conversation tokens | Write to file, reference by path |
| Tool results kept in thread | Accumulates fast | Summarise results before adding to messages |
| Re-explaining the same thing | Redundant tokens | Alias it ("the auth module from earlier") |
| Hooks firing on every prompt | Injects tokens each message | Use `Stop` hook instead of `UserPromptSubmit` |
| Verbose agent persona text | Overhead per subagent | Use functional roles, not character descriptions |
| Unused MCP servers enabled | Tool descriptions in every session | Disable in settings.json |

---

## Debugging Context Usage

To inspect current token usage, ask Claude Code:
```
/tokens
```

Or check breakdown format:
```
Model · X/200k tokens (Y%)
  System prompt:  Xk  (X%)
  System tools:   Xk  (X%)
  Custom agents:  Xk  (X%)
  Memory files:   Xk  (X%)
  Skills:         Xk  (X%)
  Messages:       Xk  (X%)  ← watch this grow
  Free space:     Xk  (X%)
  Autocompact buffer: 33k (16.5%)
```

**Warning signs**: Messages > 50k, Free space < 60k — consider `/compact` now.

---

## Session Architecture for Long Tasks

For multi-hour sessions or tasks spanning many files:
1. **Start**: Write a session brief to `.claude/sessions/YYYY-MM-DD.md`
2. **Checkpoints**: After each major milestone, append summary to session file
3. **On compaction**: Reference `.claude/sessions/YYYY-MM-DD.md` in your next message to restore precision
4. **New session**: Load session file with `@.claude/sessions/YYYY-MM-DD.md`

This preserves exact variable names, decisions, and error states that compaction would lose.
