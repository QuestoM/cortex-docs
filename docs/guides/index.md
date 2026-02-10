# How-To Guides

Task-oriented recipes for common corteX operations. Each guide assumes you have a working installation and solves a specific problem.

---

## LLM Providers

Connect corteX to the LLM backend that fits your needs.

| Guide | What you will learn |
|---|---|
| [Connect to OpenAI](providers/openai.md) | Configure OpenAI and Azure OpenAI as your LLM provider. |
| [Connect to Google Gemini](providers/gemini.md) | Set up Gemini models including gemini-2.5-pro and gemini-2.5-flash. |
| [Use Local Models](providers/local-models.md) | Run agents against Ollama, vLLM, or any OpenAI-compatible server. |
| [Switch Between Providers](providers/switching-providers.md) | Set up multi-provider failover and split orchestrator/worker routing. |

## Configuration

Fine-tune how your agent thinks and responds.

| Guide | What you will learn |
|---|---|
| [Tune Agent Weights](config/weight-tuning.md) | Adjust verbosity, formality, autonomy, and other synaptic weights. |
| [Configure Context Profiles](config/context-profiles.md) | Choose memory profiles, set token budgets, and manage context tiers. |
| [Control Temperature & Creativity](config/temperature.md) | Understand dual-process temperature routing and apply manual overrides. |

## Tools

Extend your agent with custom capabilities.

| Guide | What you will learn |
|---|---|
| [Create Custom Tools](tools/custom-tools.md) | Build tools with the `@cortex.tool()` decorator, handle async, and validate parameters. |
| [Manage Tool Reputation](tools/tool-reputation.md) | Monitor tool trust scores, understand quarantine, and inspect reputation stats. |

## Advanced

Power-user techniques for simulation, modulation, persistence, and observability.

| Guide | What you will learn |
|---|---|
| [Run What-If Simulations](advanced/what-if-simulation.md) | Use the digital twin to test weight changes before committing them. |
| [Override with Targeted Modulation](advanced/modulation-overrides.md) | Force-activate or silence tools with time-based expiry. |
| [Persist Weights Across Sessions](advanced/weight-persistence.md) | Save and restore learned weights so agents improve over time. |
| [Monitor Your Agent](advanced/observability.md) | Access stats, export metrics, and configure logging for production. |

---

## Prerequisites

All guides assume you have already completed the [Quick Start](../getting-started/quickstart.md) and have at least one LLM provider configured.

## Conventions

- Code examples use `async/await`. Wrap top-level calls in `asyncio.run()` if you are not already inside an async context.
- Examples that show multiple providers use tabbed blocks -- click the tab for your provider.
- Admonition boxes (`tip`, `warning`, `note`) highlight important context.
