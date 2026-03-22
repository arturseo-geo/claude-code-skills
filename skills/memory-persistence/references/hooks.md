# Hooks for Memory Automation

Claude Code hooks let you run scripts before, during, or after sessions. This
reference covers how to use hooks for automatic memory persistence.

## Hook Types for Memory

| Hook | When it runs | Memory use case |
|---|---|---|
| `Stop` | Session ends | Save learnings to JSONL |
| `PreToolUse` | Before a tool call | Load relevant memory before edits |
| `PostToolUse` | After a tool call | Record outcomes of commands |
| `UserPromptSubmit` | Every user message | Load context (use sparingly) |

**Preferred hook for saving**: `Stop` — runs once per session, no latency impact.
Avoid `UserPromptSubmit` for saving — it runs on every message and adds latency.

## Stop Hook — Auto-Save Learnings

The most important memory hook. At session end, extracts what Claude learned and
appends to persistent storage.

### Basic Implementation

**`.claude/hooks/stop-memory.sh`**:
```bash
#!/bin/bash
# Auto-save session learnings at session end
# Runs as a Stop hook — once per session, no latency impact

MEMORY_FILE=".claude/memory.jsonl"
TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%SZ)

# Ensure memory file exists with schema
if [ ! -f "$MEMORY_FILE" ]; then
    echo '{"schema": "v1", "fields": ["ts", "type", "key", "value", "context", "importance"]}' > "$MEMORY_FILE"
fi

# Ask Claude to extract and append learnings
claude -p "Review this session. Extract any:
- Debugging patterns discovered
- Project-specific facts learned
- User preferences observed
- Architectural decisions made
- Errors encountered and their fixes

For each learning, output exactly one JSON line (no other text):
{\"ts\": \"$TIMESTAMP\", \"type\": \"[type]\", \"key\": \"[short_key]\", \"value\": \"[what to remember]\", \"context\": \"[brief context]\", \"importance\": [0.0-1.0]}

Types: fact, decision, pattern, preference, error, relationship
Importance: 1.0 = critical, 0.7 = useful, 0.4 = minor
If nothing worth saving, output nothing.

Append output to $MEMORY_FILE — do not overwrite the file."
```

### Registration

**`.claude/settings.json`**:
```json
{
  "hooks": {
    "Stop": [
      {
        "command": "bash .claude/hooks/stop-memory.sh",
        "timeout": 30000
      }
    ]
  }
}
```

### Advanced: Session Summary + JSONL

Save both a human-readable session file and machine-readable JSONL entries:

```bash
#!/bin/bash
# .claude/hooks/stop-full.sh
MEMORY_FILE=".claude/memory.jsonl"
SESSION_DIR=".claude/sessions"
TODAY=$(date +%Y-%m-%d)
TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%SZ)

mkdir -p "$SESSION_DIR"

# 1. Write session summary
claude -p "Summarise this session in markdown:
- Goal: what we set out to do
- State: what's working, what's not
- Files changed: path and what changed
- Decisions: what we decided and why
- Next steps: ordered list
- Blockers: open questions

Save to $SESSION_DIR/$TODAY.md"

# 2. Append JSONL learnings
claude -p "Extract learnings from this session as JSON lines.
Format: {\"ts\": \"$TIMESTAMP\", \"type\": \"[type]\", \"key\": \"[key]\", \"value\": \"[value]\", \"context\": \"[context]\", \"importance\": [0-1]}
Append to $MEMORY_FILE"
```

---

## PreToolUse Hook — Load Context Before Edits

Surface relevant memories before Claude edits files, so it has context for decisions.

**`.claude/hooks/load-memory.sh`**:
```bash
#!/bin/bash
# Load recent memories before file edits
# Runs as PreToolUse hook with matcher: Edit|Write

MEMORY_FILE=".claude/memory.jsonl"

if [ -f "$MEMORY_FILE" ]; then
    echo "=== Recent project memories ==="
    tail -20 "$MEMORY_FILE" 2>/dev/null
    echo "=== End memories ==="
fi
```

**Registration**:
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "bash .claude/hooks/load-memory.sh",
        "timeout": 5000
      }
    ]
  }
}
```

**Warning**: keep PreToolUse hooks fast (< 1s). They run on every matching tool
call and add latency to the user experience.

---

## PostToolUse Hook — Record Outcomes

After a Bash command runs, record whether it succeeded or failed — useful for
building a troubleshooting memory.

**`.claude/hooks/record-outcome.sh`**:
```bash
#!/bin/bash
# Record command outcomes for future debugging
# Input: tool result from stdin

MEMORY_FILE=".claude/memory.jsonl"
TIMESTAMP=$(date -u +%Y-%m-%dT%H:%M:%SZ)

# Read the tool result
RESULT=$(cat)

# Only record failures
if echo "$RESULT" | grep -q "error\|Error\|FAILED\|failed"; then
    echo "{\"ts\": \"$TIMESTAMP\", \"type\": \"error\", \"key\": \"cmd_failure\", \"value\": \"$(echo $RESULT | head -c 200)\", \"context\": \"auto-captured by PostToolUse hook\", \"importance\": 0.6}" >> "$MEMORY_FILE"
fi
```

---

## Hook Timing and Performance

| Hook | Frequency | Max time | Impact |
|---|---|---|---|
| Stop | Once per session | 30s OK | None (session is ending) |
| PreToolUse | Every matching tool call | < 1s | Adds latency |
| PostToolUse | Every matching tool call | < 2s | Adds latency |
| UserPromptSubmit | Every user message | < 500ms | High latency risk |

**Rule of thumb**: save in Stop hooks (no latency cost), load in PreToolUse hooks
(keep fast), avoid UserPromptSubmit for anything slow.

---

## Debugging Hooks

If a hook is not running:

1. Check `.claude/settings.json` syntax — invalid JSON silently disables all hooks
2. Verify the script is executable: `chmod +x .claude/hooks/stop-memory.sh`
3. Test the script manually: `bash .claude/hooks/stop-memory.sh`
4. Check the matcher regex matches the tool name exactly
5. Look for timeout — default is 10s, increase with `"timeout": 30000`
