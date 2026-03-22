# Token Budget Allocation Strategies

How to plan and manage token budgets across CLAUDE.md, skills, tools, and conversations.

## The Context Budget

Think of the 200k context window as a budget with fixed and variable expenses:

```
┌─────────────────────────────────────────────────────────┐
│ 200k total context window                               │
├─────────────────────────────────────────────────────────┤
│ Fixed overhead          │  ~20k tokens                  │
│  - System prompt        │   2.7k                        │
│  - Built-in tools       │  16.8k                        │
├─────────────────────────────────────────────────────────┤
│ Configurable overhead   │  Varies (target: < 10k)       │
│  - CLAUDE.md + memory   │   target < 2k                 │
│  - Skill metadata       │   target < 1k                 │
│  - MCP tool descriptions│   target < 5k                 │
│  - Custom agents        │   target < 2k                 │
├─────────────────────────────────────────────────────────┤
│ Autocompact buffer      │  33k tokens (fixed)           │
├─────────────────────────────────────────────────────────┤
│ Working space           │  ~137k tokens (at defaults)   │
│  - Conversation         │   grows with each exchange    │
│  - Tool results         │   grows with each tool call   │
│  - Skill bodies         │   loaded on demand            │
└─────────────────────────────────────────────────────────┘
```

## Budget Targets by Component

### CLAUDE.md — Target: < 500 tokens (~100 lines)

CLAUDE.md loads in **every session**. Every token here reduces your working space permanently.

**What belongs in CLAUDE.md:**
- Project name and stack (1 line)
- Package manager and key commands (2-3 lines)
- Critical conventions that apply to ALL tasks (5-10 lines)
- Skill trigger table (routing keywords to skills)

**What does NOT belong in CLAUDE.md:**
- Detailed API documentation (move to a skill or reference file)
- Step-by-step workflows (move to a skill)
- Architecture explanations (move to a reference file)
- Historical decisions (move to a session log)

### Skill Descriptions — Target: < 100 tokens each

The skill `description` field in SKILL.md frontmatter loads in every session. It exists solely to help Claude decide WHEN to activate the skill.

**Good description (72 tokens):**
```yaml
description: >
  Deploy applications to production. Trigger when the user mentions
  deploy, release, ship, CI/CD, staging, production, or rollback.
```

**Bad description (300+ tokens):**
```yaml
description: >
  This skill handles all deployment operations including building
  Docker containers, pushing to ECR, updating ECS task definitions,
  running database migrations, invalidating CloudFront caches,
  monitoring health checks, and rolling back failed deployments.
  It supports blue-green deployments, canary releases, and...
```

The detailed capabilities belong in the SKILL.md body, not the description.

### MCP Tool Descriptions — Target: < 5k tokens total

Each MCP server registers tool descriptions that consume context. A server with 20 tools might add 2-3k tokens.

**Audit your MCP overhead:**
1. Run `/tokens` and note the system tools number
2. Subtract ~16.8k for built-in tools
3. The remainder is MCP overhead

**Reduction strategies:**
- Disable servers you rarely use: `disabledMcpServers` in settings
- If a server has 30 tools and you use 3, consider a lighter alternative
- Some MCP servers support tool filtering — enable only the tools you need

### Custom Agents — Target: < 2k tokens total

Agent definitions (`.claude/agents/*.md`) load their descriptions into context.

- Keep agent system prompts focused on role and constraints
- Avoid verbose persona descriptions
- Move detailed reference material to files the agent can read on demand

## Budget Planning for Different Workflows

### Solo Developer — Light Setup

```
CLAUDE.md:         200 tokens
Skills (3):        300 tokens
MCP tools (1):     500 tokens
Agents (0):          0 tokens
─────────────────────────────
Configurable:     1.0k tokens
Working space:   ~146k tokens
```

Best for: Individual projects, quick tasks, maximum session length.

### Team Project — Medium Setup

```
CLAUDE.md:         500 tokens
Skills (8):        800 tokens
MCP tools (3):    3.0k tokens
Agents (2):       1.0k tokens
─────────────────────────────
Configurable:     5.3k tokens
Working space:   ~142k tokens
```

Best for: Established codebases with team conventions, moderate automation.

### Enterprise — Full Setup

```
CLAUDE.md:        1.0k tokens
Skills (15):      1.5k tokens
MCP tools (6):    6.0k tokens
Agents (5):       3.0k tokens
─────────────────────────────
Configurable:    11.5k tokens
Working space:  ~136k tokens
```

Best for: Complex projects with many integrations. Note the reduced working space — consider aggressive compaction settings.

## Monitoring and Adjusting

### Weekly Budget Review

Run `/tokens` at the start of a session before doing any work. Compare:

- Is configurable overhead growing? (New skills, MCP servers, memory files)
- Is working space sufficient for your typical tasks?
- Are compactions happening too frequently?

### Signs Your Budget Is Wrong

| Sign | Problem | Action |
|---|---|---|
| Compaction fires within 10 messages | Overhead too high or messages too large | Reduce configurable overhead, use file caching |
| Claude loses details frequently | Working space too small | Trim overhead, lower compaction threshold |
| Skills never trigger | Too many skills competing | Remove unused skills, sharpen descriptions |
| Sessions feel short | Total overhead > 15k | Audit and trim every component |

### The 80/20 Rule

Aim for **80% of your context** to be available as working space. With 200k tokens:
- Fixed overhead: ~20k (10%)
- Autocompact buffer: 33k (16.5%)
- Configurable overhead: < 7k (3.5%)
- Working space: > 140k (70%)

If working space drops below 130k, audit your configurable overhead.
