---
name: agents
description: Design, build, and debug AI agent systems — including multi-step reasoning loops, tool use, memory, planning, multi-agent orchestration, and autonomous workflows. Use this skill whenever the user wants to build an AI agent, agentic pipeline, or autonomous workflow; asks about agent architectures (ReAct, Plan-and-Execute, reflection loops); wants to chain multiple LLM calls together; needs to build a system that uses tools, searches the web, reads files, executes code, or takes actions; mentions MCP (Model Context Protocol), LangChain, LangGraph, CrewAI, AutoGen, or similar frameworks; or wants to automate a multi-step task end-to-end. Also trigger when the user asks about prompt chaining, structured outputs for downstream use, or building reliable pipelines on top of Claude or other LLMs.
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
Generate output → Critique output → Revise → Repeat until quality threshold met
```
Best for: Writing, code generation, analysis quality improvement

### Multi-Agent Orchestration
```
Orchestrator agent → dispatches tasks to specialist agents
├── Research agent
├── Writing agent
├── QA/Review agent
└── Publishing agent
```
Best for: Parallel workstreams, role separation, large pipelines

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

---

## Memory Systems

| Type | What | How | When to use |
|---|---|---|---|
| **In-context** | System prompt, conversation history | Pass everything in each call | Short sessions, <50k tokens |
| **External/retrieval** | Vector DB, database | Embed + retrieve relevant chunks | Long-term memory, large knowledge bases |
| **Episodic** | Summaries of past sessions | Summarize + store after each session | Multi-session agents |
| **Semantic** | Facts about user/world | Structured key-value store | Persistent user preferences |

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

---

## Common Agent Patterns

### Research Agent
1. Receive topic
2. Generate 3–5 search queries
3. Execute searches in parallel
4. Synthesize findings
5. Identify gaps → additional searches
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

---

## Reliability & Error Handling
- **Retry with backoff**: Transient API errors — retry 3x with exponential backoff
- **Max iterations**: Always cap agent loops (default: 10 iterations)
- **Timeout**: Set per-step and total timeouts
- **Fallback**: If tool fails, agent should try alternative approach, not halt
- **Human-in-the-loop checkpoints**: For high-stakes actions (send email, publish, delete), pause and confirm
- **Logging**: Log every tool call + result for debugging

---

## Frameworks (when to use each)
| Framework | Best for |
|---|---|
| Raw Anthropic API | Full control, simple pipelines, production |
| LangChain | Rapid prototyping, many pre-built integrations |
| LangGraph | Stateful multi-agent graphs, complex workflows |
| CrewAI | Role-based multi-agent teams |
| AutoGen | Conversational multi-agent systems |
| MCP | Standardized tool servers, cross-agent reuse |

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

**LangSmith** (if using LangChain): wrap chains with `langsmith.traceable` for full trace visibility — inputs, outputs, latency per step.

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
Week 1-2: Research agents (gather + synthesize)
Week 3-4: Content agents (write + review)
Week 5-6: Distribution agents (publish + track)

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
