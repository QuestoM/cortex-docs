# Connect to Anthropic Claude

Configure Anthropic Claude as the LLM provider for your corteX agents.

---

## Set your API key

=== "Environment variable (recommended)"

    ```bash
    export ANTHROPIC_API_KEY="sk-ant-..."
    ```

=== "Passed directly"

    ```python
    import cortex

    engine = cortex.Engine(
        providers={
            "anthropic": {"api_key": "sk-ant-api03-..."},
        },
    )
    ```

!!! warning
    Never commit API keys to source control. Use environment variables or a secrets manager in production.

## Choose a model

Pass the model name to `orchestrator_model` and/or `worker_model`:

```python
engine = cortex.Engine(
    providers={"anthropic": {"api_key": "sk-ant-..."}},
    orchestrator_model="claude-opus-4-6",      # (1)!
    worker_model="claude-haiku-4-5",           # (2)!
)
```

1. The orchestrator handles planning, goal tracking, and multi-step reasoning. Claude Opus 4.6 is the highest-accuracy option.
2. The worker handles simple completions and tool calls. Haiku keeps costs down with fast responses.

Common Claude model choices:

| Model | Best for | Context window |
|---|---|---|
| `claude-opus-4-6` | Highest-accuracy orchestration and complex reasoning | 200k tokens |
| `claude-sonnet-4-5` | Balanced quality and speed (recommended default) | 200k tokens |
| `claude-haiku-4-5` | Cost-effective worker tasks, fast responses | 200k tokens |

!!! tip
    `claude-sonnet-4-5` is the best all-round choice for most applications. Use Opus for orchestration when accuracy is critical, and Haiku for worker tasks when speed matters.

## Extended thinking

Claude supports an extended thinking mode where the model reasons step-by-step before responding. This is especially useful for complex planning and analysis tasks:

```python
engine = cortex.Engine(
    providers={
        "anthropic": {
            "api_key": "sk-ant-...",
            "default_model": "claude-sonnet-4-5",
        },
    },
    orchestrator_model="claude-sonnet-4-5",
)
```

Extended thinking is managed automatically by the brain engine. When the system detects high uncertainty or complex tasks, it enables thinking mode with an appropriate budget.

## Create an agent and run a message

Once the engine is configured, everything else works the same regardless of provider:

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={"anthropic": {"api_key": "sk-ant-..."}},
        orchestrator_model="claude-sonnet-4-5",
        worker_model="claude-haiku-4-5",
    )

    agent = engine.create_agent(
        name="analyst",
        system_prompt="You are a thorough business analyst.",
    )

    session = agent.start_session(user_id="user_123")
    response = await session.run("Analyze the pros and cons of microservices architecture.")

    print(response.content)
    print(f"Model: {response.metadata.model_used}")
    print(f"Tokens: {response.metadata.tokens_used}")

    await session.close()


asyncio.run(main())
```

## Stream responses

Streaming works identically across all providers:

```python
async for chunk in session.run_stream("Explain quantum computing in simple terms."):
    print(chunk.content, end="")
```

## Use Claude via Amazon Bedrock

For production workloads on AWS, you can access Claude through [Amazon Bedrock](https://aws.amazon.com/bedrock/). Bedrock uses your existing AWS IAM credentials -- no Anthropic API key is needed.

### Set up authentication

Bedrock uses the standard AWS credential chain. The SDK discovers credentials automatically from your environment.

=== "Local development"

    ```bash
    aws configure
    # or
    export AWS_ACCESS_KEY_ID="..."
    export AWS_SECRET_ACCESS_KEY="..."
    ```

=== "Production (ECS, EKS, Lambda)"

    Attach an IAM role to your workload. The AWS SDK discovers it automatically -- no code changes needed.

=== "CI/CD"

    Use IAM roles with OIDC federation, or set `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` environment variables.

### Configure the engine

Pass `bedrock=True` along with your preferred AWS region:

```python
engine = cortex.Engine(
    providers={
        "anthropic": {
            "bedrock": True,
            "aws_region": "us-west-2",   # optional, defaults to us-east-1
            "default_model": "anthropic.claude-sonnet-4-5-20250514-v1:0",
        },
    },
    orchestrator_model="anthropic.claude-sonnet-4-5-20250514-v1:0",
)
```

!!! note
    Bedrock model IDs follow the `anthropic.claude-*` naming convention and include a version suffix. Check the [AWS Bedrock documentation](https://docs.aws.amazon.com/bedrock/latest/userguide/models-supported.html) for current model IDs.

### Install the Bedrock extra

```bash
pip install 'anthropic[bedrock]'
```

## Use Claude via Google Vertex AI

You can also access Claude through [Google Vertex AI](https://cloud.google.com/vertex-ai), which provides enterprise features like VPC Service Controls, CMEK, and data residency. Authentication uses Google's Application Default Credentials (ADC).

### Set up authentication

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
        "anthropic": {
            "vertex_ai": True,
            "project": "your-project-id",
            "location": "us-east5",   # optional, defaults to us-east5
            "default_model": "claude-sonnet-4-5@20250514",
        },
    },
    orchestrator_model="claude-sonnet-4-5@20250514",
)
```

!!! note
    Vertex AI model IDs use the `model@date` format. Check the [Vertex AI Model Garden](https://cloud.google.com/vertex-ai/generative-ai/docs/partner-models/use-claude) for current model names and supported regions.

### Install the Vertex extra

```bash
pip install 'anthropic[vertex]'
```

### When to use API key vs Bedrock vs Vertex AI

| | API Key | Amazon Bedrock | Google Vertex AI |
|---|---|---|---|
| **Auth** | API key | AWS IAM | Google ADC |
| **Rate limits** | Per-tier (Anthropic) | AWS account limits | GCP quota |
| **Billing** | Anthropic billing | AWS billing | GCP billing |
| **Enterprise features** | Limited | VPC, CloudTrail, KMS | VPC-SC, CMEK, audit logs |
| **Best for** | Prototyping, direct usage | AWS-native deployments | GCP-native deployments |

!!! tip
    You can start with an API key for development and switch to Bedrock or Vertex AI for production by changing only the provider config -- no other code changes required.

## Multi-provider setup

You can register Claude alongside other providers and route each role to a different backend:

```python
engine = cortex.Engine(
    providers={
        "anthropic": {"api_key": "sk-ant-...", "default_model": "claude-sonnet-4-5"},
        "gemini": {"api_key": "AIza...", "default_model": "gemini-3-pro-preview"},
    },
    orchestrator_model="claude-opus-4-6",
    worker_model="gemini-3-flash-preview",
)
```

This sends planning and reasoning to Claude while using Gemini's fast models for tool execution. See [Switch Between Providers](switching-providers.md) for advanced routing strategies.

## Verify the connection

If you want to confirm your credentials are valid before creating agents, check the engine's provider list after construction. A misconfigured key will raise a `cortex.ProviderAuthError` on the first `session.run()` call.

!!! tip
    During development, set `orchestrator_model` and `worker_model` to the same model to simplify debugging. Split them later when optimizing cost.

---

## Next steps

- [Connect to OpenAI](openai.md) -- add OpenAI as a second provider.
- [Connect to Google Gemini](gemini.md) -- add Gemini as a second provider.
- [Switch Between Providers](switching-providers.md) -- configure failover and split routing.
- [Tune Agent Weights](../config/weight-tuning.md) -- adjust how your agent behaves.
