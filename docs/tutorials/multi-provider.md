# Set Up Multi-Provider Failover

In this tutorial you will configure a resilient multi-provider setup with OpenAI as the primary orchestrator model and Gemini as both a fast worker model and a fallback. You will see how the dual-process router maps System 1 (fast) requests to the worker model and System 2 (slow) requests to the orchestrator, and how automatic failover kicks in when a provider is unavailable.

---

## What you will build

- A multi-provider engine with OpenAI and Gemini
- Dual-process routing: System 1 (Gemini worker) and System 2 (OpenAI orchestrator)
- Automatic failover from OpenAI to Gemini when the primary provider fails
- A monitoring loop that shows which model handles each request

## Prerequisites

- Python 3.11+
- corteX installed: `pip install cortex-engine[openai,gemini]`
- An OpenAI API key and a Google Gemini API key

---

## Step 1: Set up the project

```bash
mkdir multi-provider && cd multi-provider
touch failover_demo.py
```

```python title="failover_demo.py"
import asyncio
import os
import cortex
```

---

## Step 2: Configure the multi-provider engine

Register both providers and assign roles. The `orchestrator_model` handles complex reasoning (System 2), while the `worker_model` handles routine tasks (System 1).

```python title="failover_demo.py"
async def main():
    engine = cortex.Engine(
        providers={
            "openai": {
                "api_key": os.environ.get("OPENAI_API_KEY", "sk-..."),
            },
            "gemini": {
                "api_key": os.environ.get("GEMINI_API_KEY", "AIza..."),
            },
        },
        orchestrator_model="gpt-4o",              # (1)!
        worker_model="gemini-3-flash-preview",           # (2)!
        fallback_models=["gemini-3-flash-preview"],      # (3)!
    )
```

1. The orchestrator model handles System 2 (slow, deliberate) routing. This is your quality-optimized model for complex reasoning, planning, and high-stakes decisions.
2. The worker model handles System 1 (fast, automatic) routing. This is your speed-optimized model for routine tasks, simple tool calls, and conversational turns.
3. The fallback list specifies which models to try if the primary orchestrator is unavailable. corteX will try each model in order until one succeeds.

!!! info "Why split models?"
    Splitting orchestrator and worker models is a core corteX design pattern. Most agent turns are routine -- greeting the user, calling a simple tool, formatting output. These do not need the most powerful model. By routing 70-80% of decisions through a fast, cheap worker model and reserving the orchestrator for the 20-30% that require deep reasoning, you reduce cost and latency without sacrificing quality on the tasks that matter.

---

## Step 3: Create an agent and session

```python title="failover_demo.py"
    agent = engine.create_agent(
        name="resilient_agent",
        system_prompt=(
            "You are a helpful assistant. Answer questions clearly and concisely."
        ),
        goal_tracking=True,
    )

    session = agent.start_session(user_id="demo_user")
```

---

## Step 4: Demonstrate System 1 routing (worker model)

Send a simple message that the dual-process router will classify as routine, routing it to the fast worker model.

```python title="failover_demo.py"
    # Simple question -- expect System 1 (Gemini worker)
    print("--- System 1: Simple Question ---")
    r1 = await session.run("What is 2 + 2?")
    print(f"Response: {r1.content}")
    print(f"Model used: {r1.metadata.model_used}")
    print(f"Latency: {r1.metadata.latency_ms:.0f}ms")
```

Expected output:

```text
--- System 1: Simple Question ---
Response: 2 + 2 = 4.
Model used: gemini-3-flash-preview
Latency: 312ms
```

---

## Step 5: Demonstrate System 2 routing (orchestrator model)

Send a complex reasoning question. The router should detect high novelty and escalate to System 2.

```python title="failover_demo.py"
    # Complex reasoning -- expect System 2 (OpenAI orchestrator)
    print("\n--- System 2: Complex Reasoning ---")
    r2 = await session.run(
        "A farmer has 17 sheep. All but 9 die. How many sheep does "
        "the farmer have left? Explain your reasoning step by step."
    )
    print(f"Response: {r2.content}")
    print(f"Model used: {r2.metadata.model_used}")
    print(f"Latency: {r2.metadata.latency_ms:.0f}ms")
```

Expected output:

```text
--- System 2: Complex Reasoning ---
Response: The farmer has 9 sheep left.

Step-by-step reasoning:
1. The farmer starts with 17 sheep.
2. "All but 9 die" means every sheep except 9 dies.
3. Therefore, 9 sheep survive.

The key is in the phrasing: "all but 9" means 9 are excluded from dying.

Model used: gpt-4o
Latency: 1847ms
```

---

## Step 6: Observe routing statistics

Check the dual-process stats to confirm the routing pattern:

```python title="failover_demo.py"
    # Check routing statistics
    print("\n--- Routing Statistics ---")
    dp_stats = session.get_dual_process_stats()
    print(f"System 1 (worker) decisions:      {dp_stats['system1_count']}")
    print(f"System 2 (orchestrator) decisions: {dp_stats['system2_count']}")
    print(f"System 2 ratio:                   {dp_stats['system2_ratio']:.1%}")
```

Expected output:

```text
--- Routing Statistics ---
System 1 (worker) decisions:      1
System 2 (orchestrator) decisions: 1
System 2 ratio:                   50.0%
```

!!! tip "Healthy System 2 ratio"
    In production, a healthy System 2 ratio is typically 10-20%. If it exceeds 30%, your agent may be encountering too many novel situations -- consider improving the system prompt or adding tools to handle recurring patterns. If it is below 5%, the escalation thresholds may be too high and the agent could be missing edge cases.

---

## Step 7: Demonstrate automatic failover

To test failover, send a request that simulates the primary provider being unavailable. In a real scenario, corteX automatically retries with the fallback model when the primary returns an error or times out.

```python title="failover_demo.py"
    # Simulate a scenario where the orchestrator might be slow or unavailable.
    # corteX automatically falls back to the next model in the fallback list.
    print("\n--- Failover Demonstration ---")
    print("If OpenAI were unavailable, corteX would automatically route to Gemini.")
    print("The fallback is transparent -- same API, same response format.")
    print(f"Configured fallback models: {['gemini-3-flash-preview']}")

    # Send another complex question to show the system handles it gracefully
    r3 = await session.run(
        "Explain the difference between concurrency and parallelism "
        "in the context of Python's asyncio."
    )
    print(f"\nResponse: {r3.content}")
    print(f"Model used: {r3.metadata.model_used}")
```

!!! warning "Testing failover in development"
    To test failover without actually breaking a provider, you can temporarily set an invalid API key for the primary provider. corteX will detect the authentication failure and fall back to the next available model.

    ```python
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": "sk-invalid-key-for-testing"},
            "gemini": {"api_key": os.environ["GEMINI_API_KEY"]},
        },
        orchestrator_model="gpt-4o",
        worker_model="gemini-3-flash-preview",
        fallback_models=["gemini-3-flash-preview"],
    )
    ```

---

## Step 8: Close the session

```python title="failover_demo.py"
    stats = await session.close()
    print(f"\nSession closed. Turns: {stats['turns']}, Tokens: {stats['total_tokens']}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Step 9: Run the demo

```bash
export OPENAI_API_KEY="sk-..."
export GEMINI_API_KEY="AIza..."
python failover_demo.py
```

---

## Complete code

```python title="failover_demo.py"
import asyncio
import os
import cortex


async def main():
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": os.environ.get("OPENAI_API_KEY", "sk-...")},
            "gemini": {"api_key": os.environ.get("GEMINI_API_KEY", "AIza...")},
        },
        orchestrator_model="gpt-4o",
        worker_model="gemini-3-flash-preview",
        fallback_models=["gemini-3-flash-preview"],
    )

    agent = engine.create_agent(
        name="resilient_agent",
        system_prompt="You are a helpful assistant. Answer clearly and concisely.",
        goal_tracking=True,
    )
    session = agent.start_session(user_id="demo_user")

    print("--- System 1: Simple Question ---")
    r1 = await session.run("What is 2 + 2?")
    print(f"Response: {r1.content}")
    print(f"Model: {r1.metadata.model_used} | Latency: {r1.metadata.latency_ms:.0f}ms")

    print("\n--- System 2: Complex Reasoning ---")
    r2 = await session.run(
        "A farmer has 17 sheep. All but 9 die. How many are left? "
        "Explain step by step."
    )
    print(f"Response: {r2.content}")
    print(f"Model: {r2.metadata.model_used} | Latency: {r2.metadata.latency_ms:.0f}ms")

    print("\n--- Routing Statistics ---")
    dp = session.get_dual_process_stats()
    print(f"System 1: {dp['system1_count']} | System 2: {dp['system2_count']} "
          f"| Ratio: {dp['system2_ratio']:.1%}")

    print("\n--- Complex Follow-up ---")
    r3 = await session.run("Explain concurrency vs parallelism in Python asyncio.")
    print(f"Response: {r3.content}")
    print(f"Model: {r3.metadata.model_used} | Latency: {r3.metadata.latency_ms:.0f}ms")

    stats = await session.close()
    print(f"\nSession closed. Turns: {stats['turns']}, Tokens: {stats['total_tokens']}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## What you learned

- How to configure multiple LLM providers in a single `cortex.Engine`
- How `orchestrator_model` and `worker_model` map to System 2 and System 1 routing
- How `fallback_models` provides automatic failover when the primary provider is unavailable
- How to monitor routing decisions with `get_dual_process_stats()`

## Next steps

- [Deploy to Production](production.md) -- enterprise safety, audit logging, and performance tuning
- [Multi-Model Routing concepts](../concepts/llm-routing.md) -- deep dive into Nash Equilibrium routing and minimax safety
- [LLM Provider guides](../guides/providers/openai.md) -- provider-specific configuration
