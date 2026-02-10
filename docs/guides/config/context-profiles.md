# Configure Context Profiles

Use `ContextManagementConfig` to control how your agent manages memory, allocates token budgets, and prioritizes information across conversation tiers.

---

## What is a context profile?

A context profile is a preset configuration for the Cortical Context Engine. It controls:

- **Token budget** -- how many tokens the context window can hold before eviction.
- **Hot/warm/cold ratios** -- how memory is distributed across recency tiers.
- **Summarization strategy** -- when and how older messages are compressed.

## Built-in profiles

corteX ships with three built-in profiles:

| Profile | Token budget | Hot ratio | Best for |
|---|---|---|---|
| `general` | Balanced | 40% hot, 35% warm, 25% cold | General-purpose assistants |
| `coding` | Large | 50% hot, 30% warm, 20% cold | Code generation and review |
| `research` | Maximum | 30% hot, 35% warm, 35% cold | Long-document analysis |

## Apply a profile

Pass the profile name to `ContextManagementConfig` at agent creation:

=== "General (default)"

    ```python
    import cortex

    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
        orchestrator_model="gpt-4o",
    )

    agent = engine.create_agent(
        name="assistant",
        system_prompt="You are a helpful assistant.",
        context_config=cortex.ContextManagementConfig(profile="general"),
    )
    ```

=== "Coding"

    ```python
    agent = engine.create_agent(
        name="coder",
        system_prompt="You are an expert Python developer.",
        context_config=cortex.ContextManagementConfig(profile="coding"),  # (1)!
    )
    ```

    1. The `coding` profile allocates more tokens to the hot tier so recent code snippets stay in full context.

=== "Research"

    ```python
    agent = engine.create_agent(
        name="researcher",
        system_prompt="You analyze long documents thoroughly.",
        context_config=cortex.ContextManagementConfig(profile="research"),  # (1)!
    )
    ```

    1. The `research` profile maximizes cold storage to retain information from much earlier in the conversation.

## Memory tiers explained

The context engine organizes conversation history into three tiers:

```
Hot tier    -- Most recent turns. Kept verbatim. Full fidelity.
Warm tier   -- Older turns. May be lightly summarized.
Cold tier   -- Oldest turns. Heavily summarized or compressed.
```

As new messages arrive:

1. They enter the **hot** tier.
2. When the hot tier exceeds its budget, the oldest hot messages move to **warm**.
3. When warm exceeds its budget, the oldest warm messages move to **cold** (with summarization).
4. When cold exceeds its budget, the oldest cold messages are evicted.

!!! note
    The context engine automatically handles tier transitions. You do not need to manage eviction manually.

## Inspect context stats

During a session, check how the context engine is performing:

```python
session = agent.start_session(user_id="user_123")

# After several interactions...
response = await session.run("Summarize everything we discussed.")

stats = session.get_context_stats()
print(stats)  # Shows token usage per tier, eviction count, summarization events
```

## Pair profiles with model context windows

Choose your profile based on the model's context window:

| Model | Context window | Recommended profile |
|---|---|---|
| `gpt-4o` | 128k tokens | `general` or `coding` |
| `gpt-4o-mini` | 128k tokens | `general` |
| `gemini-2.5-pro` | 1M tokens | `research` |
| `gemini-2.5-flash` | 1M tokens | `coding` or `research` |
| Local models (8B) | 8-32k tokens | `general` |

!!! tip
    Gemini models with 1M token context windows work exceptionally well with the `research` profile. You get deep cold-tier memory without aggressive summarization.

## Complete example

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={"gemini": {"api_key": "AIza..."}},
        orchestrator_model="gemini-2.5-pro",
        worker_model="gemini-2.5-flash",
    )

    agent = engine.create_agent(
        name="analyst",
        system_prompt="You analyze financial reports in detail.",
        context_config=cortex.ContextManagementConfig(profile="research"),
    )

    session = agent.start_session(user_id="analyst_1")

    # Long conversation -- the context engine manages memory automatically
    await session.run("Here is our Q1 report: ...")
    await session.run("And the Q2 report: ...")
    await session.run("Compare Q1 and Q2 revenue trends.")

    # Check memory health
    stats = session.get_context_stats()
    print(f"Context stats: {stats}")

    await session.close()


asyncio.run(main())
```

---

## Next steps

- [Tune Agent Weights](weight-tuning.md) -- adjust behavioral weights alongside context configuration.
- [Control Temperature & Creativity](temperature.md) -- pair context profiles with temperature settings.
- [Monitor Your Agent](../advanced/observability.md) -- track context engine metrics in production.
