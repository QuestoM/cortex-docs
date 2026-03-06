# Streaming Responses

Use `run_stream()` to receive tokens as they are generated, rather than waiting for the complete response.

---

## `run_stream()` vs `run()`

| Method | Returns | Use case |
|---|---|---|
| `session.run(message)` | `Response` | Backend processing, batch jobs, full metadata. |
| `session.run_stream(message)` | `AsyncIterator[StreamChunk]` | Chat UIs, real-time displays, low time-to-first-token. |

---

## Basic streaming

```python
import asyncio
import cortex


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
    )

    agent = engine.create_agent(
        name="assistant",
        system_prompt="You are a helpful assistant.",
    )

    session = agent.start_session(user_id="user_1")

    async for chunk in session.run_stream("Explain quantum entanglement."):
        print(chunk.content, end="", flush=True)  # (1)!

    print()  # Newline after stream completes

    await session.close()


asyncio.run(main())
```

1.  Each `StreamChunk` contains a fragment of the response. Print without newline for a smooth typing effect.

---

## StreamChunk

Each chunk yielded by `run_stream()` is a `StreamChunk` with these fields:

| Field | Type | Description |
|---|---|---|
| `content` | `str` | The text fragment for this chunk. |
| `is_final` | `bool` | `True` for the last chunk in the stream. |
| `model` | `str` | The model that generated this chunk. |

---

## FastAPI integration

Stream directly to an HTTP response:

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
import cortex

app = FastAPI()
engine = cortex.Engine(providers={"openai": {"api_key": "sk-..."}})
agent = engine.create_agent(name="api", system_prompt="You are an API assistant.")


@app.post("/chat")
async def chat(message: str, user_id: str = "anonymous"):
    session = agent.start_session(user_id=user_id)

    async def generate():
        async for chunk in session.run_stream(message):
            yield chunk.content

    return StreamingResponse(generate(), media_type="text/plain")
```

---

## When to use streaming

!!! note "Trade-offs"
    `run_stream()` is a lightweight path. It processes feedback and records the turn, but skips some of the heavier brain components (dual-process routing, tool execution, quality estimation). For the full 14-step pipeline with tool calls, use `run()`.

| Scenario | Recommended |
|---|---|
| Chat UI with typing indicator | `run_stream()` |
| Agent with tool calls | `run()` |
| Batch processing | `run()` |
| Server-sent events | `run_stream()` |

---

Next: [Concepts](../concepts/index.md)
