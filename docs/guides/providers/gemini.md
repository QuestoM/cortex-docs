# Connect to Google Gemini

Configure Google Gemini as the LLM provider for your corteX agents.

---

## Set your API key

=== "Environment variable (recommended)"

    ```bash
    export GEMINI_API_KEY="AIza..."
    ```

=== "Passed directly"

    ```python
    import cortex

    engine = cortex.Engine(
        providers={
            "gemini": {"api_key": "AIzaSy..."},
        },
    )
    ```

!!! warning
    Keep API keys out of version control. Use environment variables or a secrets manager for all deployed environments.

## Choose a model

Gemini offers models optimized for different cost and capability trade-offs:

```python
engine = cortex.Engine(
    providers={"gemini": {"api_key": "AIza..."}},
    orchestrator_model="gemini-2.5-pro",     # (1)!
    worker_model="gemini-2.5-flash",         # (2)!
)
```

1. Best reasoning capability in the Gemini family. Ideal for orchestration and complex planning.
2. Optimized for speed and cost. Excellent for tool calls and simple completions.

Common Gemini model choices:

| Model | Best for | Context window |
|---|---|---|
| `gemini-2.5-pro` | High-accuracy orchestration and reasoning | 1M tokens |
| `gemini-2.5-flash` | Fast, cost-effective worker tasks | 1M tokens |
| `gemini-2.0-flash` | Balanced speed and quality | 1M tokens |

!!! tip
    Gemini models support up to 1 million tokens of context. This pairs well with the `research` context profile, which allocates larger token budgets. See [Configure Context Profiles](../config/context-profiles.md).

## Create an agent and run a message

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={"gemini": {"api_key": "AIza..."}},
        orchestrator_model="gemini-2.5-pro",
        worker_model="gemini-2.5-flash",
    )

    agent = engine.create_agent(
        name="researcher",
        system_prompt="You are a thorough research assistant.",
    )

    session = agent.start_session(user_id="user_123")
    response = await session.run("Summarize the key ideas in transformer architectures.")

    print(response.content)
    print(f"Model: {response.metadata.model_used}")
    print(f"Latency: {response.metadata.latency_ms:.0f}ms")

    await session.close()


asyncio.run(main())
```

## Stream responses

Streaming works identically across all providers:

```python
async for chunk in session.run_stream("List five applications of reinforcement learning."):
    print(chunk.content, end="")
```

## Use Gemini with Vertex AI

For Google Cloud deployments using Vertex AI, supply your project and location:

```python
engine = cortex.Engine(
    providers={
        "gemini": {
            "project": "my-gcp-project",
            "location": "us-central1",
        },
    },
    orchestrator_model="gemini-2.5-pro",
)
```

!!! note
    Vertex AI authentication uses Application Default Credentials (ADC). Run `gcloud auth application-default login` locally, or attach a service account in production.

## Multi-provider setup with OpenAI

You can register Gemini alongside OpenAI and route each role to a different provider:

```python
engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
        "gemini": {"api_key": "AIza..."},
    },
    orchestrator_model="gpt-4o",
    worker_model="gemini-2.5-flash",
)
```

This sends planning and reasoning to OpenAI while using Gemini's fast models for tool execution. See [Switch Between Providers](switching-providers.md) for advanced routing strategies.

---

## Next steps

- [Connect to OpenAI](openai.md) -- add OpenAI as a second provider.
- [Connect to Anthropic Claude](anthropic.md) -- add Claude as a provider with extended thinking.
- [Use Local Models](local-models.md) -- run fully offline with Ollama or vLLM.
- [Switch Between Providers](switching-providers.md) -- configure failover and split routing.
