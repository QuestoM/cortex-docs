# Connect to OpenAI

Configure OpenAI or Azure OpenAI as the LLM provider for your corteX agents.

---

## Set your API key

=== "Environment variable (recommended)"

    ```bash
    export OPENAI_API_KEY="sk-..."
    ```

=== "Passed directly"

    ```python
    import cortex

    engine = cortex.Engine(
        providers={
            "openai": {"api_key": "sk-proj-abc123..."},
        },
    )
    ```

!!! warning
    Never commit API keys to source control. Use environment variables or a secrets manager in production.

## Choose a model

Pass the model name to `orchestrator_model` and/or `worker_model`:

```python
engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4.1",          # (1)!
    worker_model="gpt-4.1-mini",           # (2)!
)
```

1. The orchestrator handles planning, goal tracking, and multi-step reasoning. GPT-4.1 excels at coding and instruction following with a 1M token context.
2. The worker handles simple completions and tool calls -- a smaller model keeps costs down while maintaining strong performance.

Common OpenAI model choices:

| Model | Best for | Context window |
|---|---|---|
| `gpt-4.1` | High-accuracy orchestration, coding, instruction following | 1M tokens |
| `gpt-4.1-mini` | Cost-effective worker tasks with strong performance | 1M tokens |
| `gpt-4.1-nano` | Ultra-fast, lightweight tasks | 1M tokens |
| `o3` | Complex reasoning and multi-step problems | 200k tokens |
| `o4-mini` | Cost-efficient reasoning tasks | 200k tokens |
| `gpt-4o` | General-purpose orchestration (legacy) | 128k tokens |
| `gpt-4o-mini` | Lightweight worker tasks (legacy) | 128k tokens |

## Use Azure OpenAI

Azure OpenAI deployments are supported through the same `openai` provider type. Supply your Azure-specific endpoint and deployment name:

```python
engine = cortex.Engine(
    providers={
        "openai": {
            "api_key": "your-azure-key",
            "base_url": "https://my-resource.openai.azure.com/openai/deployments/gpt-4o",
            "api_version": "2024-12-01-preview",
        },
    },
    orchestrator_model="gpt-4o",
)
```

!!! note
    The `base_url` must point to the full deployment URL including `/openai/deployments/<deployment-name>`. The model name you pass to `orchestrator_model` should match your Azure deployment name.

## Create an agent and run a message

Once the engine is configured, everything else works the same regardless of provider:

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
        orchestrator_model="gpt-4o",
    )

    agent = engine.create_agent(
        name="helper",
        system_prompt="You are a concise assistant.",
    )

    session = agent.start_session(user_id="user_123")
    response = await session.run("Explain Newton's first law in one sentence.")

    print(response.content)
    print(f"Model: {response.metadata.model_used}")
    print(f"Tokens: {response.metadata.tokens_used}")

    await session.close()


asyncio.run(main())
```

## Stream responses

For real-time output, use `run_stream`:

```python
async for chunk in session.run_stream("Write a haiku about recursion."):
    print(chunk.content, end="")
```

## Verify the connection

If you want to confirm your credentials are valid before creating agents, check the engine's provider list after construction. A misconfigured key will raise a `cortex.ProviderAuthError` on the first `session.run()` call.

!!! tip
    During development, set `orchestrator_model` and `worker_model` to the same model to simplify debugging. Split them later when optimizing cost.

---

## Next steps

- [Connect to Google Gemini](gemini.md) -- add a second provider for failover.
- [Connect to Anthropic Claude](anthropic.md) -- add Claude as a provider with extended thinking.
- [Switch Between Providers](switching-providers.md) -- route orchestrator and worker to different backends.
- [Tune Agent Weights](../config/weight-tuning.md) -- adjust how your agent behaves.
