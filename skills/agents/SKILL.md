---
name: agents
description: Design, build, and debug AI agent systems — including multi-step reasoning loops, tool use, memory, planning, multi-agent orchestration, and autonomous workflows. Use this skill whenever the user wants to build an AI agent, agentic pipeline, or autonomous workflow; asks about agent architectures (ReAct, Plan-and-Execute, reflection loops, Tree of Thoughts, LATS); wants to chain multiple LLM calls together; needs to build a system that uses tools, searches the web, reads files, executes code, or takes actions; mentions MCP (Model Context Protocol), LangChain, LangGraph, CrewAI, AutoGen, Claude Agent SDK, or similar frameworks; or wants to automate a multi-step task end-to-end. Also trigger when the user asks about prompt chaining, structured outputs for downstream use, building reliable pipelines on top of Claude or other LLMs, or agent evaluation and observability.
---

# Agents Skill

## Agent Architectures

### ReAct (Reason + Act) — Default Pattern
```
Thought: What do I need to figure out?
Action: [tool_name] with [parameters]
Observation: [tool result]
Thought: What does this tell me? What's next?
Action: ...
Final Answer: [synthesized result]
```
Best for: Research, data gathering, multi-step Q&A

### Plan-and-Execute
```
Step 1: Planner agent creates a full plan (list of steps)
Step 2: Executor agent runs each step in order
Step 3: Replanner reviews results and updates plan if needed
```
Best for: Complex tasks with many interdependent steps

### Reflection / Self-Critique Loop
```
Generate output -> Critique output -> Revise -> Repeat until quality threshold met
```
Best for: Writing, code generation, analysis quality improvement

Key implementation detail: define explicit quality criteria before entering the loop. Without measurable criteria, reflection loops spin without converging. Common criteria: factual accuracy, completeness against a checklist, adherence to a style guide, or passing a test suite.

### Tree of Thoughts (ToT)
```
Step 1: Generate N candidate next-steps (branches)
Step 2: Evaluate each branch with a scoring heuristic
Step 3: Expand the top-K branches
Step 4: Repeat until a solution branch reaches the goal
```
Best for: Problems with large search spaces — math proofs, puzzle solving, strategic planning. More expensive than ReAct (requires multiple LLM calls per step), so use only when single-path reasoning fails.

### Language Agent Tree Search (LATS)
```
Step 1: Select a node using UCT (Upper Confidence bound for Trees)
Step 2: Expand by generating candidate actions
Step 3: Evaluate with environment feedback + LLM self-reflection
Step 4: Backpropagate scores up the tree
Step 5: Repeat until budget exhausted or solution found
```
Best for: Complex reasoning tasks where you want Monte Carlo Tree Search-style exploration combined with LLM reasoning. Produces higher-quality results than ReAct on hard problems at the cost of more LLM calls.

### Multi-Agent Orchestration
```
Orchestrator agent -> dispatches tasks to specialist agents
  |-- Research agent
  |-- Writing agent
  |-- QA/Review agent
  +-- Publishing agent
```
Best for: Parallel workstreams, role separation, large pipelines

See `references/multi-agent.md` for handoff patterns, shared memory, and framework comparisons.

---

## Tool Design

### Tool Specification Template
```python
{
  "name": "tool_name",
  "description": "What this tool does, when to use it, what it returns. Be very specific.",
  "parameters": {
    "type": "object",
    "properties": {
      "param1": {
        "type": "string",
        "description": "What this parameter is for"
      }
    },
    "required": ["param1"]
  }
}
```

### Tool Design Principles
- **Descriptions are critical**: The agent uses descriptions to decide when to call a tool. Be specific about inputs, outputs, and when NOT to use the tool.
- **Idempotency**: Tools that take actions (write, delete, post) should be safe to retry
- **Return structured data**: Return JSON, not prose — agents parse outputs, humans don't read them
- **Error messages should be actionable**: Not "Error 404" but "Post not found. Try fetching the list of posts first."
- **One responsibility per tool**: Don't combine search + summarize into one tool
- **Keep parameter counts low**: Fewer than 5 parameters per tool. Use sensible defaults. Tools with many required params confuse agents.
- **Validate inputs server-side**: Never trust agent-provided inputs — validate types, ranges, and permissions in the tool implementation.

### MCP (Model Context Protocol) Integration
MCP standardizes how agents connect to external tools and data. Instead of writing custom tool wrappers per agent, build an MCP server once and any MCP-compatible client can use it.

**When to use MCP vs. direct function calling:**
| Scenario | Approach |
|---|---|
| Tools shared across multiple agents | MCP server |
| Single-agent, few tools | Direct function calling |
| Third-party integrations (DB, APIs) | MCP server |
| Tight coupling to agent logic | Direct function calling |

**MCP server structure:**
```python
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("my-tools")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="search_docs",
            description="Search internal documentation by keyword",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "Search query"}
                },
                "required": ["query"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "search_docs":
        results = search_index(arguments["query"])
        return [TextContent(type="text", text=json.dumps(results))]
```

See `references/tools.md` for full MCP patterns, function calling best practices, and tool composition.

---

## Memory Systems

| Type | What | How | When to use |
|---|---|---|---|
| **In-context (short-term)** | System prompt, conversation history | Pass everything in each call | Short sessions, <50k tokens |
| **External/retrieval (long-term)** | Vector DB, database | Embed + retrieve relevant chunks | Long-term memory, large knowledge bases |
| **Episodic** | Summaries of past sessions | Summarize + store after each session | Multi-session agents |
| **Semantic** | Facts about user/world | Structured key-value store | Persistent user preferences |
| **Procedural** | Learned workflows and patterns | Store successful action sequences | Agents that improve over time |

### Memory Implementation Patterns

**Short-term memory (conversation buffer):**
Keep the last N messages. Simple, but loses context in long sessions. Use a sliding window with summarization: summarize messages beyond the window into a running summary prepended to the system prompt.

**Long-term memory (retrieval-augmented):**
```python
# Store: embed and save to vector DB
embedding = embed(text)
vector_db.upsert(id=doc_id, vector=embedding, metadata={"source": source})

# Retrieve: query with current context
query_embedding = embed(current_question)
results = vector_db.query(query_embedding, top_k=5)
context = "\n".join([r.text for r in results])
```

**Episodic memory (session summaries):**
At session end, generate a structured summary: what was accomplished, key decisions made, open items. Store as a timestamped record. At next session start, retrieve the last N summaries to restore context.

**Procedural memory (learned patterns):**
When an agent successfully completes a multi-step task, save the action sequence as a template. On similar future tasks, retrieve and adapt the template instead of reasoning from scratch.

---

## Building with Claude (Anthropic API)

### Basic Tool Use
```python
import anthropic

client = anthropic.Anthropic()

tools = [
    {
        "name": "web_search",
        "description": "Search the web for current information",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Search query"}
            },
            "required": ["query"]
        }
    }
]

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=4096,
    tools=tools,
    messages=[{"role": "user", "content": "What happened in AI news this week?"}]
)
```

### Agentic Loop
```python
def run_agent(user_message, tools, tool_executor):
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        # If done, return final text
        if response.stop_reason == "end_turn":
            return next(b.text for b in response.content if b.type == "text")

        # Process tool calls
        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = tool_executor(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })

            messages.append({"role": "user", "content": tool_results})
```

### Structured Outputs for Agent Pipelines
When agents pass data between steps, enforce structure to prevent drift:

```python
import json

# Force JSON output with a schema
response = client.messages.create(
    model="claude-sonnet-4-5",
    max_tokens=2000,
    system="""Return ONLY valid JSON matching this schema:
{
  "findings": [{"title": str, "summary": str, "confidence": float}],
  "next_steps": [str],
  "requires_human_review": bool
}""",
    messages=[{"role": "user", "content": task}]
)

# Parse and validate
result = json.loads(response.content[0].text)
assert "findings" in result, "Missing required field: findings"
```

For critical pipelines, use Pydantic models to validate agent outputs before passing downstream:
```python
from pydantic import BaseModel

class AgentOutput(BaseModel):
    findings: list[dict]
    next_steps: list[str]
    requires_human_review: bool

validated = AgentOutput.model_validate_json(response.content[0].text)
```

---

## Prompt Engineering for Agents

### System Prompt Structure for Agents
```
You are [role]. Your goal is [goal].

## Available Tools
[List tools with brief descriptions]

## Rules
- Always think step by step before acting
- Use tools only when necessary
- If uncertain, ask rather than assume
- Stop and return your answer when you have enough information

## Output Format
[Specify exactly how you want the final output structured]
```

### Reducing Hallucination in Tool-Heavy Agents
- Instruct agent to cite which tool result supports each claim
- Add "If you don't know, say so — don't guess" to system prompt
- Use structured output (JSON) for factual claims
- Add a verification step: "Before finalizing, check your answer against the source"
- Ground every claim in a specific tool_use_id from the conversation

---

## Common Agent Patterns

### Research Agent
1. Receive topic
2. Generate 3-5 search queries
3. Execute searches in parallel
4. Synthesize findings
5. Identify gaps -> additional searches
6. Return structured report with sources

### Content Pipeline Agent
1. Receive brief
2. Research (search + retrieve)
3. Outline
4. Draft each section
5. Self-review against brief criteria
6. Format for target platform
7. Publish via API

### QA / Review Agent
1. Receive content/code/plan
2. Generate review criteria from context
3. Evaluate against each criterion
4. Return scored feedback + specific suggestions

### Code Generation Agent
1. Receive requirements
2. Break into functions/modules
3. Generate each unit with tests
4. Run tests, collect failures
5. Fix failures using test output as feedback
6. Repeat until all tests pass or max iterations hit

---

## Reliability & Error Handling

### Core Principles
- **Retry with backoff**: Transient API errors — retry 3x with exponential backoff
- **Max iterations**: Always cap agent loops (default: 10 iterations)
- **Timeout**: Set per-step and total timeouts
- **Fallback**: If tool fails, agent should try alternative approach, not halt
- **Human-in-the-loop checkpoints**: For high-stakes actions (send email, publish, delete), pause and confirm
- **Logging**: Log every tool call + result for debugging

### Error Recovery Strategies

**Graceful degradation:** When a tool is unavailable, the agent should attempt the task with remaining tools rather than failing entirely. Define fallback chains: if `web_search` fails, try `cached_search`; if that fails, use knowledge cutoff data and disclose it.

**Retry with reformulation:** If a tool returns an unhelpful result, the agent should reformulate its input rather than retrying with the same parameters. Example: if a search returns no results, broaden the query or try synonyms.

**Circuit breaker pattern:** After N consecutive failures on the same tool, stop calling it for the remainder of the session. Prevents infinite retry loops.

```python
class CircuitBreaker:
    def __init__(self, max_failures=3, reset_after_seconds=300):
        self.failures = 0
        self.max_failures = max_failures
        self.last_failure_time = None
        self.reset_after = reset_after_seconds

    def can_call(self) -> bool:
        if self.failures >= self.max_failures:
            if time.time() - self.last_failure_time > self.reset_after:
                self.failures = 0
                return True
            return False
        return True

    def record_failure(self):
        self.failures += 1
        self.last_failure_time = time.time()

    def record_success(self):
        self.failures = 0
```

**Checkpoint and resume:** For long-running agents, periodically save state so the agent can resume from the last checkpoint after a crash rather than restarting from scratch.

---

## Frameworks (when to use each)
| Framework | Best for |
|---|---|
| Raw Anthropic API | Full control, simple pipelines, production |
| Claude Agent SDK | First-party agent framework with built-in tool use, guardrails, and handoffs |
| LangChain | Rapid prototyping, many pre-built integrations |
| LangGraph | Stateful multi-agent graphs, complex workflows with cycles |
| CrewAI | Role-based multi-agent teams with delegation |
| AutoGen | Conversational multi-agent systems, research-style collaboration |
| MCP | Standardized tool servers, cross-agent tool reuse |

See `references/multi-agent.md` for detailed framework comparisons and migration patterns.

---

## Persona / Agent Identity Pattern
When building a specialized agent, define its identity beyond just tools:

```markdown
# Agent Persona: [Role Name]
**Identity**: Senior [role] with [X] years experience in [domain]
**Communication style**: [e.g., direct, data-driven, asks clarifying questions]
**Decision framework**: [e.g., always quantify trade-offs, default to simplicity]
**Loaded skills**: [list of skills this persona activates]
**Workflow**: [the persona's default approach to a task]
```

Store persona files in `.claude/agents/` — Claude Code loads them automatically.

---

## Observability & Debugging
For production agents, always add logging:

```python
import logging
from datetime import datetime

def log_tool_call(tool_name: str, inputs: dict, result: str, duration_ms: int):
    logging.info({
        "ts": datetime.utcnow().isoformat(),
        "tool": tool_name,
        "inputs": inputs,
        "result_preview": result[:200],
        "duration_ms": duration_ms,
        "success": "error" not in result.lower()
    })
```

### Tracing and Spans
For multi-step agents, wrap each step in a trace span to build a full execution timeline:

```python
import time
import uuid

class AgentTracer:
    def __init__(self):
        self.trace_id = str(uuid.uuid4())
        self.spans = []

    def start_span(self, name: str, metadata: dict = None) -> dict:
        span = {
            "span_id": str(uuid.uuid4()),
            "trace_id": self.trace_id,
            "name": name,
            "start_time": time.time(),
            "metadata": metadata or {}
        }
        self.spans.append(span)
        return span

    def end_span(self, span: dict, result: str = None, error: str = None):
        span["end_time"] = time.time()
        span["duration_ms"] = (span["end_time"] - span["start_time"]) * 1000
        span["result_preview"] = (result or "")[:200]
        span["error"] = error

    def export(self) -> list[dict]:
        return self.spans
```

**LangSmith** (if using LangChain): wrap chains with `langsmith.traceable` for full trace visibility — inputs, outputs, latency per step.

**Key metrics to track:**
- Total tokens consumed per agent run
- Tool call success/failure rates
- End-to-end latency
- Iteration count (did the agent converge or hit the max?)
- Final output quality score (if using automated evaluation)

See `references/evaluation.md` for evaluation frameworks and testing patterns.

---

## Subagent Dispatch Pattern
For parallelizable work (research, audits, content generation):

```python
import asyncio

async def run_subagent(task: str, context: str) -> str:
    """Dispatch a task to a subagent with its own context"""
    response = await client.messages.create(
        model="claude-sonnet-4-5",
        max_tokens=2000,
        system=f"You are a specialist agent. Context: {context}",
        messages=[{"role": "user", "content": task}]
    )
    return response.content[0].text

async def parallel_research(topics: list[str]) -> list[str]:
    tasks = [run_subagent(f"Research: {t}", context="") for t in topics]
    return await asyncio.gather(*tasks)
```

---

## Agent Orchestration Protocol (lightweight)
For multi-agent pipelines without a heavy framework:

```
Phase 1: Research agents (gather + synthesize)
Phase 2: Content agents (write + review)
Phase 3: Distribution agents (publish + track)

Each agent receives:
- Role and goal
- Relevant skill files
- Previous agents' outputs as context
- Clear output format specification
```

---

## Context Management Best Practices
- Summarize long tool results before adding to messages (keep under 500 tokens per result)
- Use a "scratchpad" system message slot for agent working memory
- For multi-session agents: save session summary to file, load at next session start
- Prune conversation history: keep system prompt + last N turns + tool results
- For large contexts: use hierarchical summarization — summarize groups of messages into progressively shorter summaries as they age
- Token budget: reserve at least 25% of the context window for the model's response and reasoning
