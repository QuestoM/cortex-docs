# LangChain Integration

Add corteX intelligence to any LangChain agent. Your `AgentExecutor` keeps handling LLM calls and tool routing - corteX adds a brain that monitors, learns, and adapts alongside it.

## Installation

```bash
pip install cortex-ai[langchain]
```

This installs `cortex-ai` plus `langchain` and `langchain-core` dependencies.

## Quick Start

```python
from corteX.integrations.langchain import CortexBrain

# Wrap your existing LangChain agent
brain = CortexBrain(
    agent_executor=my_agent,    # Any AgentExecutor or Runnable
    goal="Resolve customer support tickets",
)

# Run with brain monitoring
result = await brain.run("Customer says their account is locked")

# Inspect what the brain learned
print(result["goal_progress"])   # 0.0 to 1.0
print(result["drift_score"])     # 0.0 = on track, 1.0 = fully drifted
print(result["loop_detected"])   # True if repetitive behavior detected
print(result["brain_state"])     # Full brain metrics (30+ components)
```

## How It Works

`CortexBrain` wraps your `AgentExecutor` without modifying it:

1. **Callback injection** - A `CortexCallbackHandler` is added to your executor's callback list. It receives events from LangChain (tool calls, LLM completions, errors) and feeds them into the corteX brain.

2. **Goal tracking** - Before each run, the input is registered as a goal plan. After the run, the output is verified against the goal to compute progress and drift.

3. **Weight learning** - Every tool call updates Bayesian posteriors in the weight engine. Tools that succeed get higher selection probability.

4. **Brain monitoring** - In full mode, 30+ brain components process every event: prediction, plasticity, attention, calibration, bias detection, and more.

## Configuration

```python
brain = CortexBrain(
    agent_executor=my_agent,
    goal="Resolve customer support tickets",

    # Brain mode: "full" (default, 30+ components) or "lite" (3 core)
    brain_mode="full",

    # Max steps before forced stop (safety cap)
    max_steps=30,

    # Drift threshold - flag when agent wanders (0.0 to 1.0)
    drift_threshold=0.3,

    # Enable cross-run memory
    enable_memory=True,

    # Enable weight learning from tool outcomes
    enable_weights=True,
)
```

## Accessing Brain State

### During a Run

```python
result = await brain.run("What's the status of order #12345?")

# Result dict contains everything
print(result["output"])          # The agent's response text
print(result["steps"])           # Number of steps taken
print(result["tool_calls"])      # Number of tool invocations
print(result["tokens_used"])     # Token usage from LangChain callbacks
print(result["latency_ms"])      # Total run time in milliseconds
print(result["errors"])          # Any errors during the run
```

### Brain Properties

```python
# Quick checks (no run needed)
brain.drift_score       # Current drift from goal
brain.drift_exceeded    # True if drift > threshold
brain.goal_progress     # Progress toward goal (0..1)
brain.loop_detected     # Whether loops were detected

# Full brain state
brain.brain_state       # Dict with all 30+ component metrics

# Run history
brain.runs              # List of all past run records
```

### Direct Component Access

```python
# Access the IntegrationBrain directly
brain.brain                     # IntegrationBrain instance
brain.brain.get_component("prediction")    # Prediction engine
brain.brain.get_component("reputation")    # Tool reputation tracker
brain.brain.get_component("calibration")   # Metacognitive calibration

# Core components
brain.weight_engine     # WeightEngine - synaptic weights
brain.goal_tracker      # GoalTracker - goal DNA verification
brain.callback_handler  # CortexCallbackHandler - LangChain bridge
```

## Synchronous Usage

```python
# If you're not in an async context
result = brain.run_sync("Help me with my billing issue")
```

## Full Example

```python
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.tools import tool as langchain_tool

from corteX.integrations.langchain import CortexBrain

# 1. Define your LangChain tools
@langchain_tool
def search_orders(order_id: str) -> str:
    """Search for an order by ID."""
    return f"Order {order_id}: Shipped, arriving tomorrow"

@langchain_tool
def update_ticket(ticket_id: str, status: str) -> str:
    """Update a support ticket status."""
    return f"Ticket {ticket_id} updated to {status}"

# 2. Create your LangChain agent (as you normally would)
llm = ChatOpenAI(model="gpt-4o")
prompt = ChatPromptTemplate.from_messages([
    ("system", "You are a customer support agent."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])
agent = create_openai_tools_agent(llm, [search_orders, update_ticket], prompt)
executor = AgentExecutor(agent=agent, tools=[search_orders, update_ticket])

# 3. Wrap with corteX brain
brain = CortexBrain(
    agent_executor=executor,
    goal="Resolve customer support tickets efficiently",
)

# 4. Run with intelligence
result = await brain.run("Where is my order #12345?")
print(result["output"])
print(f"Goal progress: {result['goal_progress']:.0%}")
print(f"Drift score: {result['drift_score']:.2f}")
print(f"Tools used: {result['tool_calls']}")
```

## Migration from Standalone corteX

If you're moving from standalone corteX to LangChain + corteX brain:

| Standalone corteX | LangChain + corteX Brain |
|-------------------|-------------------------|
| `Engine()` creates providers | LangChain handles LLM calls |
| `engine.create_agent()` | `CortexBrain(agent_executor=...)` |
| `session.run()` | `brain.run()` |
| `session.get_weights()` | `brain.weight_engine` |
| Brain drives tool selection | LangChain drives tools, brain observes |

The key difference: in standalone mode, corteX drives everything. In integration mode, your framework drives - corteX observes and learns.
