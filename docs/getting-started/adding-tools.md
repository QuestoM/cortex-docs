# Adding Tools

Tools let your agent call functions -- query databases, trigger APIs, look up records. The `@cortex.tool` decorator turns any Python function into a tool the LLM can invoke.

---

## Define a tool

```python
import cortex


@cortex.tool(name="lookup_order", description="Look up an order by ID")
async def lookup_order(order_id: str) -> str:
    """
    Retrieve the current status of a customer order.

    Args:
        order_id: The order identifier (e.g. ORD-9281)
    """
    # Your database logic here
    return f"Order {order_id}: shipped, arriving Thursday"
```

The decorator extracts parameter names, types, and descriptions from your function signature and docstring automatically. No manual JSON schema required.

### What the decorator does

1. Reads the function signature for parameter names and type hints.
2. Parses the docstring for parameter descriptions.
3. Generates an OpenAI-compatible JSON schema.
4. Wraps the function in a `ToolWrapper` that handles sync/async execution.
5. Registers the tool in a global registry.

---

## Register tools with an agent

Pass tools to `create_agent()`:

```python
agent = engine.create_agent(
    name="support",
    system_prompt="You help customers with their orders.",
    tools=[lookup_order],  # (1)!
)
```

1.  Each tool is a `ToolWrapper` instance returned by the `@cortex.tool` decorator.

---

## Type hints

Type hints determine how the LLM sends arguments. Use standard Python types:

| Python type | JSON Schema type |
|---|---|
| `str` | `string` |
| `int` | `integer` |
| `float` | `number` |
| `bool` | `boolean` |
| `list` | `array` |
| `dict` | `object` |

```python
@cortex.tool(name="calculate_discount", description="Calculate a discount")
def calculate_discount(price: float, percent: int, apply: bool = False) -> str:
    discounted = price * (1 - percent / 100)
    if apply:
        return f"Applied. New price: ${discounted:.2f}"
    return f"Preview: ${discounted:.2f} (not applied)"
```

Parameters with default values are marked as optional in the schema. Parameters without defaults are required.

---

## Sync and async

Both sync and async functions work. The executor detects which type your function is and handles it accordingly.

=== "Async"

    ```python
    @cortex.tool(name="fetch_weather", description="Get current weather")
    async def fetch_weather(city: str) -> str:
        async with httpx.AsyncClient() as client:
            resp = await client.get(f"https://api.weather.com/{city}")
            return resp.text
    ```

=== "Sync"

    ```python
    @cortex.tool(name="add_numbers", description="Add two numbers")
    def add_numbers(a: int, b: int) -> int:
        return a + b
    ```

---

## Return values

Tools must return a `str` (or a value that can be converted to `str`). This string is passed back to the LLM as the tool result.

!!! tip "Return structured data"
    For complex results, return JSON:

    ```python
    import json

    @cortex.tool(name="get_user_profile", description="Fetch user profile")
    async def get_user_profile(user_id: str) -> str:
        profile = await db.users.find(user_id)
        return json.dumps({
            "name": profile.name,
            "email": profile.email,
            "plan": profile.plan,
        })
    ```

---

## Tool reputation

corteX tracks every tool call. Successful executions increase a tool's trust score; failures decrease it. After repeated failures, a tool is automatically quarantined -- the LLM will stop receiving it as an option.

This happens transparently. You can inspect reputation scores at any time:

```python
print(session.get_reputation_stats())
```

---

## Complete example

```python
import asyncio
import cortex


@cortex.tool(name="lookup_order", description="Look up an order by ID")
async def lookup_order(order_id: str) -> str:
    orders = {
        "ORD-9281": "Shipped - arriving Thursday",
        "ORD-1042": "Processing - estimated Friday",
    }
    return orders.get(order_id, f"Order {order_id} not found")


@cortex.tool(name="cancel_order", description="Cancel an order")
async def cancel_order(order_id: str, reason: str = "") -> str:
    return f"Order {order_id} cancelled. Reason: {reason or 'none provided'}"


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
    )

    agent = engine.create_agent(
        name="support",
        system_prompt="You help customers manage their orders.",
        tools=[lookup_order, cancel_order],
    )

    session = agent.start_session(user_id="user_42")

    response = await session.run("What's the status of ORD-9281?")
    print(response.content)
    print(f"Tools called: {response.metadata.tools_called}")

    await session.close()


asyncio.run(main())
```

Expected output:

```text
Your order ORD-9281 has been shipped and is arriving Thursday.
Tools called: ['lookup_order']
```

---

Next: [Streaming](streaming.md)
