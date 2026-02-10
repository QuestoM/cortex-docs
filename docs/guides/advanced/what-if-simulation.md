# Run What-If Simulations

Use the ComponentSimulator digital twin to test weight changes, configuration tweaks, and behavioral hypotheses before committing them to a live agent.

---

## Why simulate?

Changing weights or configuration on a live agent can produce unexpected behavior. The what-if simulator runs your proposed changes through a digital twin of the current session state and predicts the impact -- without affecting the real agent.

Use cases:

- Test a weight change before applying it.
- Compare two configurations side by side.
- Validate that an extreme setting will not cause drift.

## Run a basic simulation

Call `simulate_what_if()` with a dictionary of weight overrides:

```python
import cortex

engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4o",
)

agent = engine.create_agent(
    name="assistant",
    system_prompt="You are a helpful assistant.",
    weight_config=cortex.WeightConfig(autonomy=0.5, formality=0.5),
)

session = agent.start_session(user_id="user_123")

# Interact first to build up session state
await session.run("Hello, I need help with my account.")
await session.run("Can you check my billing history?")

# Now simulate: what if we cranked autonomy to 0.9?
result = session.simulate_what_if({"autonomy": 0.9})
print(result)
```

The result describes the predicted behavioral shift -- for example, whether the agent would take independent action versus asking for confirmation.

## Simulate multiple weights at once

Pass any combination of weights:

```python
result = session.simulate_what_if({
    "autonomy": 0.9,
    "verbosity": 0.2,
    "formality": 0.8,
})
print(result)
```

!!! tip
    Simulate the exact change you plan to make. Testing individual weights in isolation may not capture interaction effects between multiple weight changes.

## A/B test two configurations

Run two simulations and compare the outputs:

```python
# Configuration A: cautious and verbose
result_a = session.simulate_what_if({
    "autonomy": 0.3,
    "verbosity": 0.9,
})

# Configuration B: autonomous and terse
result_b = session.simulate_what_if({
    "autonomy": 0.9,
    "verbosity": 0.2,
})

print("Config A:", result_a)
print("Config B:", result_b)
```

This gives you a structured comparison before you commit to either approach.

## Apply the simulated change

Once you are satisfied with the simulation result, apply the change using `override_weight`:

```python
# Simulation showed good results -- apply it
session.override_weight("autonomy", 0.9)

# Verify with live request
response = await session.run("Check my last three invoices.")
print(response.content)
print(f"Drift score: {response.metadata.drift_score}")
```

!!! warning
    `simulate_what_if` does not modify the session. You must explicitly call `override_weight` to apply changes. The simulation is read-only.

## Monitor drift after changes

After applying a weight change, monitor the drift score to ensure the agent stays on track:

```python
response = await session.run("Summarize my account status.")
print(f"Drift score: {response.metadata.drift_score}")  # (1)!

# Get full goal progress
progress = session.get_goal_progress()
print(f"Goal progress: {progress}")
```

1. A drift score close to 0.0 means the agent is on track. Scores above 0.5 suggest the agent is deviating from its intended behavior.

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
        name="support",
        system_prompt="You help customers resolve account issues.",
        weight_config=cortex.WeightConfig(
            autonomy=0.5,
            formality=0.5,
            verbosity=0.5,
        ),
        goal_tracking=True,
    )

    session = agent.start_session(user_id="user_123")

    # Build session context
    await session.run("I need to update my payment method.")
    await session.run("My card expired last week.")

    # Simulate a more autonomous configuration
    sim_result = session.simulate_what_if({"autonomy": 0.8, "initiative": 0.9})
    print(f"Simulation: {sim_result}")

    # Looks good -- apply it
    session.override_weight("autonomy", 0.8)
    session.override_weight("initiative", 0.9)

    # Continue with new behavior
    response = await session.run("What should I do next?")
    print(response.content)
    print(f"Drift: {response.metadata.drift_score}")

    await session.close()


asyncio.run(main())
```

---

## Next steps

- [Tune Agent Weights](../config/weight-tuning.md) -- understand all available weights before simulating.
- [Override with Targeted Modulation](modulation-overrides.md) -- force tool-level changes alongside weight changes.
- [Persist Weights Across Sessions](weight-persistence.md) -- save a validated configuration for reuse.
- [Monitor Your Agent](observability.md) -- track the long-term impact of configuration changes.
