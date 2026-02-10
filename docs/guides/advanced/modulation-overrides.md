# Override with Targeted Modulation

Force-activate or silence specific tools with time-based expiry, within enterprise safety boundaries.

---

## What is targeted modulation?

Targeted modulation gives you direct control over tool selection during a session. Instead of relying solely on the brain engine's automatic tool selection, you can:

- **Activate** a tool -- force it into the available pool for a set number of turns.
- **Silence** a tool -- remove it from the available pool for a set number of turns.

Both actions expire automatically after the specified number of turns.

## Activate a tool

Force a tool to be available for the next N turns, even if it is quarantined or has low reputation:

```python
import cortex

engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4o",
)

@cortex.tool(name="search_api", description="Search products")
async def search_api(query: str) -> str:
    return f"Results for: {query}"

agent = engine.create_agent(
    name="shop",
    system_prompt="You help users find products.",
    tools=[search_api],
)

session = agent.start_session(user_id="user_123")

# Force search_api to be available for the next 5 turns
session.activate_tool("search_api", turns=5)  # (1)!

response = await session.run("Find me a wireless mouse")
print(response.content)
```

1. The tool is guaranteed to be in the available pool for 5 turns, regardless of its reputation score.

## Silence a tool

Remove a tool from selection for the next N turns:

```python
# The search API is returning bad results -- silence it temporarily
session.silence_tool("search_api", turns=10)  # (1)!

# The agent will not use search_api for the next 10 turns
response = await session.run("Find me a wireless mouse")
print(f"Tools called: {response.metadata.tools_called}")  # search_api will not appear
```

1. The tool is excluded from selection for 10 turns, then automatically re-enabled.

!!! tip
    Use `silence_tool` as a quick fix when you detect a tool is misbehaving but have not yet deployed a code fix. It is faster than restarting the session.

## Time-based expiry

Both `activate_tool` and `silence_tool` accept a `turns` parameter that controls automatic expiry:

```python
# Activate for exactly 3 turns
session.activate_tool("search_api", turns=3)

# After 3 calls to session.run(), the override expires
# and the brain engine resumes normal tool selection
```

| Turns value | Behavior |
|---|---|
| `1` | Override applies to the next turn only |
| `5` | Override applies to the next 5 turns |
| `turns=10` | Common default for temporary fixes |

!!! note
    After expiry, the tool returns to whatever state the reputation system assigns it. If it was quarantined before activation, it returns to quarantine unless its trust score has recovered.

## Check active modulations

View all currently active modulation overrides:

```python
modulations = session.get_active_modulations()
print(modulations)
```

This returns a list of active overrides with their remaining turns.

## Enterprise safety boundaries

In enterprise configurations, modulation overrides respect the safety level:

```python
agent = engine.create_agent(
    name="secure_agent",
    system_prompt="You handle sensitive financial data.",
    tools=[search_api, get_account],
    enterprise_config=cortex.EnterpriseConfig(safety_level="strict"),  # (1)!
)
```

1. With `safety_level="strict"`, the enterprise config may restrict which tools can be activated or silenced.

!!! warning
    Enterprise safety boundaries take precedence over modulation overrides. If a tool is blocked by the enterprise config, `activate_tool` will not override that restriction.

## Combine modulation with simulation

Test the impact of a modulation override before applying it:

```python
# Simulate: what happens if we silence the search tool?
result = session.simulate_what_if({"autonomy": 0.5})
print(f"Simulation: {result}")

# Apply the modulation
session.silence_tool("search_api", turns=5)

# Monitor
response = await session.run("Help me find a product")
print(f"Tools called: {response.metadata.tools_called}")
print(f"Active modulations: {session.get_active_modulations()}")
```

## Complete example

```python
import asyncio
import cortex


@cortex.tool(name="search_api", description="Search products")
async def search_api(query: str) -> str:
    return f"Found 3 results for: {query}"


@cortex.tool(name="get_reviews", description="Get product reviews")
async def get_reviews(product_id: str) -> str:
    return f"Product {product_id}: 4.5 stars, 120 reviews"


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
        orchestrator_model="gpt-4o",
        worker_model="gpt-4o-mini",
    )

    agent = engine.create_agent(
        name="shop",
        system_prompt="You help users find and evaluate products.",
        tools=[search_api, get_reviews],
        enterprise_config=cortex.EnterpriseConfig(safety_level="standard"),
    )

    session = agent.start_session(user_id="user_123")

    # Normal operation
    await session.run("Find me a laptop")

    # Silence search temporarily (maybe the API is flaky)
    session.silence_tool("search_api", turns=3)

    # Force reviews to be available
    session.activate_tool("get_reviews", turns=5)

    # Check what is active
    print(f"Modulations: {session.get_active_modulations()}")

    # Continue -- only get_reviews is available
    response = await session.run("Tell me about product LP-200")
    print(response.content)
    print(f"Tools called: {response.metadata.tools_called}")

    await session.close()


asyncio.run(main())
```

---

## Next steps

- [Manage Tool Reputation](../tools/tool-reputation.md) -- understand the automatic trust system that modulation overrides.
- [Run What-If Simulations](what-if-simulation.md) -- preview the effect of modulation before applying it.
- [Monitor Your Agent](observability.md) -- track modulation events in your observability pipeline.
