# Debugging Context Issues

How to diagnose and fix common context window problems in Claude Code.

## Symptom: Claude "Forgot" Something

### Diagnosis

1. Check if compaction fired — look for the compaction indicator in the conversation
2. Run `/tokens` to see current usage breakdown
3. Ask Claude to summarize what it knows about the topic — if it gives a vague or incorrect summary, the details were lost in compaction

### Fixes

**If compaction already fired:**
- Re-inject the critical information: "For context, the bug was in `src/utils/parse.ts` line 42 — a null check on the `options` parameter"
- Reference a session file if you created one: `@.claude/sessions/today.md`

**Prevention:**
- Write critical decisions to files before compaction
- Use `/compact` manually at clean breakpoints instead of waiting for auto-fire
- Lower `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` to compact more frequently with less data loss

## Symptom: Output Quality Degrades Over Time

### Diagnosis

The model's responses become less precise, more generic, or start contradicting earlier decisions. This happens when:

1. The context is full of conflicting information from iterative attempts
2. Compaction has happened multiple times, compounding summarization loss
3. The conversation contains too many unresolved threads

### Fixes

- **Start fresh with a session file:** Create a checkpoint, then start a new session loading only the checkpoint
- **Prune dead ends:** "Ignore all discussion of approaches A and B — we're going with approach C"
- **Compact with focus:** `/compact focus on the current implementation approach and remaining tasks`

## Symptom: Unexpected Early Compaction

### Diagnosis

Compaction fires much sooner than expected. Check for context bloat sources:

```
/tokens
```

Look at the breakdown:
- **System tools > 20k** — Too many MCP tools enabled
- **Memory files > 10k** — CLAUDE.md or other memory files are too large
- **Custom agents > 5k** — Agent definitions are verbose
- **Skills > 3k** — Too many skills installed or descriptions are too long

### Fixes

| Bloat source | Action |
|---|---|
| MCP tools | Disable unused servers in `.claude/settings.json` → `disabledMcpServers` |
| Memory files | Trim CLAUDE.md to < 100 lines, move detail to skills |
| Custom agents | Shorten agent descriptions, use functional roles |
| Skills | Remove unused skills, shorten `description` fields in frontmatter |

## Symptom: Tool Results Consuming Too Much Context

### Diagnosis

Each tool call (file reads, grep results, bash output) adds its full output to the conversation history. Large outputs accumulate fast.

### Fixes

- **Limit reads:** Use `offset` and `limit` parameters when reading files — only read the section you need
- **Summarize results:** After getting a large tool result, tell Claude to extract only what matters
- **Write to files:** For large analysis outputs, write to disk instead of keeping in conversation
- **Use specific searches:** Narrow grep patterns and glob filters to reduce result size

## Symptom: Skills Not Triggering

### Diagnosis

A skill should activate based on conversation keywords but doesn't fire.

1. Check the skill's `description` field in SKILL.md frontmatter — does it list the relevant trigger words?
2. Verify the skill is installed: `ls ~/.claude/skills/`
3. Check that the skill directory contains a valid SKILL.md with proper frontmatter

### Fixes

- Add missing trigger words to the skill's `description` field
- Ensure the SKILL.md frontmatter uses the `---` delimiters correctly
- Reinstall the skill if the directory structure is wrong

## Symptom: Session Dies Unexpectedly

### Diagnosis

The session ends or becomes unresponsive, often after a very long conversation.

### Fixes

- **Proactive checkpointing:** Write state to disk every 30–60 minutes of active work
- **Lower compaction threshold:** `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE=65` gives more headroom
- **Break into sub-sessions:** For multi-hour tasks, plan natural break points where you can start a new session with a checkpoint file

## Diagnostic Checklist

When something feels wrong with context behavior, run through this:

1. **Check tokens:** `/tokens` — what's the usage breakdown?
2. **Check compaction history:** Has compaction fired? How many times?
3. **Check static overhead:** System tools + memory + skills + agents — is it > 30k?
4. **Check conversation bloat:** Are there large tool results sitting in the history?
5. **Check for conflicts:** Are there contradictory instructions in the conversation?
6. **Test retention:** Ask Claude to summarize key details — what's missing?

## Recovery Workflow

When a session has degraded beyond repair:

```
1. Ask Claude to write a complete session state to .claude/sessions/recovery.md
   Include: decisions, current task, pending work, key file paths, known issues

2. Start a new Claude Code session

3. Load the recovery file:
   @.claude/sessions/recovery.md
   "Continue from where we left off"

4. Verify: "What do you know about our current task?"
```

This gives you a fresh 200k context with only the essential state loaded.
