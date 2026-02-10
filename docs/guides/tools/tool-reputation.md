# Manage Tool Reputation

Understand how the reputation system scores tool reliability, handles quarantine, and how you can inspect and influence tool trust.

---

## How reputation works

Every tool registered with an agent accumulates a **trust score** based on its execution history. The reputation system uses game-theory-inspired trust modeling:

- **Successful calls** increase trust.
- **Failures** (exceptions, timeouts, unhelpful results) decrease trust.
- **Consecutive failures** trigger quarantine -- the tool is temporarily excluded from selection.
- **Recovery** happens automatically as the engine periodically re-tests quarantined tools.

## Check reputation stats

Inspect the current trust state of all tools at any point during a session:

```python
import cortex

engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4o",
)

@cortex.tool(name="search_api", description="Search an external API")
async def search_api(query: str) -> str:
    return f"Results for: {query}"

@cortex.tool(name="get_price", description="Get product price")
async def get_price(product_id: str) -> str:
    return f"Product {product_id}: $29.99"

agent = engine.create_agent(
    name="shop_assistant",
    system_prompt="You help users find products.",
    tools=[search_api, get_price],
)

session = agent.start_session(user_id="user_123")

# After some interactions...
response = await session.run("Find me a laptop under $500")

# Check trust scores
rep_stats = session.get_reputation_stats()
print(rep_stats)
```

The reputation stats include:

- **Trust score** per tool (0.0 to 1.0)
- **Call count** and success rate
- **Quarantine status** (active or not)
- **Last call timestamp**

## Understand quarantine

When a tool fails repeatedly, the engine quarantines it:

```
Tool "search_api" quarantined after 3 consecutive failures.
The engine will retry in 5 turns.
```

!!! note
    Quarantine is automatic and transparent. The agent simply stops being offered the quarantined tool as an option until the quarantine period expires and the tool succeeds on a retry.

During quarantine:

1. The tool is excluded from the agent's available tool list.
2. After a cooldown period, the engine re-tests the tool with a probe call.
3. If the probe succeeds, trust begins recovering.
4. If the probe fails, quarantine extends.

## Force-activate a quarantined tool

If you know a tool's underlying issue is resolved, you can override quarantine:

```python
session.activate_tool("search_api", turns=5)  # (1)!
```

1. Forces the tool back into the active pool for the next 5 turns. See [Override with Targeted Modulation](../advanced/modulation-overrides.md).

!!! warning
    Force-activating a tool that is still broken will generate more failures and further damage its trust score. Only use this when you have confirmed the root cause is fixed.

## Silence a tool temporarily

If a tool is misbehaving but not yet quarantined, you can preemptively silence it:

```python
session.silence_tool("search_api", turns=10)  # (1)!
```

1. Removes the tool from selection for the next 10 turns.

## Game theory trust model

The reputation system models each tool as a player in an iterated game:

- **Cooperate** = tool returns a useful result.
- **Defect** = tool fails or returns garbage.

The engine uses a **tit-for-tat with forgiveness** strategy:

1. New tools start with neutral trust (0.5).
2. Each success increases trust toward 1.0.
3. Each failure decreases trust toward 0.0.
4. After quarantine, a single success can restore partial trust (forgiveness).

This prevents permanently blacklisting tools that had a temporary outage.

## Monitor tool performance over time

Combine reputation stats with response metadata for full visibility:

```python
session = agent.start_session(user_id="user_123")

response = await session.run("Search for wireless keyboards")
print(f"Tools called: {response.metadata.tools_called}")

# Reputation after the call
rep = session.get_reputation_stats()
print(f"Reputation: {rep}")

# Calibration report includes tool accuracy signals
calibration = session.get_calibration_report()
print(f"Calibration: {calibration}")
```

## Complete example

```python
import asyncio
import cortex


@cortex.tool(name="search_api", description="Search products")
async def search_api(query: str) -> str:
    return f"Found 3 results for: {query}"


@cortex.tool(name="get_price", description="Get product price")
async def get_price(product_id: str) -> str:
    return f"Product {product_id}: $29.99"


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
        orchestrator_model="gpt-4o",
        worker_model="gpt-4o-mini",
    )

    agent = engine.create_agent(
        name="shop",
        system_prompt="You help users find and price products.",
        tools=[search_api, get_price],
    )

    session = agent.start_session(user_id="user_123")

    await session.run("Find me a mechanical keyboard")
    await session.run("What does product KB-100 cost?")

    # Inspect reputation
    rep = session.get_reputation_stats()
    print(f"Tool reputation: {rep}")

    # Check if any tools are quarantined
    print(f"Modulations: {session.get_active_modulations()}")

    await session.close()


asyncio.run(main())
```

---

## Next steps

- [Create Custom Tools](custom-tools.md) -- build tools that earn good reputation.
- [Override with Targeted Modulation](../advanced/modulation-overrides.md) -- manually activate or silence tools.
- [Monitor Your Agent](../advanced/observability.md) -- track tool reputation trends over time.
