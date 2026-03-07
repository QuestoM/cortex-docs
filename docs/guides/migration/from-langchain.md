# Switching from LangChain

A practical guide for LangChain developers migrating to corteX. Same goals, fewer abstractions, more built-in intelligence.

---

## Why switch?

LangChain orchestrates LLM calls through **chains** - explicit sequences of prompts, parsers, and retrievers that you wire together manually. corteX replaces chains with a **brain-inspired engine** that handles orchestration, goal tracking, and tool selection internally.

| | LangChain | corteX |
|---|---|---|
| Orchestration | You build chains/graphs | Engine handles it |
| Goal adherence | Not built-in | Automatic drift detection |
| Loop prevention | Not built-in | State hashing + detection |
| Tool trust | Not built-in | Synaptic weight reputation |
| Memory | Multiple classes to configure | Unified MemoryFabric |
| Observability | Requires LangSmith (external) | Built-in dashboard + audit |

---

## Concept mapping

| LangChain | corteX | Notes |
|---|---|---|
| `ChatOpenAI` / `ChatAnthropic` | `Engine(providers={...})` | Multi-provider with automatic routing |
| `AgentExecutor` | `session.run()` | Brain pipeline runs automatically |
| `@tool` decorator | `@tool()` from `corteX.tools.decorator` | Nearly identical syntax |
| `ConversationBufferMemory` | `MemoryFabric` | Working + short-term + long-term + episodic |
| `PromptTemplate` | `system_prompt` param | Plain string, no template wiring |
| `RunnableSequence` / LCEL | Not needed | Engine routes internally |
| `CallbackHandler` | `AuditLogger` / dashboard | Built-in observability |
| `VectorStoreRetriever` | `CorticalContextEngine` | Bring your own embeddings |
| LangSmith | Built-in dashboard | No external service required |
| LangGraph | `session.run()` | Goal tracker replaces explicit graphs |

---

## Side-by-side examples

### Simple chat agent

=== "LangChain"

    ```python
    from langchain_openai import ChatOpenAI
    from langchain.agents import AgentExecutor, create_openai_tools_agent
    from langchain_core.prompts import ChatPromptTemplate

    llm = ChatOpenAI(model="gpt-4o", api_key="sk-...")
    prompt = ChatPromptTemplate.from_messages([
        ("system", "You are a helpful assistant."),
        ("human", "{input}"),
        ("placeholder", "{agent_scratchpad}"),
    ])
    agent = create_openai_tools_agent(llm, [], prompt)
    executor = AgentExecutor(agent=agent, tools=[])

    result = executor.invoke({"input": "Explain quantum entanglement"})
    print(result["output"])
    ```

=== "corteX"

    ```python
    import asyncio
    from corteX.sdk import Engine

    async def main():
        engine = Engine(providers={"openai": {"api_key": "sk-..."}})
        agent = engine.create_agent(
            name="assistant",
            system_prompt="You are a helpful assistant.",
        )
        session = agent.start_session(user_id="user_1")
        response = await session.run("Explain quantum entanglement")
        print(response.content)
        await session.close()

    asyncio.run(main())
    ```

!!! tip
    corteX is async-first. Every `session.run()` call returns a `Response` with `.content` and `.metadata` - no need to dig into dict keys.

### Agent with custom tools

=== "LangChain"

    ```python
    from langchain_core.tools import tool

    @tool
    def get_weather(city: str) -> str:
        """Get the current weather for a city."""
        return f"22C and sunny in {city}"

    llm = ChatOpenAI(model="gpt-4o")
    prompt = ChatPromptTemplate.from_messages([...])
    agent = create_openai_tools_agent(llm, [get_weather], prompt)
    executor = AgentExecutor(agent=agent, tools=[get_weather])
    result = executor.invoke({"input": "Weather in Berlin?"})
    ```

=== "corteX"

    ```python
    from corteX.sdk import Engine
    from corteX.tools.decorator import tool

    @tool(name="get_weather", description="Get weather for a city")
    async def get_weather(city: str) -> str:
        return f"22C and sunny in {city}"

    engine = Engine(providers={"openai": {"api_key": "sk-..."}})
    agent = engine.create_agent(
        name="assistant",
        system_prompt="You help with weather questions.",
        tools=[get_weather],
    )
    session = agent.start_session(user_id="user_1")
    response = await session.run("Weather in Berlin?")
    ```

!!! note
    In LangChain, the tool must be passed to both `create_openai_tools_agent` and `AgentExecutor`. In corteX, you pass it once to `create_agent`.

### Multi-provider setup

=== "LangChain"

    ```python
    from langchain_openai import ChatOpenAI
    from langchain_google_genai import ChatGoogleGenerativeAI

    openai_llm = ChatOpenAI(model="gpt-4o")
    gemini_llm = ChatGoogleGenerativeAI(model="gemini-2.5-pro")
    # You must manually decide which LLM to use where
    ```

=== "corteX"

    ```python
    from corteX.sdk import Engine

    engine = Engine(
        providers={
            "openai": {"api_key": "sk-..."},
            "gemini": {"api_key": "AIza..."},
        },
        orchestrator_model="gpt-4o",
        worker_model="gemini-2.5-flash",
    )
    # Engine routes automatically: complex tasks to orchestrator, simple to worker
    ```

---

## What you gain

Features that corteX provides out of the box with no extra code:

- **Goal tracking** - every response is checked against the original goal. If the agent drifts, it self-corrects.
- **Loop prevention** - state hashing detects repeated patterns and breaks infinite loops before they waste tokens.
- **Synaptic weights** - tools and strategies build reputation over time. Effective tools get prioritized automatically.
- **Plasticity** - the engine adapts its behavior based on feedback, not just static configuration.
- **Production readiness scoring** - call `engine.readiness_score()` to get a 0-100 deployment readiness report.
- **Built-in compliance** - GDPR data classification, audit logging, and tenant isolation without external services.

---

## Migration steps

### 1. Install

```bash
pip install cortex-ai[openai]     # OpenAI provider
pip install cortex-ai[gemini]     # or Gemini
pip install cortex-ai[anthropic]  # or Anthropic
pip install cortex-ai[all]        # all providers
```

### 2. Replace imports

```python
# Before (LangChain)
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_openai_tools_agent
from langchain_core.tools import tool

# After (corteX)
from corteX.sdk import Engine
```

### 3. Convert chains to sessions

```python
# Before: build chain, prompt, agent, executor
llm = ChatOpenAI(model="gpt-4o")
# ... 10+ lines of chain wiring ...
result = executor.invoke({"input": query})

# After: engine -> agent -> session -> run
engine = Engine(providers={"openai": {"api_key": "sk-..."}})
agent = engine.create_agent(name="my_agent", system_prompt="...")
session = agent.start_session(user_id="user_1")
response = await session.run(query)
```

### 4. Convert tools

```python
# Before (LangChain)
@tool
def search_db(query: str) -> str:
    """Search the database."""
    return db.search(query)

# After (corteX) - add async, use @tool()
from corteX.tools.decorator import tool

@tool(name="search_db", description="Search the database")
async def search_db(query: str) -> str:
    return db.search(query)
```

### 5. Replace memory

```python
# Before (LangChain)
from langchain.memory import ConversationBufferMemory
memory = ConversationBufferMemory()
# ... pass memory to chain ...

# After (corteX) - memory is automatic
session = agent.start_session(user_id="user_1")
# MemoryFabric handles working, short-term, long-term, and episodic memory
# across the session lifecycle. No configuration needed.
```

### 6. Test

```python
response = await session.run("Your test prompt")
assert response.content  # response arrived
print(response.metadata.tokens_used)   # token usage
print(response.metadata.tools_called)  # which tools fired
print(response.metadata.latency_ms)    # round-trip time
await session.close()
```

---

## FAQ

**Can I use LangChain tools with corteX?**
Not directly. LangChain tools use a different interface. Convert them by wrapping the function body in a `@tool()` decorated async function (from `corteX.tools.decorator`) - usually a one-line change.

**Does corteX support LCEL (LangChain Expression Language)?**
corteX does not use chains or LCEL. The brain engine handles orchestration internally. If you need explicit step sequencing, use multiple `session.run()` calls.

**Can I use both during migration?**
Yes. corteX and LangChain can coexist in the same project. Migrate one agent at a time.

**What about LangGraph?**
LangGraph adds state machines on top of LangChain. corteX replaces this with goal tracking and adaptive weights - the engine decides the next step based on progress toward the goal, not a predefined graph.

**Is corteX async-only?**
The SDK is async-first. Use `asyncio.run()` or an async framework like FastAPI. This matches modern Python best practices for I/O-bound work.

---

Next: [Switching from CrewAI](from-crewai.md)
