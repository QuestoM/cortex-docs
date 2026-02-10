# Tune Agent Weights

Configure synaptic weights to control how your agent balances verbosity, formality, autonomy, initiative, and speed versus quality.

---

## What are weights?

Weights are floating-point values between 0.0 and 1.0 that shape agent behavior. They feed into the brain engine's decision pipeline and influence everything from response length to tool usage frequency.

| Weight | Low (0.0) | High (1.0) |
|---|---|---|
| `verbosity` | Terse, minimal answers | Detailed, thorough explanations |
| `formality` | Casual, conversational tone | Professional, structured tone |
| `autonomy` | Always asks for confirmation | Takes independent action |
| `initiative` | Responds only to direct questions | Proactively offers suggestions |
| `speed_vs_quality` | Prioritizes accuracy over speed | Prioritizes speed over accuracy |

## Set weights at agent creation

Pass a `WeightConfig` when creating the agent:

```python
import cortex

engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4o",
)

agent = engine.create_agent(
    name="formal_assistant",
    system_prompt="You are a professional enterprise assistant.",
    weight_config=cortex.WeightConfig(
        verbosity=0.6,
        formality=0.9,          # (1)!
        autonomy=0.3,
        initiative=0.4,
        speed_vs_quality=0.2,   # (2)!
    ),
)
```

1. High formality produces structured, professional language.
2. Low speed_vs_quality favors accuracy -- the engine spends more time reasoning.

## Override weights at runtime

You can adjust weights mid-session without restarting:

```python
session = agent.start_session(user_id="user_123")

# Start with default weights
response = await session.run("Explain microservices.")

# User wants more detail -- increase verbosity on the fly
session.override_weight("verbosity", 0.95)
response = await session.run("Go deeper on service mesh.")

# Check current weight snapshot
weights = session.get_weights()
print(weights)
```

!!! tip
    Runtime overrides take effect immediately on the next `session.run()` call. They persist for the lifetime of the session unless overridden again.

## Weight presets for common scenarios

=== "Customer support"

    ```python
    weight_config = cortex.WeightConfig(
        verbosity=0.5,
        formality=0.7,
        autonomy=0.6,
        initiative=0.8,    # Proactively suggest solutions
        speed_vs_quality=0.5,
    )
    ```

=== "Code review"

    ```python
    weight_config = cortex.WeightConfig(
        verbosity=0.7,
        formality=0.4,
        autonomy=0.8,     # Independently analyze code
        initiative=0.9,   # Flag issues without being asked
        speed_vs_quality=0.1,  # Accuracy is critical
    )
    ```

=== "Quick Q&A"

    ```python
    weight_config = cortex.WeightConfig(
        verbosity=0.2,         # Short answers
        formality=0.3,
        autonomy=0.5,
        initiative=0.2,
        speed_vs_quality=0.9,  # Speed matters most
    )
    ```

## Inspect weight state

At any point during a session, retrieve the full weight snapshot:

```python
weights = session.get_weights()
print(weights)
```

The brain engine continuously adjusts weights through its plasticity and feedback systems. The snapshot shows the **effective** weights after all internal learning adjustments.

## Use what-if simulation before committing

Before changing a weight in production, test the impact with the digital twin:

```python
result = session.simulate_what_if({"autonomy": 0.9, "verbosity": 0.2})
print(result)  # Shows predicted behavior changes
```

!!! warning
    Extreme weight values (close to 0.0 or 1.0) can cause polarized behavior. Test with `simulate_what_if` before deploying aggressive weight changes. See [Run What-If Simulations](../advanced/what-if-simulation.md).

## Learning rates and plasticity

The brain engine can learn optimal weights over time through its plasticity system. Each interaction adjusts weights slightly based on feedback signals (tool success, goal progress, user satisfaction).

- Weights drift toward effective values naturally over a session.
- Use `session.get_calibration_report()` to see how well the agent's confidence aligns with actual outcomes.
- Persist learned weights across sessions with [Weight Persistence](../advanced/weight-persistence.md).

---

## Next steps

- [Run What-If Simulations](../advanced/what-if-simulation.md) -- test weight changes before applying them.
- [Persist Weights Across Sessions](../advanced/weight-persistence.md) -- save learned weights.
- [Configure Context Profiles](context-profiles.md) -- control memory and context alongside weights.
- [Monitor Your Agent](../advanced/observability.md) -- track weight drift over time.
