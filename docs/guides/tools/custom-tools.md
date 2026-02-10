# Create Custom Tools

Use the `@cortex.tool()` decorator to give your agent custom capabilities -- from API calls to database queries to file operations.

---

## Basic tool

Decorate an async function with `@cortex.tool()`:

```python
import cortex

@cortex.tool(name="get_weather", description="Get current weather for a city")
async def get_weather(city: str, units: str = "celsius") -> str:
    # Your real implementation here
    return f"22 degrees {units} and sunny in {city}"
```

The decorator extracts the parameter schema from the function signature. Type hints are required for all parameters.

## Attach tools to an agent

Pass a list of decorated tools when creating the agent:

```python
engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4o",
    worker_model="gpt-4o-mini",
)

agent = engine.create_agent(
    name="assistant",
    system_prompt="You help users with weather and order information.",
    tools=[get_weather, lookup_order],  # (1)!
)
```

1. Pass the decorated function objects directly. Do not call them.

## Multiple real-world examples

### Database lookup

```python
@cortex.tool(name="lookup_order", description="Look up an order by its ID")
async def lookup_order(order_id: str) -> str:
    # In production, query your database
    return f"Order {order_id}: shipped on 2026-02-08, arrives 2026-02-12"
```

### REST API call

```python
import httpx

@cortex.tool(name="search_docs", description="Search the knowledge base")
async def search_docs(query: str, max_results: int = 5) -> str:
    async with httpx.AsyncClient() as client:
        resp = await client.get(
            "https://api.internal.com/search",
            params={"q": query, "limit": max_results},
        )
        resp.raise_for_status()
        results = resp.json()
    return "\n".join(r["title"] for r in results["items"])
```

### File operation

```python
from pathlib import Path

@cortex.tool(name="read_log", description="Read the last N lines of a log file")
async def read_log(file_path: str, lines: int = 50) -> str:
    path = Path(file_path)
    if not path.exists():
        return f"File not found: {file_path}"
    content = path.read_text()
    return "\n".join(content.splitlines()[-lines:])
```

### Calculation

```python
@cortex.tool(name="calculate", description="Evaluate a mathematical expression")
async def calculate(expression: str) -> str:
    try:
        result = eval(expression, {"__builtins__": {}})  # (1)!
        return str(result)
    except Exception as e:
        return f"Error: {e}"
```

1. This is a simplified example. In production, use a safe math parser instead of `eval`.

!!! warning
    Never use `eval` with untrusted input in production. Use a sandboxed math library or expression parser.

## Error handling

Return error information as a string. The agent will see the error message and can adapt its approach:

```python
@cortex.tool(name="get_user", description="Fetch user profile by ID")
async def get_user(user_id: str) -> str:
    try:
        user = await db.fetch_user(user_id)
        if user is None:
            return f"No user found with ID: {user_id}"
        return f"Name: {user.name}, Email: {user.email}"
    except ConnectionError:
        return "Database connection failed. Please try again."
```

!!! tip
    Return descriptive error messages rather than raising exceptions. The agent uses the returned string to decide what to do next -- a clear error message leads to better recovery behavior.

## Check which tools were called

After a response, inspect the metadata:

```python
session = agent.start_session(user_id="user_123")
response = await session.run("What's the weather in Tokyo?")

print(response.content)
print(f"Tools called: {response.metadata.tools_called}")
```

## Tool reputation

Every tool builds a reputation over time. Tools that return useful results gain trust; tools that fail or return unhelpful results lose trust and may be quarantined. See [Manage Tool Reputation](tool-reputation.md).

```python
# Check reputation stats for all tools
rep_stats = session.get_reputation_stats()
print(rep_stats)
```

## Complete example

```python
import asyncio
import cortex


@cortex.tool(name="get_weather", description="Get weather for a city")
async def get_weather(city: str, units: str = "celsius") -> str:
    return f"22 degrees {units} and sunny in {city}"


@cortex.tool(name="lookup_order", description="Look up order status by ID")
async def lookup_order(order_id: str) -> str:
    return f"Order {order_id}: shipped, arriving Feb 12"


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
        orchestrator_model="gpt-4o",
        worker_model="gpt-4o-mini",
    )

    agent = engine.create_agent(
        name="support",
        system_prompt="You help users with weather and orders.",
        tools=[get_weather, lookup_order],
    )

    session = agent.start_session(user_id="user_123")

    response = await session.run("What's the weather in Berlin?")
    print(response.content)
    print(f"Tools called: {response.metadata.tools_called}")

    response = await session.run("Check order ORD-5678")
    print(response.content)
    print(f"Tools called: {response.metadata.tools_called}")

    await session.close()


asyncio.run(main())
```

---

## Next steps

- [Manage Tool Reputation](tool-reputation.md) -- understand how tool trust scores work.
- [Override with Targeted Modulation](../advanced/modulation-overrides.md) -- force-activate or silence specific tools.
- [Tune Agent Weights](../config/weight-tuning.md) -- the `autonomy` weight affects how aggressively the agent uses tools.
