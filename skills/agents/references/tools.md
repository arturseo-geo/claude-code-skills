# Tool Use Patterns

## Function Calling Best Practices

### Tool Description Quality
The tool description is the single most important factor in whether an agent uses a tool correctly. Bad descriptions cause bad tool use.

**Bad:**
```json
{"name": "search", "description": "Search things"}
```

**Good:**
```json
{
  "name": "search_knowledge_base",
  "description": "Search the internal knowledge base for documentation, FAQs, and policy documents. Returns up to 10 results ranked by relevance. Use this BEFORE attempting to answer questions about company policies. Do NOT use for general web searches — use web_search instead.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Natural language search query. Be specific — 'vacation policy for contractors' works better than 'vacation'."
      },
      "max_results": {
        "type": "integer",
        "description": "Number of results to return (1-20). Default: 5.",
        "default": 5
      }
    },
    "required": ["query"]
  }
}
```

### Key Rules for Tool Definitions
1. **Say when NOT to use the tool** — negative examples prevent misuse
2. **Describe the return format** — "Returns a JSON array of {title, url, snippet}" helps the agent parse results
3. **Include parameter hints** — example values in descriptions improve accuracy
4. **Keep required params minimal** — every required param is a chance for the agent to get stuck
5. **Use enums for constrained values** — `"enum": ["asc", "desc"]` is better than describing valid values in text

---

## MCP (Model Context Protocol) Integration

### When to Build an MCP Server
Build an MCP server when:
- Multiple agents or clients need the same tools
- You want to decouple tool implementation from agent logic
- You are integrating with databases, APIs, or file systems that multiple projects share
- You want tool versioning and independent deployment

### MCP Server Template
```python
from mcp.server import Server
from mcp.types import Tool, TextContent
import json

server = Server("my-service")

@server.list_tools()
async def list_tools() -> list[Tool]:
    return [
        Tool(
            name="get_customer",
            description="Look up a customer by ID. Returns name, email, plan, and account status.",
            inputSchema={
                "type": "object",
                "properties": {
                    "customer_id": {
                        "type": "string",
                        "description": "Customer ID (format: cust_XXXX)"
                    }
                },
                "required": ["customer_id"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict) -> list[TextContent]:
    if name == "get_customer":
        customer = await db.get_customer(arguments["customer_id"])
        if not customer:
            return [TextContent(
                type="text",
                text=json.dumps({"error": "Customer not found", "suggestion": "Verify the ID format is cust_XXXX"})
            )]
        return [TextContent(type="text", text=json.dumps(customer))]
```

### MCP Configuration in Claude Code
```json
// .mcp.json
{
  "mcpServers": {
    "my-service": {
      "command": "python",
      "args": ["-m", "my_mcp_server"],
      "env": {
        "DATABASE_URL": "postgresql://..."
      }
    }
  }
}
```

### MCP Best Practices
- **One server per domain** — don't mix customer tools and analytics tools in one server
- **Return JSON always** — even for errors, return structured JSON with an "error" key
- **Include suggestions in errors** — help the agent self-correct
- **Log all calls server-side** — you need observability into what agents are doing with your tools
- **Rate limit per client** — prevent runaway agents from overwhelming your services
- **Version your tools** — when tool behavior changes, update the description to reflect it

---

## Tool Composition Patterns

### Sequential Tool Chain
Agent calls tools in sequence where each output feeds the next:
```
search("quarterly revenue") -> extract_table(url) -> calculate_growth(data)
```

### Parallel Tool Fan-Out
Agent identifies independent subtasks and calls tools in parallel:
```
parallel:
  search("competitor A revenue")
  search("competitor B revenue")
  search("competitor C revenue")
then: compare_results(results)
```

Claude supports parallel tool calls natively — return multiple `tool_use` blocks in a single response.

### Conditional Tool Use
Agent decides which tool to use based on context:
```
if input looks like a URL -> fetch_page(url)
if input looks like a question -> search(query)
if input looks like code -> execute_code(code)
```

Implement this in the system prompt: "If the user provides a URL, use fetch_page. If they ask a question, use search first."

### Tool Fallback Chain
When the primary tool fails, fall back to alternatives:
```python
async def resilient_search(query: str) -> str:
    """Try multiple search tools in order"""
    for tool in ["primary_search", "cached_search", "knowledge_base"]:
        try:
            result = await call_tool(tool, {"query": query})
            if result and "error" not in result:
                return result
        except Exception:
            continue
    return {"error": "All search tools failed", "query": query}
```

---

## Structured Output Enforcement

### JSON Mode
Force agents to return valid JSON by:
1. Specifying the exact schema in the system prompt
2. Including an example of valid output
3. Validating with Pydantic before passing downstream

```python
from pydantic import BaseModel, Field

class SearchResult(BaseModel):
    query: str
    results: list[dict] = Field(description="List of {title, url, snippet}")
    total_found: int
    confidence: float = Field(ge=0.0, le=1.0)

# Validate agent output
try:
    validated = SearchResult.model_validate_json(agent_output)
except ValidationError as e:
    # Ask agent to fix its output
    retry_with_error(f"Your output was invalid JSON: {e}")
```

### When to Use Structured Outputs
- Agent-to-agent communication (always)
- Database writes (always)
- User-facing reports (use structured intermediate, render to prose at the end)
- Logging and observability (always)

---

## Security Considerations

### Tool Permission Levels
Define what each tool can do and enforce it server-side:
```python
TOOL_PERMISSIONS = {
    "read_file": {"level": "read", "requires_approval": False},
    "write_file": {"level": "write", "requires_approval": True},
    "delete_file": {"level": "destructive", "requires_approval": True},
    "web_search": {"level": "read", "requires_approval": False},
    "send_email": {"level": "external", "requires_approval": True},
}
```

### Input Sanitization
Never pass agent-generated inputs directly to:
- SQL queries (use parameterized queries)
- Shell commands (use allowlists, not blocklists)
- File paths (validate against an allowed directory)
- API calls with authentication (validate scopes)

### Principle of Least Privilege
Give agents only the tools they need for the current task. Don't expose a "delete all records" tool to a research agent.
