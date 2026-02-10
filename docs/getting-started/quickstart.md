# Quick Start

A working agent in four steps. Total time: under five minutes.

---

## 1. Install

```bash
pip install cortex-engine[openai]
```

## 2. Configure

Set your API key:

```bash
export OPENAI_API_KEY="sk-..."
```

## 3. Run

Create a file called `main.py`:

```python
import asyncio
import cortex


async def main():
    # 1. Create the engine with your provider
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},  # (1)!
    )

    # 2. Create an agent with a personality
    agent = engine.create_agent(
        name="assistant",
        system_prompt="You are a concise, helpful assistant.",
    )

    # 3. Start a conversation session
    session = agent.start_session(user_id="quickstart_user")

    # 4. Run a message
    response = await session.run("What are the three laws of thermodynamics?")
    print(response.content)

    # Clean up
    await session.close()


asyncio.run(main())
```

1.  Replace with your actual API key, or omit to use the `OPENAI_API_KEY` environment variable.

## 4. See the output

```bash
python main.py
```

```text
The three laws of thermodynamics:

1. **First Law (Conservation of Energy):** Energy cannot be created or
   destroyed, only transformed from one form to another.

2. **Second Law (Entropy):** In any spontaneous process, the total entropy
   of an isolated system always increases over time.

3. **Third Law (Absolute Zero):** As temperature approaches absolute zero,
   the entropy of a perfect crystal approaches zero.
```

---

## What just happened

| Step | Object | Purpose |
|---|---|---|
| `cortex.Engine(...)` | `Engine` | Registers LLM providers and manages routing. |
| `engine.create_agent(...)` | `Agent` | Stateless template with a name and system prompt. |
| `agent.start_session(...)` | `Session` | Stateful conversation. Initializes all 20 brain components. |
| `session.run(...)` | `Response` | Executes the 14-step brain pipeline and returns a response. |

## Response metadata

Every `Response` carries metadata about what happened inside the brain:

```python
print(response.metadata.model_used)    # e.g. "gpt-4o"
print(response.metadata.tokens_used)   # e.g. 384
print(response.metadata.latency_ms)    # e.g. 1247.3
print(response.metadata.tools_called)  # e.g. []
```

---

Next: [Your First Agent](first-agent.md)
