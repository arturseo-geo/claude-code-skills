# CLAUDE.md as Memory

CLAUDE.md is Claude Code's native memory mechanism. It injects into every session
automatically, requiring zero setup. This reference covers how to structure it
effectively as a persistent knowledge store.

## File Locations and Load Order

Claude Code loads CLAUDE.md files from three locations, in order:

| Location | Scope | When to use |
|---|---|---|
| `~/.claude/CLAUDE.md` | Global (all projects) | Personal preferences, global tool config |
| `CLAUDE.md` (project root) | Project (shared with team) | Stack, conventions, test commands |
| `.claude/CLAUDE.md` | Project (private) | Personal notes about this project |

All three merge into a single context block at session start. If the same fact
appears in multiple files, the most specific one (project > global) takes precedence
in practice because it loads last.

## Size Budget

CLAUDE.md content loads every session and persists in context for the entire
conversation. Every line costs tokens permanently.

| Size | Impact | Recommendation |
|---|---|---|
| < 50 lines | Negligible | Free to use |
| 50 - 150 lines | Acceptable | Keep it focused |
| 150 - 300 lines | Costly | Move details to references |
| 300+ lines | Wasteful | Refactor immediately |

**Target**: under 150 lines total across all three CLAUDE.md files.

## Recommended Structure

```markdown
# Project: [name]

## Stack
- Framework: Next.js 14 (App Router)
- Database: Supabase (PostgreSQL)
- Auth: Clerk
- Styling: Tailwind CSS
- Hosting: Vercel

## Commands
- Test: `pnpm test:unit`
- Lint: `pnpm lint`
- Dev: `pnpm dev`
- Build: `pnpm build`

## Rules
- Never commit directly to main
- All API routes in /app/api (never /pages/api)
- Use Zod for validation (not Yup)
- Error handling: see /lib/errors.ts

## Key Decisions
- [2026-03-15] Chose Clerk over NextAuth for auth — managed solution, less maintenance
- [2026-03-18] Using server components by default — client components only when needed

## Memory Index
- [Architecture](references/architecture.md)
- [API docs](references/api.md)
- [Decisions log](references/decisions.md)
```

## Router Pattern

When your project knowledge exceeds 150 lines, use CLAUDE.md as a router — a
table of contents that points to detail files.

```markdown
## Memory Index
- [Architecture](references/architecture.md) — system design, service map, data flow
- [Conventions](references/conventions.md) — naming, file structure, anti-patterns
- [API Reference](references/api.md) — endpoints, auth headers, rate limits
- [Decisions](references/decisions.md) — architectural decision records
- [Troubleshooting](references/troubleshooting.md) — known issues and fixes
```

Claude loads only the index (~5 lines). When it needs details, it reads the
specific file with `@references/architecture.md`.

**Benefits**:
- CLAUDE.md stays small (< 50 lines)
- Details load on demand, not every session
- Each reference file can be as long as needed
- Easy to organize by topic

## Auto-Injection vs. Manual Loading

| Content | Loading method | Why |
|---|---|---|
| Stack, commands, rules | Auto (CLAUDE.md) | Needed every session |
| Architecture diagrams | Manual (@-import) | Only when relevant |
| API reference | Manual (@-import) | Only when working on API |
| Session files | Manual (@-import) | Only when resuming |
| Memory JSONL | Hook (auto) or manual | Depends on workflow |

## Common Mistakes

| Mistake | Impact | Fix |
|---|---|---|
| Dumping entire API docs | Wastes 500+ tokens every session | Move to reference file |
| Duplicating between levels | Same fact loaded twice | One location per fact |
| Stale decisions | Claude follows outdated rules | Date your decisions, review monthly |
| No structure (wall of text) | Claude misses important facts | Use headers and bullet points |
| Secrets in CLAUDE.md | Committed to git | Use .env files, never CLAUDE.md |

## Team vs. Personal Memory

For team projects, split memory between shared and personal:

```
CLAUDE.md              # Shared: stack, commands, conventions (committed to git)
.claude/CLAUDE.md      # Personal: my preferences, my notes (gitignored)
~/.claude/CLAUDE.md    # Global: applies to all my projects
```

Add `.claude/CLAUDE.md` to `.gitignore` so personal notes stay private.
