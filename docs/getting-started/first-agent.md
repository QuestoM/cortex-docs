# Your First Agent

corteX uses a three-layer architecture: **Engine** creates **Agents**, Agents create **Sessions**. Each layer has a clear responsibility.

---

## Engine

The `Engine` is the root object. It registers LLM providers and holds global configuration. Create one per application.

```python
import cortex

engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
    },
    orchestrator_model="gpt-4o",   # Used for complex reasoning
    worker_model="gpt-4o-mini",    # Used for routine tasks
)
```

The engine's internal **LLM Router** (the "thalamus") decides which model handles each request based on dual-process routing: System 1 (fast/worker) for routine inputs, System 2 (slow/orchestrator) for novel or high-stakes ones.

!!! info "Multiple providers"
    Register as many providers as you need. The router can fail over between them automatically.

    ```python
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": "sk-..."},
            "gemini": {"api_key": "AIza..."},
            "local":  {"base_url": "http://localhost:11434/v1"},
        },
    )
    ```

---

## Agent

An `Agent` is a stateless template. It defines *who* the agent is -- its name, system prompt, tools, and configuration. One agent can serve many concurrent sessions.

```python
agent = engine.create_agent(
    name="support",
    system_prompt="You are a customer support agent for Acme Corp. "
                  "Be empathetic, concise, and always offer next steps.",
    goal_tracking=True,       # Track progress toward the user's goal
    loop_prevention=True,     # Detect and break repetitive patterns
)
```

### Configuration options

| Parameter | Type | Default | Description |
|---|---|---|---|
| `name` | `str` | required | Identifier for the agent. |
| `system_prompt` | `str` | `""` | Personality and instructions. |
| `tools` | `list` | `[]` | List of `@cortex.tool` wrappers. |
| `goal_tracking` | `bool` | `True` | Enable goal progress and drift detection. |
| `loop_prevention` | `bool` | `True` | Detect repetitive tool/response loops. |
| `weight_config` | `WeightConfig` | defaults | Behavioral weight presets. |
| `enterprise_config` | `EnterpriseConfig` | `None` | Safety, audit, and compliance settings. |
| `context_config` | `ContextManagementConfig` | `None` | Context window management profile. |

---

## Session

A `Session` is a live, stateful conversation between one agent and one user. When you call `start_session()`, corteX initializes all 20 brain components:

```python
session = agent.start_session(user_id="user_42")
```

Each session maintains its own:

- Conversation history
- Synaptic weight state (Bayesian posteriors)
- Goal tracker
- Memory fabric (working, episodic, semantic)
- Context engine (hot/warm/cold tiers)
- Tool reputation scores
- Prediction and calibration state

### The run loop

Call `session.run()` to send a message and receive a response. Each call executes the full 14-step brain pipeline.

```python
r1 = await session.run("I need to return an item I bought last week.")
print(r1.content)

r2 = await session.run("The order number is ORD-9281.")
print(r2.content)

r3 = await session.run("Thanks, that worked!")
print(r3.content)
```

The session learns across turns. Weights shift, predictions improve, and tool preferences sharpen -- all within the conversation.

### Closing a session

Always close sessions to consolidate state:

```python
stats = await session.close()
print(stats["turns"])          # Number of conversation turns
print(stats["total_tokens"])   # Total tokens consumed
```

---

## Complete example

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
    )

    agent = engine.create_agent(
        name="support",
        system_prompt="You are a helpful support agent for Acme Corp.",
    )

    session = agent.start_session(user_id="user_42")

    response = await session.run("How do I reset my password?")
    print(response.content)

    # Inspect the brain
    print(session.get_weights())             # Current weight state
    print(session.get_goal_progress())       # Goal tracking
    print(session.get_dual_process_stats())  # System 1/2 routing

    await session.close()


asyncio.run(main())
```

---

Next: [Adding Tools](adding-tools.md)
