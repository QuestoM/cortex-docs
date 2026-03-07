# Integrations

## Two Ways to Use corteX

corteX is designed with **dual positioning** - use it as a standalone SDK to build agents from scratch, or as an **intelligence layer** on top of your existing framework.

### Standalone - Build Agents from Scratch

```python
import asyncio
from corteX import quick_agent, tool

@tool(name="lookup", description="Look up product info")
def lookup(query: str) -> str:
    return f"Result for '{query}': Premium Plan - $99/mo"

agent, session = quick_agent("support", "You help customers.", tools=[lookup])
response = asyncio.run(session.run("What is the Premium plan price?"))
print(response.content)
```

### Intelligence Layer - Add Brain to Your Existing Framework

Already using LangChain, CrewAI, or raw OpenAI calls? **Keep them.** Add corteX as a brain layer on top. Your framework handles LLM calls and tool routing. corteX adds goal tracking, loop detection, adaptive weights, and 30+ brain components.

```
Your Framework (LangChain / CrewAI / OpenAI)
     |
     v
corteX IntegrationBrain (30+ components)
     |
     v
Goal tracking, loop detection, drift alerts, weight learning, predictions...
```

**Why this matters:** Switch from LangChain to CrewAI tomorrow? Your corteX brain transfers. The intelligence layer stays - only the framework changes.

## Supported Frameworks

| Framework | Wrapper Class | Install |
|-----------|--------------|---------|
| [LangChain](langchain.md) | `CortexBrain` | `pip install cortex-ai[langchain]` |
| [CrewAI](crewai.md) | `CortexCrewBrain` | `pip install cortex-ai[crewai]` |
| [OpenAI](openai.md) | `CortexOpenAI` | `pip install cortex-ai[openai]` |
| Any framework | [`IntegrationBrain`](brain-api.md) | `pip install cortex-ai` |

## Brain Modes

All integrations support two brain modes:

| Mode | Components | Cost | Best For |
|------|-----------|------|----------|
| **full** (default) | 30+ brain components | Free | Production use - maximum intelligence |
| **lite** | 3 core components | Free | Minimal footprint, fastest startup |

Both modes are **completely free**. Full mode is the default because there is no reason not to use it unless you need the fastest possible startup time.

### What Full Mode Adds

Beyond the 3 core components (weights, goal tracker, feedback), full mode activates:

- **Prediction engine** - anticipates next steps, detects surprises
- **Plasticity** - adjusts learning rate based on performance
- **Bayesian tool selector** - Thompson Sampling for optimal tool choice
- **Tool reputation** - tracks reliability scores per tool
- **Attention filtering** - focuses on relevant context
- **Calibration** - metacognitive confidence tracking
- **Bias detector** - flags when LLM defaults override instructions
- **Context pinner** - reinforces critical instructions against drift
- **Degradation chain** - graceful fallback on errors
- **Memory fabric** - cross-session learning
- And 20+ more components

## Quick Comparison

```python
# Before: Raw LangChain - no intelligence
result = await agent.ainvoke({"input": "Help me with billing"})
# Hope it works... no visibility into what happened

# After: LangChain + corteX brain
from corteX.integrations.langchain import CortexBrain

brain = CortexBrain(agent_executor=agent, goal="Resolve billing issues")
result = await brain.run("Help me with billing")

print(result["goal_progress"])   # 0.85 - goal nearly complete
print(result["drift_score"])     # 0.05 - agent stayed on track
print(result["loop_detected"])   # False - no repetitive loops
print(result["brain_state"])     # 30+ component metrics
```

## Next Steps

- [LangChain Integration Guide](langchain.md) - detailed setup with AgentExecutor
- [CrewAI Integration Guide](crewai.md) - multi-agent crews with brain monitoring
- [OpenAI Wrapper Guide](openai.md) - raw chat completions with intelligence
- [IntegrationBrain API](brain-api.md) - build your own integration for any framework
