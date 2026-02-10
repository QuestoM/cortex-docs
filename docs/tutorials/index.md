# Tutorials

Hands-on projects that walk you through building complete agents from start to finish. Each tutorial teaches a different set of corteX capabilities by having you build something real.

---

## Before you start

All tutorials assume you have completed the [Quick Start](../getting-started/quickstart.md) guide and have a working corteX installation with at least one LLM provider configured.

```bash
pip install cortex-engine[openai]
```

---

## Learning path

Work through the tutorials in order, or jump to the one that matches your use case.

| Tutorial | Difficulty | Time | What you build |
|----------|:----------:|:----:|----------------|
| [Customer Support Agent](support-agent.md) | Beginner | 30 min | A support agent with tools, weight tuning, goal tracking, and enterprise safety controls. |
| [Code Review Agent](code-review-agent.md) | Intermediate | 20 min | A code review agent that uses the coding context profile and demonstrates System 1/2 routing on simple vs. complex reviews. |
| [Research Agent](research-agent.md) | Intermediate | 20 min | A research agent with web search, document reading, memory fabric for long sessions, and multi-step goal tracking. |
| [Multi-Provider Failover](multi-provider.md) | Intermediate | 15 min | A resilient setup with OpenAI as the primary orchestrator, Gemini as worker and fallback, and automatic failover. |
| [Deploy to Production](production.md) | Advanced | 25 min | Enterprise configuration, audit logging, environment management, monitoring, on-prem deployment, and performance tuning. |

---

## What each tutorial covers

### Customer Support Agent

Build a full-featured support agent for a fictional e-commerce company. You will define tools for order lookup, FAQ search, and human escalation. Then you will tune the agent's behavioral weights for a formal, helpful tone, enable goal tracking for multi-step troubleshooting workflows, add enterprise safety controls, and observe brain metrics in real time.

**Key concepts:** tools, weight tuning, goal tracking, enterprise safety, brain metrics.

### Code Review Agent

Build an agent that reads source files, analyzes code quality, and produces structured review feedback. You will configure the coding context profile for optimal token management, set higher autonomy weights so the agent can work independently, and observe how the dual-process router handles trivial formatting issues (System 1) versus complex architectural concerns (System 2).

**Key concepts:** context profiles, autonomy weights, dual-process routing.

### Research Agent

Build an agent that conducts multi-step research across web sources and documents. You will configure the research context profile, use the memory fabric to maintain coherence across long sessions, and track progress through a structured research plan with goal tracking.

**Key concepts:** memory fabric, context profiles, goal tracking, long sessions.

### Multi-Provider Failover

Set up a production-grade multi-provider configuration with OpenAI as the primary orchestrator and Gemini as both a worker model and a fallback. You will see how the dual-process router maps System 1 to the fast worker model and System 2 to the orchestrator, and how automatic failover kicks in when a provider is unavailable.

**Key concepts:** multi-provider setup, model routing, failover, System 1/2 mapping.

### Deploy to Production

Take any of the agents you have built and prepare it for production deployment. You will configure enterprise safety controls, set up audit logging, manage secrets through environment variables, add structured logging, monitor session statistics, and tune performance for high-throughput workloads.

**Key concepts:** enterprise config, audit logging, monitoring, on-prem deployment, performance tuning.
