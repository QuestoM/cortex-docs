# Quick Start

A working agent in 60 seconds. corteX auto-detects your API key.

---

## 1. Install

```bash
pip install cortex-ai
export GEMINI_API_KEY="your-key"  # or OPENAI_API_KEY / ANTHROPIC_API_KEY
```

## 2. Run

```python
import asyncio
from corteX import quick_agent, tool

@tool(name="lookup", description="Look up product info")
def lookup(query: str) -> str:
    return f"Result for '{query}': Premium Plan - $99/mo"

agent, session = quick_agent("support", "You help customers.", tools=[lookup])

response = asyncio.run(session.run("What is the Premium plan price?"))
print(response.content)
print(f"Goal progress: {response.metadata.goal_progress:.0%}")
print(f"Drift score:   {response.metadata.drift_score:.2f}")
print(f"Tools called:  {response.metadata.tools_called}")
```

## 3. See the output

```text
The Premium plan costs $99 per month.

Goal progress: 100%
Drift score:   0.05
Tools called:  ['lookup']
```

The brain automatically tracked goal progress, measured drift from the original question, and selected the right tool.

---

## What just happened

| Step | Object | Purpose |
|---|---|---|
| `quick_agent(...)` | `Agent` + `Session` | Creates engine, agent, and session with auto-detected provider. |
| `@tool(...)` | `ToolWrapper` | Registers a function as a tool the agent can call. |
| `session.run(...)` | `Response` | Executes the 14-step brain pipeline and returns a response. |

## Response metadata

Every `Response` carries metadata about what happened inside the brain:

```python
print(response.metadata.goal_progress)  # 0.0 to 1.0 - how close to completing the goal
print(response.metadata.drift_score)    # 0.0 to 1.0 - how far the agent drifted from the goal
print(response.metadata.loop_detected)  # True if a loop was detected and escaped
print(response.metadata.model_used)     # e.g. "gemini-2.5-flash"
print(response.metadata.tokens_used)    # e.g. 384
print(response.metadata.latency_ms)     # e.g. 1247.3
print(response.metadata.tools_called)   # e.g. ["lookup"]
```

## Full control version

For advanced configuration, use the Engine/Agent/Session pattern directly:

```python
from corteX.sdk import Engine

engine = Engine(providers={
    "openai": {"api_key": "sk-..."},
    "gemini": {"api_key": "AIza..."},
})

agent = engine.create_agent(
    name="assistant",
    system_prompt="You are a helpful assistant.",
    goal_tracking=True,       # verify every step against the goal
    loop_prevention=True,     # detect and escape infinite loops
)

session = agent.start_session(user_id="user-1")
response = await session.run("What is the capital of France?")
```

---

Next: [Your First Agent](first-agent.md)
