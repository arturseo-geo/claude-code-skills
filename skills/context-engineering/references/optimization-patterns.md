# Context Optimization Patterns

Proven patterns for extending session life and maintaining output quality throughout long Claude Code sessions.

## Pattern 1: Chunked Processing

**Problem:** A task requires processing 50+ files, but loading them all at once exhausts the context window.

**Solution:** Process in batches, writing intermediate results to disk.

```
Step 1: List all files matching src/**/*.ts
Step 2: Process files 1-10, write findings to .claude/analysis/batch-1.md
Step 3: /compact
Step 4: Process files 11-20, write findings to .claude/analysis/batch-2.md
...
Final: Read all batch files, synthesize into final report
```

**Key insight:** Each batch gets a fresh-ish context. The compaction between batches clears the processing details while the disk files preserve the results.

## Pattern 2: Progressive Disclosure in Skills

**Problem:** A skill needs extensive reference material, but loading it all wastes tokens when only a subset is needed.

**Solution:** Layer your skill content:

```
SKILL.md              → 100–200 tokens (triggers + decision tree)
  ├── references/overview.md    → 500 tokens (common patterns)
  ├── references/advanced.md    → 1000 tokens (edge cases)
  └── references/full-spec.md   → 2000 tokens (complete reference)
```

In SKILL.md, include conditional loading instructions:
```markdown
If the user asks about basic usage, the content above is sufficient.
If they need advanced patterns, read references/advanced.md.
If they need the full specification, read references/full-spec.md.
```

## Pattern 3: Session Checkpointing

**Problem:** Long sessions lose precision after compaction.

**Solution:** Write state to disk at natural breakpoints.

```markdown
<!-- .claude/sessions/2026-03-22-refactor.md -->
## Session: Auth Refactor

### Decisions Made
- Using JWT with rotating refresh tokens (not sessions)
- Middleware in src/middleware/auth.ts
- Token TTL: access=15min, refresh=7d

### Current State
- [x] Middleware implemented
- [x] Tests passing (47/47)
- [ ] Migration script for existing sessions
- [ ] Rate limiting on token refresh endpoint

### Key File Paths
- src/middleware/auth.ts (new)
- src/routes/auth.ts (modified)
- tests/auth.test.ts (modified)
- prisma/migrations/20260322_auth_refactor/ (new)
```

After compaction, reference the file: `@.claude/sessions/2026-03-22-refactor.md`

## Pattern 4: Context Pruning via Masking

**Problem:** Earlier parts of the conversation contain resolved discussions that Claude keeps re-reading.

**Solution:** Explicitly close topics:

```
The database migration is complete and working. Do not reference our
earlier discussion about migration strategies — treat it as resolved.
Let's move on to the API endpoints.
```

This signals Claude to deprioritize that content during generation, effectively freeing cognitive bandwidth.

## Pattern 5: File-Based Caching

**Problem:** Repeated tool calls return the same large output, bloating the conversation.

**Solution:** Cache results to files on first retrieval:

```
Read src/config/schema.ts and save the type definitions to
.claude/cache/schema-types.md. We'll reference that file
instead of re-reading the source each time.
```

Benefits:
- File reads are on-demand (not in message history)
- The cached version can be a trimmed extract (only the relevant parts)
- Multiple sessions can reuse the same cache

## Pattern 6: Subagent Decomposition

**Problem:** A task requires reasoning over more content than fits in one context window.

**Solution:** Split into independent subagent tasks:

```
Orchestrator (plan + synthesis)
├── Agent A: Analyze authentication module → auth-analysis.md
├── Agent B: Analyze database layer → db-analysis.md
├── Agent C: Analyze API routes → api-analysis.md
└── Orchestrator: Read all analysis files, produce unified report
```

Each subagent gets a clean 200k context. The orchestrator only needs the compressed outputs.

**When to use subagents:**
- Total input > 100k tokens
- Task is naturally parallelizable
- Sub-tasks don't depend on each other's intermediate steps

## Pattern 7: CLAUDE.md as a Router

**Problem:** CLAUDE.md is large because it contains detailed instructions for many scenarios.

**Solution:** Turn CLAUDE.md into a routing table that points to detailed files:

```markdown
<!-- CLAUDE.md — kept under 100 lines -->
## Project: myapp

Stack: Next.js 15, Prisma, PostgreSQL, Tailwind

## Skill Triggers
| Keywords | Skill |
|---|---|
| deploy, release, CI | deploy |
| test, coverage, jest | testing |
| schema, migration, database | database |
| auth, login, JWT, session | auth |

## Conventions
- Use `pnpm`, never `npm` or `yarn`
- All API routes in src/app/api/
- Database changes require migration + seed update
```

The detailed "how" for each area lives in the respective skill or reference file, loaded only when triggered.

## Pattern 8: Warm-Start Sessions

**Problem:** Starting a new session requires re-establishing all context from scratch.

**Solution:** Maintain a session brief that can bootstrap a new session:

```bash
# At end of session
Save current state to .claude/sessions/latest.md including:
- What we accomplished
- What's still pending
- Key decisions and their rationale
- Files modified and their purpose

# At start of next session
@.claude/sessions/latest.md — continuing from where we left off
```

## Token Budget Rules of Thumb

| Component | Target | Warning |
|---|---|---|
| CLAUDE.md | < 500 tokens | > 1000 tokens — trim or move to skills |
| Each skill description | < 100 tokens | > 200 tokens — too verbose |
| Total skill metadata | < 2k tokens | > 5k tokens — too many skills |
| MCP tool descriptions | < 5k tokens | > 10k tokens — disable unused servers |
| Free space after setup | > 150k tokens | < 130k tokens — overhead too high |
