# Use Local Models

Run corteX agents against Ollama, vLLM, or any OpenAI-compatible local model server for full offline operation and data privacy.

---

## Prerequisites

You need a local model server running and accessible over HTTP. The examples below cover the most common options.

## Ollama

### 1. Start Ollama

```bash
ollama serve
ollama pull llama3.1:8b
```

### 2. Configure corteX

```python
import cortex

engine = cortex.Engine(
    providers={
        "local": {
            "base_url": "http://localhost:11434/v1",  # (1)!
        },
    },
    orchestrator_model="llama3.1:8b",
    worker_model="llama3.1:8b",
)
```

1. Ollama exposes an OpenAI-compatible API at `/v1` by default.

!!! tip
    No API key is required for local providers. If the server does not enforce authentication, you can omit the `api_key` field entirely.

## vLLM

### 1. Start the vLLM server

```bash
python -m vllm.entrypoints.openai.api_server \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --port 8000
```

### 2. Configure corteX

```python
engine = cortex.Engine(
    providers={
        "local": {
            "base_url": "http://localhost:8000/v1",
        },
    },
    orchestrator_model="meta-llama/Llama-3.1-8B-Instruct",
    worker_model="meta-llama/Llama-3.1-8B-Instruct",
)
```

## Any OpenAI-compatible server

corteX works with any server that implements the OpenAI Chat Completions API. This includes LM Studio, LocalAI, llama.cpp server, and TGI.

```python
engine = cortex.Engine(
    providers={
        "local": {
            "base_url": "http://your-server:port/v1",
            "api_key": "optional-if-required",       # (1)!
        },
    },
    orchestrator_model="your-model-name",
    worker_model="your-model-name",
)
```

1. Some servers require a dummy API key even if they do not validate it. Pass any non-empty string.

## Complete example

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={
            "local": {"base_url": "http://localhost:11434/v1"},
        },
        orchestrator_model="llama3.1:8b",
        worker_model="llama3.1:8b",
    )

    agent = engine.create_agent(
        name="private_assistant",
        system_prompt="You are a helpful assistant. All data stays on-premises.",
        enterprise_config=cortex.EnterpriseConfig(safety_level="strict"),
    )

    session = agent.start_session(user_id="local_user")
    response = await session.run("Summarize our Q4 revenue report.")

    print(response.content)
    print(f"Model: {response.metadata.model_used}")
    print(f"Latency: {response.metadata.latency_ms:.0f}ms")

    await session.close()


asyncio.run(main())
```

## Tool calling with local models

!!! warning
    Not all local models support tool calling. Models that do not support function calling will ignore tool definitions. Use a model with native tool support (e.g., Llama 3.1 Instruct, Mistral Instruct) for the best results.

```python
@cortex.tool(name="search_docs", description="Search internal documents")
async def search_docs(query: str) -> str:
    # Your search logic here
    return f"Results for: {query}"

agent = engine.create_agent(
    name="doc_search",
    system_prompt="You search internal documentation.",
    tools=[search_docs],
)
```

## Mix local and cloud providers

You can use a cloud model for orchestration and a local model for data-sensitive worker tasks:

```python
engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
        "local": {"base_url": "http://localhost:11434/v1"},
    },
    orchestrator_model="gpt-4o",           # Cloud for planning
    worker_model="llama3.1:8b",            # Local for execution
)
```

!!! note
    When mixing providers, the orchestrator model must come from one registered provider and the worker model from another. corteX routes automatically based on the model name.

---

## Next steps

- [Switch Between Providers](switching-providers.md) -- configure failover between local and cloud.
- [Connect to OpenAI](openai.md) -- add a cloud provider alongside your local setup.
- [Monitor Your Agent](../advanced/observability.md) -- track latency differences between local and cloud models.
