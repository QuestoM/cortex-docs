# Switch Between Providers

Set up multi-provider routing so your orchestrator and worker use different LLM backends, with automatic failover when a provider is unavailable.

---

## Register multiple providers

Pass every provider you want available to the `Engine`:

```python
import cortex

engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
        "gemini": {"api_key": "AIza..."},
        "anthropic": {"api_key": "sk-ant-..."},
        "local": {"base_url": "http://localhost:11434/v1"},
    },
    orchestrator_model="claude-opus-4-6",
    worker_model="gemini-3-flash-preview",
)
```

corteX identifies the correct provider for each model automatically. In the example above, `claude-opus-4-6` routes to Anthropic and `gemini-3-flash-preview` routes to Gemini.

## Split orchestrator and worker

The **orchestrator** handles planning, goal decomposition, and multi-step reasoning. The **worker** handles individual completions, tool calls, and sub-tasks. Splitting them lets you optimize for accuracy where it matters and speed where it does not.

=== "Accuracy-first"

    ```python
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": "sk-..."},
            "gemini": {"api_key": "AIza..."},
        },
        orchestrator_model="gpt-4o",           # Best reasoning
        worker_model="gemini-2.5-flash",       # Fast execution
    )
    ```

=== "Cost-first"

    ```python
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": "sk-..."},
            "gemini": {"api_key": "AIza..."},
        },
        orchestrator_model="gemini-2.5-flash", # Cheap planning
        worker_model="gemini-2.5-flash",       # Cheap execution
    )
    ```

=== "Claude + Gemini"

    ```python
    engine = cortex.Engine(
        providers={
            "anthropic": {"api_key": "sk-ant-..."},
            "gemini": {"api_key": "AIza..."},
        },
        orchestrator_model="claude-opus-4-6",  # Best reasoning
        worker_model="gemini-3-flash-preview", # Fast execution
    )
    ```

=== "Privacy-first"

    ```python
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": "sk-..."},
            "local": {"base_url": "http://localhost:11434/v1"},
        },
        orchestrator_model="gpt-4o",           # Cloud planning (no PII)
        worker_model="llama3.1:8b",            # Local execution (PII stays on-prem)
    )
    ```

## Automatic failover

When multiple providers are registered, corteX fails over automatically. If the primary provider returns an error or times out, the engine retries with the next available provider that supports a compatible model.

```python
engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
        "gemini": {"api_key": "AIza..."},
    },
    orchestrator_model="gpt-4o",
    worker_model="gemini-3-flash-preview",
)

agent = engine.create_agent(
    name="resilient",
    system_prompt="You are a reliable assistant.",
)

session = agent.start_session(user_id="user_123")

# If OpenAI is down, corteX retries with Gemini automatically
response = await session.run("What is the capital of France?")

print(response.metadata.model_used)  # Shows which model actually served the request
```

!!! tip
    Check `response.metadata.model_used` after each call to see which provider handled the request. This is especially useful for monitoring failover events in production.

## Inspect routing decisions

Use session stats to understand how calls are being routed:

```python
session = agent.start_session(user_id="user_123")

response = await session.run("Analyze this dataset.")
print(f"Model used: {response.metadata.model_used}")
print(f"Latency:    {response.metadata.latency_ms:.0f}ms")
print(f"Tokens:     {response.metadata.tokens_used}")

# Dual-process stats show whether System 1 (fast) or System 2 (slow) was used
dp_stats = session.get_dual_process_stats()
print(f"Routing:    {dp_stats}")
```

## Complete example

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": "sk-..."},
            "gemini": {"api_key": "AIza..."},
        },
        orchestrator_model="gpt-4o",
        worker_model="gemini-2.5-flash",
    )

    agent = engine.create_agent(
        name="support",
        system_prompt="You help customers with order issues.",
        goal_tracking=True,
    )

    session = agent.start_session(user_id="customer_42")

    response = await session.run("Where is my order #1234?")
    print(response.content)
    print(f"Served by: {response.metadata.model_used}")

    await session.close()


asyncio.run(main())
```

!!! note
    Provider failover is transparent to the agent and session. The system prompt, tools, and weights remain identical regardless of which provider serves the request.

---

## Next steps

- [Connect to OpenAI](openai.md) -- detailed OpenAI and Azure configuration.
- [Connect to Google Gemini](gemini.md) -- detailed Gemini and Vertex AI configuration.
- [Connect to Anthropic Claude](anthropic.md) -- detailed Claude configuration with extended thinking.
- [Use Local Models](local-models.md) -- add an on-premises provider to your mix.
- [Monitor Your Agent](../advanced/observability.md) -- track which providers serve each request.
