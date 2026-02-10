# Persist Weights Across Sessions

Save learned weights so your agents improve over time, and restore them when starting new sessions.

---

## Why persist weights?

During a session, the brain engine continuously adjusts weights through its plasticity and feedback systems. By default, these learned values are lost when the session closes. Weight persistence lets you:

- **Carry over learning** from one session to the next.
- **Pre-configure agents** with weights that have been validated in production.
- **A/B test** different weight snapshots across user cohorts.

## Save weights from a session

At any point, retrieve the current weight snapshot and store it:

```python
import json
import cortex

engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4o",
)

agent = engine.create_agent(
    name="support",
    system_prompt="You help customers.",
    weight_config=cortex.WeightConfig(autonomy=0.5, formality=0.5),
    goal_tracking=True,
)

session = agent.start_session(user_id="user_123")

# Run several interactions so the engine can learn
await session.run("I need to return an item.")
await session.run("The item arrived damaged.")
await session.run("Order number is ORD-789.")

# Capture the learned weights
weights = session.get_weights()
print(f"Learned weights: {weights}")

# Save to disk
with open("learned_weights.json", "w") as f:
    json.dump(weights, f)

await session.close()
```

!!! tip
    Save weights after sessions where the agent performed well. Use `session.get_calibration_report()` to assess quality before persisting.

## Load weights into a new session

Read the saved weights and apply them as the starting configuration:

```python
import json
import cortex

# Load previously saved weights
with open("learned_weights.json") as f:
    saved_weights = json.load(f)

engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4o",
)

# Apply saved weights via WeightConfig
agent = engine.create_agent(
    name="support",
    system_prompt="You help customers.",
    weight_config=cortex.WeightConfig(**saved_weights),  # (1)!
)

session = agent.start_session(user_id="user_456")

# This session starts with the learned weights from the previous session
response = await session.run("I have a billing question.")
print(response.content)
```

1. Unpack the saved dictionary directly into `WeightConfig`. The keys match the weight parameter names.

## Cross-session learning workflow

A typical production workflow:

```
Session 1  -->  Learn weights  -->  Save snapshot
                                        |
Session 2  <--  Load snapshot  <--------+
                                        |
Session 2  -->  Learn more     -->  Save updated snapshot
                                        |
Session 3  <--  Load snapshot  <--------+
```

Each session starts from where the last one ended, creating a continuous learning curve.

## Validate before persisting

Always verify that a weight snapshot produces good behavior before promoting it:

```python
session = agent.start_session(user_id="validator")

# Run test scenarios
r1 = await session.run("Test question 1")
r2 = await session.run("Test question 2")

# Check quality signals
calibration = session.get_calibration_report()
goal_progress = session.get_goal_progress()
weights = session.get_weights()

print(f"Calibration: {calibration}")
print(f"Goal progress: {goal_progress}")

# Only save if calibration looks good
if calibration_is_acceptable(calibration):
    with open("validated_weights.json", "w") as f:
        json.dump(weights, f)
    print("Weights saved.")
else:
    print("Weights did not pass validation. Not saving.")

await session.close()
```

!!! warning
    Do not blindly persist weights from every session. A session with unusual inputs or edge cases may produce weights that perform poorly on normal traffic. Always validate.

## Per-user weight persistence

Store weights per user to provide personalized agent behavior:

```python
import json
from pathlib import Path

WEIGHT_DIR = Path("weights")
WEIGHT_DIR.mkdir(exist_ok=True)


def save_user_weights(user_id: str, weights: dict):
    path = WEIGHT_DIR / f"{user_id}.json"
    path.write_text(json.dumps(weights))


def load_user_weights(user_id: str) -> dict | None:
    path = WEIGHT_DIR / f"{user_id}.json"
    if path.exists():
        return json.loads(path.read_text())
    return None


# Usage
user_id = "user_123"
saved = load_user_weights(user_id)

if saved:
    weight_config = cortex.WeightConfig(**saved)
else:
    weight_config = cortex.WeightConfig()  # Defaults

agent = engine.create_agent(
    name="personal_assistant",
    system_prompt="You adapt to each user's preferences.",
    weight_config=weight_config,
)

session = agent.start_session(user_id=user_id)
# ... interact ...

# Save at end
save_user_weights(user_id, session.get_weights())
await session.close()
```

!!! note
    For production deployments, replace the file-based storage with a database or key-value store (Redis, DynamoDB, etc.) for durability and concurrent access.

## Use what-if simulation with persisted weights

Before deploying a new weight snapshot, simulate its effect:

```python
# Load candidate weights
with open("candidate_weights.json") as f:
    candidate = json.load(f)

# Simulate in a live session
result = session.simulate_what_if(candidate)
print(f"Predicted behavior: {result}")
```

---

## Next steps

- [Tune Agent Weights](../config/weight-tuning.md) -- understand what each weight controls.
- [Run What-If Simulations](what-if-simulation.md) -- test weight snapshots before deploying.
- [Monitor Your Agent](observability.md) -- track weight drift and calibration across sessions.
