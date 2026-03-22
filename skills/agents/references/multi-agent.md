# Multi-Agent Orchestration Patterns

## Core Concepts

Multi-agent systems use multiple specialized LLM-powered agents that collaborate to complete tasks. Each agent has a defined role, set of tools, and communication protocol.

**When to use multi-agent vs. single agent:**
- Single agent: task is sequential, one domain, <10 tool calls
- Multi-agent: parallel subtasks, multiple domains, complex workflows, role separation needed

---

## Orchestration Patterns

### Hub-and-Spoke (Orchestrator)
A central orchestrator agent dispatches tasks to specialist agents and synthesizes results.

```
                 Orchestrator
                /     |      \
         Research   Writer   QA Agent
          Agent     Agent
```

**Implementation:**
```python
async def orchestrator(goal: str):
    # Step 1: Plan
    plan = await plan_agent(goal)

    # Step 2: Dispatch to specialists
    research = await research_agent(plan.research_tasks)

    # Step 3: Pass results downstream
    draft = await writer_agent(plan.writing_brief, context=research)

    # Step 4: Quality check
    review = await qa_agent(draft, criteria=plan.quality_criteria)

    if review.passes:
        return draft
    else:
        # Revision loop
        revised = await writer_agent(review.feedback, context=draft)
        return revised
```

**Pros:** Clear control flow, easy to debug, deterministic routing.
**Cons:** Orchestrator is a bottleneck, single point of failure.

### Pipeline (Sequential Handoff)
Agents process work in sequence, each adding to or transforming the output.

```
Input -> Agent A -> Agent B -> Agent C -> Output
```

**Best for:** Content pipelines, data processing, ETL-style workflows.

**Implementation:**
```python
async def pipeline(input_data: dict) -> dict:
    stages = [research_agent, outline_agent, draft_agent, review_agent, format_agent]

    result = input_data
    for stage in stages:
        result = await stage(result)

        # Checkpoint after each stage
        save_checkpoint(stage.__name__, result)

    return result
```

### Debate / Adversarial
Two or more agents argue different positions. A judge agent synthesizes the best answer.

```
Agent A (Pro) --\
                 --> Judge Agent --> Final Answer
Agent B (Con) --/
```

**Best for:** Decision-making, risk analysis, exploring trade-offs.

### Swarm (Dynamic Routing)
Agents hand off to each other based on the current state. No central orchestrator.

```
Agent A --handoff--> Agent B --handoff--> Agent C
    ^                                        |
    +-------- handoff if needed -------------+
```

**Best for:** Customer support flows, complex routing logic, conversational agents that switch between domains.

---

## Agent Handoff Patterns

### Explicit Handoff
The current agent decides when to hand off and to whom:
```python
# Agent returns a handoff instruction
{
    "action": "handoff",
    "target_agent": "billing_specialist",
    "context": "Customer has a billing dispute about invoice #1234",
    "conversation_history": [...]
}
```

### Context Transfer
When handing off between agents, transfer only what the next agent needs:
```python
def prepare_handoff(source_agent_output: dict, target_role: str) -> dict:
    return {
        "summary": source_agent_output["summary"],        # What was accomplished
        "key_findings": source_agent_output["findings"],   # Important data
        "open_questions": source_agent_output["gaps"],     # What still needs work
        "constraints": source_agent_output["constraints"], # Rules to follow
        # Don't pass: raw tool outputs, internal reasoning, full conversation history
    }
```

### Shared Memory (Blackboard Pattern)
All agents read from and write to a shared state store:
```python
class SharedMemory:
    def __init__(self):
        self.state = {}
        self.lock = asyncio.Lock()

    async def read(self, key: str) -> any:
        return self.state.get(key)

    async def write(self, key: str, value: any, agent_id: str):
        async with self.lock:
            self.state[key] = {
                "value": value,
                "updated_by": agent_id,
                "timestamp": time.time()
            }

    async def get_full_state(self) -> dict:
        return {k: v["value"] for k, v in self.state.items()}
```

**Use shared memory when:** agents work on overlapping data, order of operations is flexible, you want loose coupling between agents.

---

## Framework Comparison (2026)

### Claude Agent SDK
Anthropic's first-party agent framework. Provides built-in tool use, guardrails, handoff primitives, and tracing.

**Best for:** Production agents built on Claude. Tightest integration with Anthropic API features.

```python
from claude_agent import Agent, Tool, Handoff

research_agent = Agent(
    name="researcher",
    model="claude-sonnet-4-5",
    tools=[web_search, document_reader],
    instructions="You research topics thoroughly. Return structured findings."
)

writer_agent = Agent(
    name="writer",
    model="claude-sonnet-4-5",
    tools=[],
    instructions="You write clear, engaging content based on research findings."
)

# Handoff from researcher to writer
orchestrator = Agent(
    name="orchestrator",
    model="claude-opus-4-5",
    handoffs=[Handoff(target=research_agent), Handoff(target=writer_agent)]
)
```

### LangGraph
Stateful graph-based agent orchestration. Agents are nodes, edges define transitions. Supports cycles (loops), conditional branching, and persistent state.

**Best for:** Complex workflows with conditional logic, loops, and parallel branches. When you need visual debugging of agent flow.

```python
from langgraph.graph import StateGraph

graph = StateGraph(AgentState)
graph.add_node("research", research_node)
graph.add_node("write", write_node)
graph.add_node("review", review_node)
graph.add_edge("research", "write")
graph.add_conditional_edges("review", should_revise, {"revise": "write", "done": END})
```

### CrewAI
Role-based multi-agent teams with delegation. Each agent has a role, goal, and backstory.

**Best for:** Teams of agents with clear role separation. Good for content creation, research, and analysis pipelines.

```python
from crewai import Agent, Task, Crew

researcher = Agent(role="Senior Researcher", goal="Find accurate data", backstory="...")
writer = Agent(role="Content Writer", goal="Write engaging content", backstory="...")

task1 = Task(description="Research the topic", agent=researcher)
task2 = Task(description="Write the article", agent=writer)

crew = Crew(agents=[researcher, writer], tasks=[task1, task2])
result = crew.kickoff()
```

### AutoGen
Conversational multi-agent framework. Agents chat with each other to solve problems.

**Best for:** Research collaboration, brainstorming, code review, tasks that benefit from back-and-forth discussion.

### Choosing a Framework

```
Do you need production reliability and use Claude?
  Yes -> Claude Agent SDK or raw Anthropic API
  No ->
    Do you need complex graph-based workflows?
      Yes -> LangGraph
      No ->
        Do you want role-based teams?
          Yes -> CrewAI
          No ->
            Do you want conversational collaboration?
              Yes -> AutoGen
              No -> Raw API + custom orchestration
```

---

## Anti-Patterns

### The "Too Many Agents" Trap
Don't create a separate agent for every small subtask. If two agents always run in sequence with no branching, merge them into one agent with two phases.

### Infinite Delegation
Agent A asks Agent B, which asks Agent C, which asks Agent A. Always enforce a maximum delegation depth and acyclic handoff graphs (or explicit cycle budgets).

### Shared State Corruption
Multiple agents writing to the same state without coordination. Use locks, or better: have each agent write to its own namespace and let the orchestrator merge.

### Over-Communicating Context
Passing the full conversation history to every agent. Each agent should receive only the context it needs — a summary, key findings, and its specific task.
