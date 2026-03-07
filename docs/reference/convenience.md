# Convenience Functions

## Module: `corteX.sdk_convenience`

Shortcut functions that reduce boilerplate for common use cases. Auto-detect LLM providers from environment variables and create agents in one call.

Also available via `from corteX import quick_agent, detect_provider, auto_engine`.

---

## `quick_agent`

```python
def quick_agent(
    name: str = "agent",
    system_prompt: str = "You are a helpful assistant.",
    tools: Optional[List[ToolWrapper]] = None,
    user_id: str = "default",
    **engine_kwargs,
) -> Tuple[Agent, Session]
```

Create an agent and session in one call with auto-detected provider. The fastest way to go from zero to a working agent.

**Parameters**:

- `name` (`str`): Agent name. Default: `"agent"`
- `system_prompt` (`str`): System instructions. Default: `"You are a helpful assistant."`
- `tools` (`Optional[List[ToolWrapper]]`): List of `@tool`-decorated functions
- `user_id` (`str`): User identifier for the session. Default: `"default"`
- `**engine_kwargs`: Extra kwargs passed to `Engine()`

**Returns**: `Tuple[Agent, Session]` ready to use.

**Raises**: `ValueError` if no API key found in environment.

**Example**:

```python
from corteX import quick_agent, tool

@tool(name="lookup", description="Look up product info")
def lookup(query: str) -> str:
    return f"Premium Plan: $99/mo"

agent, session = quick_agent("support", "You help customers.", tools=[lookup])
response = await session.run("What is the Premium plan price?")
```

---

## `auto_engine`

```python
def auto_engine(**engine_kwargs) -> Engine
```

Create an `Engine` with auto-detected provider from environment variables. Checks `GEMINI_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` in order.

**Returns**: Configured `Engine` instance.

**Raises**: `ValueError` if no API key found.

**Example**:

```python
from corteX.sdk_convenience import auto_engine

engine = auto_engine()
agent = engine.create_agent(name="assistant", system_prompt="You help users.")
```

---

## `detect_provider`

```python
def detect_provider() -> Tuple[str, str]
```

Auto-detect an LLM provider from environment variables.

**Returns**: `(provider_name, api_key)` or `("", "")` if none found.

**Example**:

```python
from corteX.sdk_convenience import detect_provider

provider, api_key = detect_provider()
if provider:
    print(f"Using {provider}")
else:
    print("No API key found")
```
