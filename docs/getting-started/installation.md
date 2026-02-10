# Installation

## Requirements

- **Python 3.11** or higher
- One LLM provider: OpenAI, Gemini, or a local model server

## Install

Choose the extras that match your provider.

=== "OpenAI"

    ```bash
    pip install cortex-engine[openai]
    ```

=== "Gemini"

    ```bash
    pip install cortex-engine[gemini]
    ```

=== "Local models"

    ```bash
    pip install cortex-engine
    ```

    No extras required. Point the `base_url` at your Ollama or vLLM instance.

=== "Everything"

    ```bash
    pip install cortex-engine[all]
    ```

    Includes OpenAI, Gemini, FastAPI server, and Playwright browser tools.

## Environment variables

Set your API key as an environment variable. The SDK reads it at provider registration time.

=== "OpenAI"

    ```bash
    export OPENAI_API_KEY="sk-..."
    ```

=== "Gemini"

    ```bash
    export GEMINI_API_KEY="AIza..."
    ```

=== "Local"

    No API key needed. Ensure your model server is running:

    ```bash
    # Ollama
    ollama serve

    # vLLM
    vllm serve meta-llama/Llama-3-8B --port 8000
    ```

!!! tip "Use `.env` files"
    The SDK depends on `python-dotenv`. Place a `.env` file in your project root and it will be loaded automatically:

    ```
    OPENAI_API_KEY=sk-...
    GEMINI_API_KEY=AIza...
    ```

## Verify

```python
import cortex

print(cortex.__version__)  # 3.0.0-alpha
```

## Provider configuration reference

Each provider accepts the following keys when passed to `Engine(providers={...})`:

| Key | Type | Description |
|---|---|---|
| `api_key` | `str` | API key for the provider. |
| `base_url` | `str` | Override the default endpoint (required for local models). |
| `default_model` | `str` | Model to use when no explicit model is specified. |
| `organization` | `str` | Organization ID (OpenAI only). |

Example with multiple providers:

```python
import cortex

engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
        "gemini": {"api_key": "AIza..."},
        "local": {"base_url": "http://localhost:11434/v1"},
    },
    orchestrator_model="gpt-4o",       # Complex reasoning
    worker_model="gemini-2.0-flash",   # Fast, simple tasks
)
```

---

Next: [Quick Start](quickstart.md)
