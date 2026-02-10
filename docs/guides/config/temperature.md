# Control Temperature & Creativity

Understand how the dual-process router selects temperature automatically, and learn to apply manual overrides when you need precise control.

---

## How temperature works in corteX

Unlike a static temperature setting, corteX uses a **dual-process router** inspired by cognitive science:

- **System 1 (fast)** -- Low temperature. Used for factual recall, structured outputs, and tool calls.
- **System 2 (slow)** -- Higher temperature. Used for creative writing, brainstorming, and open-ended reasoning.

The router analyzes each incoming message and selects the appropriate system automatically. You do not need to set temperature manually for most use cases.

## Inspect routing decisions

Check which system handled a particular request:

```python
import cortex

engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4o",
)

agent = engine.create_agent(
    name="assistant",
    system_prompt="You are a versatile assistant.",
)

session = agent.start_session(user_id="user_123")

# Factual question -- likely routes to System 1
response = await session.run("What is the speed of light?")
print(f"Model: {response.metadata.model_used}")

# Creative question -- likely routes to System 2
response = await session.run("Write a poem about quantum entanglement.")
print(f"Model: {response.metadata.model_used}")

# See the routing stats
dp_stats = session.get_dual_process_stats()
print(dp_stats)
```

!!! note
    The dual-process router does not literally change the LLM temperature parameter in all cases. It selects between different processing pipelines that may include different prompting strategies, model choices, and confidence thresholds.

## How auto-tuning works

The brain engine adjusts temperature sensitivity over time based on:

1. **Task type detection** -- Code generation gets lower effective temperature than brainstorming.
2. **Goal progress** -- When the agent is close to completing a goal, temperature decreases to avoid drift.
3. **Calibration feedback** -- If the agent is overconfident, temperature increases slightly to explore more options.
4. **Drift score** -- If `response.metadata.drift_score` is high, the engine tightens temperature to stay on track.

Check the calibration report to see how well temperature auto-tuning is performing:

```python
calibration = session.get_calibration_report()
print(calibration)
```

## Influence temperature through weights

The `speed_vs_quality` weight indirectly affects temperature behavior:

=== "Favor accuracy (lower temperature)"

    ```python
    agent = engine.create_agent(
        name="precise",
        system_prompt="You give exact, verified answers.",
        weight_config=cortex.WeightConfig(
            speed_vs_quality=0.1,  # (1)!
        ),
    )
    ```

    1. Low speed_vs_quality tells the engine to spend more time reasoning, which correlates with lower effective temperature.

=== "Favor creativity (higher temperature)"

    ```python
    agent = engine.create_agent(
        name="creative",
        system_prompt="You brainstorm creative ideas.",
        weight_config=cortex.WeightConfig(
            speed_vs_quality=0.9,  # (1)!
        ),
    )
    ```

    1. High speed_vs_quality allows faster, more exploratory responses with higher effective temperature.

## Simulate temperature changes

Use the what-if simulator to predict how a weight change will affect routing:

```python
result = session.simulate_what_if({"speed_vs_quality": 0.9})
print(result)  # Shows predicted shift toward System 1 or System 2
```

!!! tip
    The `simulate_what_if` method lets you preview temperature-sensitive behavior without committing to a change. See [Run What-If Simulations](../advanced/what-if-simulation.md).

## Complete example

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
        orchestrator_model="gpt-4o",
        worker_model="gpt-4o-mini",
    )

    agent = engine.create_agent(
        name="writer",
        system_prompt="You are a technical writer who can also brainstorm.",
        weight_config=cortex.WeightConfig(
            speed_vs_quality=0.5,  # Balanced default
        ),
    )

    session = agent.start_session(user_id="user_123")

    # System 1 task: factual
    r1 = await session.run("List the HTTP status codes in the 4xx range.")
    print(f"Drift score: {r1.metadata.drift_score}")

    # System 2 task: creative
    r2 = await session.run("Brainstorm five names for an API monitoring product.")
    print(f"Drift score: {r2.metadata.drift_score}")

    # Compare routing
    print(session.get_dual_process_stats())

    await session.close()


asyncio.run(main())
```

!!! warning
    Avoid forcing extreme creativity settings on tasks that require factual accuracy (e.g., code generation, data extraction). The dual-process router exists to prevent this mismatch.

---

## Next steps

- [Tune Agent Weights](weight-tuning.md) -- `speed_vs_quality` is the primary lever for temperature behavior.
- [Run What-If Simulations](../advanced/what-if-simulation.md) -- test temperature-affecting changes safely.
- [Monitor Your Agent](../advanced/observability.md) -- track dual-process routing patterns.
