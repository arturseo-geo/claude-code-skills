# Agent Architecture Patterns

## ReAct (Reason + Act)

The default pattern for most agent tasks. The agent alternates between reasoning about what to do and taking actions via tools.

**When to use:** General-purpose tasks — research, data gathering, multi-step Q&A, code debugging.

**When to avoid:** Tasks requiring exploration of multiple solution paths simultaneously (use ToT or LATS instead).

**Implementation notes:**
- Keep the reasoning trace visible (don't suppress `Thought:` steps) — it improves accuracy
- Cap iterations at 10-15 to prevent infinite loops
- If the agent repeats the same action twice, inject a hint: "You already tried this. Try a different approach."

```
Thought: I need to find the current price of AAPL stock.
Action: web_search("AAPL stock price today")
Observation: AAPL is trading at $198.50 as of 3:45 PM ET.
Thought: I have the price. The user also asked about the P/E ratio.
Action: web_search("AAPL P/E ratio 2026")
Observation: AAPL forward P/E is 28.3.
Final Answer: AAPL is at $198.50 with a forward P/E of 28.3.
```

---

## Plan-and-Execute

Separates planning from execution. A planner agent creates a step-by-step plan, then an executor agent runs each step. After execution, a replanner reviews results and adjusts.

**When to use:** Complex tasks with 5+ interdependent steps, tasks where the full scope is known upfront.

**When to avoid:** Exploratory tasks where the path is unknown, simple tasks (overhead not justified).

**Implementation:**
```python
# Phase 1: Plan
plan_response = client.messages.create(
    model="claude-opus-4-5",
    system="Create a numbered step-by-step plan to accomplish the goal. Each step should be a single atomic action.",
    messages=[{"role": "user", "content": f"Goal: {goal}"}]
)
steps = parse_plan(plan_response)

# Phase 2: Execute each step
results = []
for step in steps:
    result = execute_step(step, context=results)
    results.append(result)

# Phase 3: Replan if needed
if any(r.failed for r in results):
    revised_plan = replan(original_plan=steps, results=results)
    # Execute revised steps...
```

**Key design decisions:**
- Planner model can be more expensive (opus) while executor uses a cheaper model (sonnet)
- Include a replanning step after every N steps or after any failure
- Pass accumulated results as context to each subsequent step

---

## Reflection / Self-Critique

The agent generates an output, then critiques it against explicit criteria, then revises. Repeats until the output meets quality thresholds or max iterations are reached.

**When to use:** Writing tasks, code generation, analysis where quality matters more than speed.

**When to avoid:** Simple factual lookups, time-critical tasks.

**Implementation:**
```python
MAX_REVISIONS = 3
QUALITY_THRESHOLD = 0.8

output = generate_initial(task)
for i in range(MAX_REVISIONS):
    critique = evaluate(output, criteria=[
        "factual_accuracy",
        "completeness",
        "clarity",
        "follows_style_guide"
    ])

    if critique.score >= QUALITY_THRESHOLD:
        break

    output = revise(output, feedback=critique.feedback)
```

**Critique criteria examples:**
- Code: Does it pass all tests? Are there edge cases? Is it readable?
- Writing: Does it address all points in the brief? Is it the right length? Tone?
- Analysis: Are claims supported by data? Are there logical gaps?

---

## Tree of Thoughts (ToT)

Explores multiple reasoning paths in parallel, evaluates each, and expands the most promising ones. Think of it as breadth-first search over reasoning chains.

**When to use:** Math problems, puzzles, strategic planning, creative tasks with many valid approaches.

**When to avoid:** Simple tasks (massive overkill), tasks with obvious single paths.

**Implementation:**
```python
def tree_of_thoughts(problem: str, breadth: int = 3, depth: int = 4) -> str:
    candidates = [{"path": [], "state": problem}]

    for level in range(depth):
        next_candidates = []
        for candidate in candidates:
            # Generate N next-steps
            branches = generate_branches(candidate["state"], n=breadth)
            for branch in branches:
                score = evaluate_branch(branch, problem)
                next_candidates.append({
                    "path": candidate["path"] + [branch],
                    "state": apply_step(candidate["state"], branch),
                    "score": score
                })

        # Keep top-K candidates
        candidates = sorted(next_candidates, key=lambda x: x["score"], reverse=True)[:breadth]

    return candidates[0]  # Best solution path
```

**Cost consideration:** ToT requires `breadth * depth` LLM calls at minimum. For breadth=3 and depth=4, that is 12+ calls per problem. Use only when single-path reasoning fails.

---

## Language Agent Tree Search (LATS)

Combines Monte Carlo Tree Search (MCTS) with LLM reasoning. Uses UCT (Upper Confidence bound for Trees) to balance exploration vs. exploitation.

**When to use:** Hard reasoning tasks where you want the best possible answer and have budget for many LLM calls. Benchmarks show LATS outperforms ReAct and ToT on complex reasoning.

**When to avoid:** Simple tasks, latency-sensitive applications, budget-constrained scenarios.

**Key components:**
1. **Selection:** Use UCT to pick which node to expand next
2. **Expansion:** Generate candidate actions from the selected node
3. **Evaluation:** Use environment feedback (test results, search results) + LLM self-reflection to score
4. **Backpropagation:** Update scores up the tree

**Comparison to other patterns:**

| Pattern | LLM calls per task | Quality | Best for |
|---|---|---|---|
| ReAct | 3-15 | Good | General tasks |
| Plan-Execute | 5-20 | Good | Structured tasks |
| Reflection | 3-9 | Better | Quality-critical tasks |
| ToT | 10-50 | Better | Search/puzzle tasks |
| LATS | 20-100+ | Best | Hard reasoning, math |

---

## Choosing the Right Pattern

```
Is the task simple (1-3 steps)?
  Yes -> ReAct
  No ->
    Is the full plan knowable upfront?
      Yes -> Plan-and-Execute
      No ->
        Does quality matter more than speed?
          Yes ->
            Is there a verifiable correct answer?
              Yes -> LATS or ToT
              No -> Reflection loop
          No -> ReAct with more iterations

    Are there independent subtasks?
      Yes -> Multi-agent with parallel dispatch
```
