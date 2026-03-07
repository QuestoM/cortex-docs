# OpenAI Wrapper

Add corteX intelligence to raw OpenAI chat completion calls. This wrapper does NOT replace the OpenAI client - it adds brain monitoring to every completion so your function-calling loops gain goal tracking, loop detection, and weight learning.

## Installation

```bash
pip install cortex-ai[openai]
```

## Quick Start

```python
from corteX.integrations.openai_wrapper import CortexOpenAI

client = CortexOpenAI(
    api_key="sk-...",
    goal="Help user manage calendar events",
)

result = await client.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Schedule a meeting for tomorrow at 2pm"}],
)

print(result["content"])
print(result["brain_state"])     # 30+ brain component metrics
print(result["goal_progress"])
print(result["drift_score"])
```

## How It Works

`CortexOpenAI` creates both sync and async OpenAI clients internally, then wraps every `create()` call:

1. **Goal registration** - The user message is extracted and registered as a goal plan step.
2. **Completion** - The OpenAI client runs the chat completion (with optional tool loop).
3. **Goal verification** - The output is verified against the goal to compute progress and drift.
4. **Brain metrics** - All 30+ brain components process the interaction and update their state.

### Tool Loop Support

When you provide `tools` and a `tool_executor`, the wrapper runs an automatic function-calling loop:

```python
async def execute_tool(function_name: str, arguments_json: str) -> str:
    """Your tool executor - called for each function call."""
    if function_name == "get_calendar":
        return '{"events": [{"title": "Team standup", "time": "10:00"}]}'
    return "Unknown tool"

result = await client.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "What meetings do I have today?"}],
    tools=[{
        "type": "function",
        "function": {
            "name": "get_calendar",
            "description": "Get calendar events",
            "parameters": {"type": "object", "properties": {
                "date": {"type": "string"}
            }},
        },
    }],
    tool_executor=execute_tool,
)
```

The loop runs up to `max_loops` iterations (default 15), with corteX monitoring each iteration for loops and drift.

## Configuration

```python
client = CortexOpenAI(
    api_key="sk-...",
    goal="Help user manage calendar events",

    # Custom base URL (Azure, vLLM, Ollama, LM Studio)
    base_url="https://my-endpoint.openai.azure.com/",

    # Max function-calling loop iterations
    max_loops=15,

    # Brain mode: "full" (default, 30+ components) or "lite" (3 core)
    brain_mode="full",
)
```

### Works with Any OpenAI-Compatible Endpoint

```python
# Azure OpenAI
client = CortexOpenAI(
    api_key="azure-key",
    goal="...",
    base_url="https://my-resource.openai.azure.com/",
)

# vLLM / Ollama / LM Studio
client = CortexOpenAI(
    api_key="not-needed",
    goal="...",
    base_url="http://localhost:8000/v1",
)
```

## Accessing Brain State

```python
# Properties
client.drift_score       # Current drift from goal
client.goal_progress     # Progress toward goal (0..1)
client.loop_detected     # Whether loops were detected
client.brain_state       # Full brain metrics dict
client.runs              # History of all completions

# Direct brain access
client.brain             # IntegrationBrain instance
client.brain.get_component("prediction")
client.brain.get_component("bayesian_selector")

# Core components
client.weight_engine     # WeightEngine
client.goal_tracker      # GoalTracker
client.callback_handler  # CortexOpenAICallbackHandler
```

## Synchronous Usage

```python
result = client.create_sync(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Cancel my 3pm meeting"}],
)
```

## Full Example

```python
import asyncio
from corteX.integrations.openai_wrapper import CortexOpenAI

# 1. Create the wrapper
client = CortexOpenAI(
    api_key="sk-...",
    goal="Help users manage their calendar efficiently",
)

# 2. Define tools
tools = [{
    "type": "function",
    "function": {
        "name": "list_events",
        "description": "List calendar events for a date",
        "parameters": {
            "type": "object",
            "properties": {
                "date": {"type": "string", "description": "Date in YYYY-MM-DD format"},
            },
            "required": ["date"],
        },
    },
}, {
    "type": "function",
    "function": {
        "name": "create_event",
        "description": "Create a new calendar event",
        "parameters": {
            "type": "object",
            "properties": {
                "title": {"type": "string"},
                "date": {"type": "string"},
                "time": {"type": "string"},
            },
            "required": ["title", "date", "time"],
        },
    },
}]

# 3. Tool executor
async def execute(name: str, args: str) -> str:
    import json
    params = json.loads(args)
    if name == "list_events":
        return '{"events": [{"title": "Standup", "time": "10:00"}]}'
    if name == "create_event":
        return f"Created: {params['title']} on {params['date']} at {params['time']}"
    return "Unknown tool"

# 4. Run with brain monitoring
result = await client.create(
    model="gpt-4o",
    messages=[
        {"role": "system", "content": "You help manage calendars."},
        {"role": "user", "content": "Schedule a team lunch tomorrow at noon"},
    ],
    tools=tools,
    tool_executor=execute,
)

print(result["content"])
print(f"Goal progress: {result['goal_progress']:.0%}")
print(f"Drift: {result['drift_score']:.2f}")
print(f"Loops detected: {result['loop_detected']}")

# 5. Check brain state
state = client.brain_state
print(f"Brain mode: {state['mode']}")
print(f"Components: {len(state.get('full_components', []))}")
```

## Key Differences from Other Integrations

| Feature | OpenAI Wrapper | LangChain / CrewAI |
|---------|---------------|-------------------|
| LLM client | Built-in (creates OpenAI client) | External (you provide executor/crew) |
| Tool routing | Built-in function-calling loop | Handled by framework |
| Agent abstraction | None (raw completions) | Full agent with memory |
| Best for | Simple tool loops, prototyping | Production multi-step agents |
| Configuration | API key + goal | Framework setup + goal |
