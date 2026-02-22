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
    orchestrator_model="gemini-3-pro-preview",   # (1)!
    worker_model="gemini-3-flash-preview",       # (2)!
)
```

1. State-of-the-art reasoning in the Gemini family. Ideal for orchestration, agentic tasks, and complex planning.
2. Pro-level intelligence at Flash speed. Excellent for tool calls and simple completions.

Common Gemini model choices:

| Model | Best for | Context window |
|---|---|---|
| `gemini-3-pro-preview` | State-of-the-art reasoning, agentic tasks, and coding | 1M tokens |
| `gemini-3-flash-preview` | Pro-level intelligence at Flash speed and pricing | 1M tokens |
| `gemini-2.5-pro` | Stable production fallback with strong reasoning | 1M tokens |
| `gemini-2.5-flash` | Stable, fast, cost-effective worker tasks | 1M tokens |

!!! tip
    Gemini models support up to 1 million tokens of context. This pairs well with the `research` context profile, which allocates larger token budgets. See [Configure Context Profiles](../config/context-profiles.md).

## Create an agent and run a message

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={"gemini": {"api_key": "AIza..."}},
        orchestrator_model="gemini-3-pro-preview",
        worker_model="gemini-3-flash-preview",
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

For production workloads that exceed the free-tier rate limits (25 RPM / 250 RPD on Tier-1 API keys), use Vertex AI for pay-as-you-go access with significantly higher throughput, SLA guarantees, and enterprise features like VPC Service Controls and data residency.

### Set up authentication

Vertex AI uses Google's Application Default Credentials (ADC). No API keys are needed -- the SDK discovers credentials automatically from your environment.

=== "Local development"

    ```bash
    gcloud auth application-default login
    ```

=== "Production (GKE, Cloud Run, Compute Engine)"

    Attach a service account to your workload. ADC discovers it automatically -- no code changes needed.

=== "CI/CD"

    Set the `GOOGLE_APPLICATION_CREDENTIALS` environment variable to point to a service account key file, or use workload identity federation for keyless auth.

### Configure the engine

Pass `vertex_ai=True` along with your GCP project ID:

```python
engine = cortex.Engine(
    providers={
        "gemini": {
            "vertex_ai": True,
            "project": "your-project-id",
            "location": "us-central1",   # optional, defaults to us-central1
        },
    },
    orchestrator_model="gemini-3-pro-preview",
    worker_model="gemini-3-flash-preview",
)
```

Everything else -- agents, sessions, tools, streaming -- works identically to API key mode. The same `google-genai` package handles both modes.

### When to use Vertex AI vs API key

| | API Key | Vertex AI |
|---|---|---|
| **Rate limits** | 25 RPM / 250 RPD (Tier-1) | Pay-as-you-go, much higher |
| **Billing** | Free tier, then per-request | GCP billing account |
| **Auth setup** | Copy-paste API key | `gcloud auth` or service account |
| **Enterprise features** | None | VPC-SC, CMEK, audit logs, data residency |
| **Best for** | Prototyping, demos, low-volume | Production, enterprise, high-volume |

!!! tip
    You can start with an API key for development and switch to Vertex AI for production by changing only the provider config -- no other code changes required.

## Multi-provider setup with OpenAI

You can register Gemini alongside OpenAI and route each role to a different provider:

```python
engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
        "gemini": {"api_key": "AIza..."},
    },
    orchestrator_model="gpt-4o",
    worker_model="gemini-3-flash-preview",
)
```

This sends planning and reasoning to OpenAI while using Gemini's fast models for tool execution. See [Switch Between Providers](switching-providers.md) for advanced routing strategies.

---

## Next steps

- [Connect to OpenAI](openai.md) -- add OpenAI as a second provider.
- [Connect to Anthropic Claude](anthropic.md) -- add Claude as a provider with extended thinking.
- [Use Local Models](local-models.md) -- run fully offline with Ollama or vLLM.
- [Switch Between Providers](switching-providers.md) -- configure failover and split routing.
