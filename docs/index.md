# corteX

**Enterprise AI Agent SDK -- brain-inspired, goal-driven, on-prem ready.**

```python
import cortex

engine = cortex.Engine(providers={"openai": {"api_key": "sk-..."}})
agent = engine.create_agent(name="support", system_prompt="You help users.")
session = agent.start_session(user_id="user_123")
response = await session.run("Help me with my order")
```

---

<div class="grid cards" markdown>

-   :material-rocket-launch: **Getting Started**

    ---

    Install, configure, and run your first agent in under five minutes.

    [:octicons-arrow-right-24: Quick start](getting-started/quickstart.md)

-   :material-brain: **Concepts**

    ---

    Understand the architecture: 20 brain components, dual-process routing, and the 14-step pipeline.

    [:octicons-arrow-right-24: Architecture](concepts/architecture.md)

-   :material-shield-lock: **Enterprise**

    ---

    Multi-tenant isolation, audit logging, compliance controls, and on-premises deployment.

    [:octicons-arrow-right-24: Enterprise](enterprise/index.md)

</div>

---

## Why corteX

| Capability | Detail |
|---|---|
| **20 brain components** | Weights, goals, prediction, plasticity, attention, calibration, concept graphs, and more -- organized in four priority tiers (P0--P3). |
| **Provider-agnostic** | OpenAI, Gemini, and local models (Ollama, vLLM) behind a single router. Switch providers without changing application code. |
| **On-prem ready** | Run entirely within your infrastructure. No data leaves your network. |
| **Enterprise security** | Safety levels, blocked topics, audit logging, compliance rules, and per-session token budgets. |
| **Adaptive learning** | Bayesian weight updates, Hebbian association, prospect-theory loss aversion, and dual-process System 1/2 routing -- all within a single session. |
| **10,000+ step context** | Cortical Context Engine with hot/warm/cold memory tiers, automatic summarization, and checkpointing. |

---

## Minimal requirements

- Python 3.11+
- One LLM provider (OpenAI, Gemini, or a local model)

```bash
pip install cortex-engine[openai]
```

[:octicons-arrow-right-24: Full installation guide](getting-started/installation.md)
