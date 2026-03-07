# corteX vs Other Frameworks

How corteX compares to popular AI agent frameworks. We focus on architectural differences, not marketing claims.

## Feature Matrix

| Feature | corteX | LangChain | CrewAI | AutoGen |
|---------|--------|-----------|--------|---------|
| Goal tracking per step | Built-in | No | No | No |
| Loop detection | Multi-resolution (state hash + semantic + behavioral) | No | No | No |
| Adaptive tool selection | Bayesian (Thompson Sampling) | No | No | No |
| Implicit feedback learning | 4-tier (corrections, frustration, satisfaction) | No | No | No |
| Context window management | Cortical engine (10K+ steps) | Basic buffer | No | No |
| Drift detection | Real-time with auto-replan | No | No | No |
| Production readiness score | 0-100 automated scoring | No | No | No |
| On-prem capable | 100% (zero external deps) | Partial | Partial | Partial |
| Provider support | OpenAI, Gemini, Claude, Local | OpenAI, Gemini, Claude, many | OpenAI, Gemini, Claude | OpenAI, Gemini, Claude |
| MCP protocol | Yes | Yes | No | No |
| A2A protocol | Yes | No | No | No |
| SOC 2 / GDPR compliance | Built-in audit, PII redaction | No | No | No |
| Multi-tenant isolation | Per-engine registries | No | No | No |
| Token budget control | Adaptive budget with early exit | No | No | No |
| Dependency count | ~5 core | 50+ | 20+ | 30+ |

## Code Comparison: Same Task

**Task**: Create an agent with a tool, run it, and inspect what happened.

### corteX (8 lines)

```python
import asyncio
from corteX import quick_agent, tool

@tool(name="lookup", description="Look up product info")
def lookup(query: str) -> str:
    return f"Premium Plan: $99/mo"

agent, session = quick_agent("support", "You help customers.", tools=[lookup])
response = asyncio.run(session.run("What is the Premium plan price?"))

print(response.content)
print(f"Goal progress: {response.metadata.goal_progress:.0%}")
print(f"Drift score:   {response.metadata.drift_score:.2f}")
print(f"Tools called:  {response.metadata.tools_called}")
```

**What you get**: goal tracking, drift detection, loop prevention, adaptive tool selection - all automatic.

### LangChain (25+ lines)

```python
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.tools import tool
from langchain_core.prompts import ChatPromptTemplate

@tool
def lookup(query: str) -> str:
    """Look up product info"""
    return f"Premium Plan: $99/mo"

llm = ChatOpenAI(model="gpt-4o")
prompt = ChatPromptTemplate.from_messages([
    ("system", "You help customers."),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])
agent = create_tool_calling_agent(llm, [lookup], prompt)
executor = AgentExecutor(agent=agent, tools=[lookup])

result = executor.invoke({"input": "What is the Premium plan price?"})
print(result["output"])
# No goal tracking, no drift score, no loop detection
```

**What you don't get**: no goal tracking, no drift detection, no loop prevention. You'd need LangSmith ($400/mo) for basic observability.

### CrewAI (20+ lines)

```python
from crewai import Agent, Task, Crew
from crewai.tools import tool

@tool("Lookup product info")
def lookup(query: str) -> str:
    return f"Premium Plan: $99/mo"

agent = Agent(
    role="Support Agent",
    goal="Help customers with product questions",
    backstory="You are a helpful support agent.",
    tools=[lookup],
    llm="gpt-4o",
)

task = Task(
    description="What is the Premium plan price?",
    expected_output="The price of the Premium plan",
    agent=agent,
)

crew = Crew(agents=[agent], tasks=[task])
result = crew.kickoff()
print(result)
# No per-step goal verification, no drift detection, no loop prevention
```

**What you don't get**: no real-time goal verification, no drift detection, no adaptive learning.

## Architectural Differences

### Why corteX is different

Most frameworks focus on **connecting agents to tools** (the "plumbing" problem). corteX focuses on **making agents themselves smarter** (the "brain" problem).

| Layer | Other Frameworks | corteX |
|-------|------------------|--------|
| LLM calls | Pass-through | Brain-augmented (goal DNA injected, context pinned) |
| Tool selection | Random or ordered | Bayesian with Thompson Sampling |
| Failure handling | Retry or crash | 5-level graceful degradation chain |
| Learning | None (stateless) | Cross-session weight persistence |
| Quality | Hope for the best | Automated quality gate with regen |
| Cost | Unbounded | Adaptive budget with early exit |

### Dependencies

corteX core has 5 dependencies. LangChain pulls 50+.

```bash
# corteX (minimal install)
pip install cortex-ai  # pydantic, httpx, numpy, aiohttp, cryptography

# LangChain
pip install langchain langchain-openai  # 50+ transitive dependencies
```

### On-Prem Deployment

corteX runs 100% on-prem with any OpenAI-compatible local endpoint:

```python
engine = Engine(providers={
    "local": {"base_url": "http://localhost:8000/v1"},
})
```

No telemetry, no external calls, no cloud dependencies. Enterprise compliance (SOC 2, GDPR) is built in, not bolted on.

## When to Use What

| Scenario | Recommendation |
|----------|---------------|
| Quick prototype, lots of integrations | LangChain has the largest connector ecosystem |
| Multi-agent role-play workflows | CrewAI has opinionated role/task abstractions |
| Research and experimentation | AutoGen has flexible conversation patterns |
| **Production agents that must work reliably** | **corteX** - goal tracking, loop prevention, adaptive learning |
| **Enterprise with compliance requirements** | **corteX** - SOC 2, GDPR, on-prem, audit trails |
| **Cost-sensitive deployments** | **corteX** - adaptive budget, early exit, token optimization |

## Migration

Moving from another framework to corteX typically takes 1-2 hours for a basic agent. The main change is replacing framework-specific abstractions with corteX's Engine/Agent/Session pattern. Your existing tools, prompts, and business logic stay the same.

See the [Quickstart Guide](https://docs.cortex-ai.com/quickstart/) for setup instructions.
