# Switching from CrewAI

A practical guide for teams migrating from CrewAI to corteX.

---

## Why Switch

CrewAI orchestrates crews of role-playing agents with sequential or hierarchical processes.
corteX provides a brain-inspired engine where every step is verified against the original goal,
weights adapt from experience, and loops are detected automatically.

| Dimension | CrewAI | corteX |
|---|---|---|
| Orchestration | Crew process (sequential/hierarchical) | Brain pipeline - 14-step cognitive loop |
| Provider lock-in | OpenAI-focused | Provider agnostic - OpenAI, Gemini, Anthropic, local |
| Deployment | Cloud only | On-prem first, zero external dependencies |
| Goal safety | None - agents can drift | Goal tracker verifies every step |
| Loop prevention | None | State hashing + drift detection |
| Pricing | Free tier limited to 50 runs/month | Unlimited - you bring your own keys |

---

## Concept Mapping

| CrewAI | corteX | Notes |
|---|---|---|
| `Crew` | `Engine` | Root object that manages providers and config |
| `Agent(role=..., goal=...)` | `engine.create_agent(name=..., system_prompt=...)` | corteX agents are stateless templates |
| `Task(description=...)` | `session.run("...")` | Goals are tracked automatically |
| `@tool` decorator | `@tool()` decorator | Nearly identical - see examples below |
| `crew.kickoff()` | `await session.run()` | corteX is async by default |
| `Process.sequential` | Default session loop | Brain pipeline handles ordering |
| `Process.hierarchical` | Multi-agent with `a2a_agents` | A2A protocol for agent-to-agent |
| `memory=True` | `MemoryFabric` + context engine | Built-in working, short-term, long-term, episodic |
| `LLM(model=...)` | `providers={"openai": {...}}` | Configure at engine level, not per-agent |

---

## Side-by-Side Examples

### Single Agent with a Tool

=== "CrewAI"

    ```python
    from crewai import Agent, Task, Crew
    from crewai.tools import tool

    @tool
    def search_docs(query: str) -> str:
        """Search internal documentation."""
        return f"Results for: {query}"

    agent = Agent(
        role="Support Rep",
        goal="Help users find answers",
        backstory="You are a helpful support agent.",
        tools=[search_docs],
        llm="gpt-4o",
    )

    task = Task(
        description="Help the user reset their password",
        agent=agent,
        expected_output="Step-by-step instructions",
    )

    crew = Crew(agents=[agent], tasks=[task])
    result = crew.kickoff()
    print(result)
    ```

=== "corteX"

    ```python
    import asyncio
    import cortex
    from cortex.tools.decorator import tool

    @tool(description="Search internal documentation.")
    def search_docs(query: str) -> str:
        return f"Results for: {query}"

    async def main():
        engine = cortex.Engine(
            providers={"openai": {"api_key": "sk-..."}},
        )

        agent = engine.create_agent(
            name="support",
            system_prompt="You are a helpful support agent.",
            tools=[search_docs],
        )

        session = agent.start_session(user_id="user_1")
        response = await session.run("Help me reset my password")
        print(response.content)
        await session.close()

    asyncio.run(main())
    ```

!!! tip "Key difference"
    CrewAI requires you to define a separate `Task` object with `expected_output`.
    corteX handles goal tracking internally - just pass your request to `session.run()`.

### Multi-Agent Workflow

=== "CrewAI"

    ```python
    from crewai import Agent, Task, Crew, Process

    researcher = Agent(
        role="Researcher",
        goal="Find relevant information",
        backstory="Expert researcher.",
        llm="gpt-4o",
    )
    writer = Agent(
        role="Writer",
        goal="Write clear summaries",
        backstory="Technical writer.",
        llm="gpt-4o",
    )

    research_task = Task(
        description="Research AI agent frameworks",
        agent=researcher,
        expected_output="Research notes",
    )
    write_task = Task(
        description="Write a summary from the research",
        agent=writer,
        expected_output="A 500-word summary",
    )

    crew = Crew(
        agents=[researcher, writer],
        tasks=[research_task, write_task],
        process=Process.sequential,
    )
    result = crew.kickoff()
    ```

=== "corteX"

    ```python
    import asyncio
    import cortex

    async def main():
        engine = cortex.Engine(
            providers={"openai": {"api_key": "sk-..."}},
        )

        researcher = engine.create_agent(
            name="researcher",
            system_prompt="You are an expert researcher. Find relevant information.",
        )
        writer = engine.create_agent(
            name="writer",
            system_prompt="You are a technical writer. Write clear summaries.",
        )

        # Step 1: Research
        research_session = researcher.start_session()
        research = await research_session.run("Research AI agent frameworks")

        # Step 2: Write using research output
        write_session = writer.start_session()
        summary = await write_session.run(
            f"Write a 500-word summary from this research:\n{research.content}"
        )
        print(summary.content)

        await research_session.close()
        await write_session.close()

    asyncio.run(main())
    ```

!!! info "Sequential vs parallel"
    corteX gives you explicit control. Chain sessions sequentially as shown above,
    or use `asyncio.gather()` for parallel execution. No framework magic to debug.

### Switching Providers

=== "CrewAI"

    ```python
    # CrewAI: set per-agent, OpenAI-centric
    agent = Agent(role="...", llm="gpt-4o")
    ```

=== "corteX"

    ```python
    # corteX: configure at engine level, all agents inherit
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": "sk-..."},
            "gemini": {"api_key": "AIza..."},
            "local": {"base_url": "http://localhost:11434/v1"},
        },
        orchestrator_model="gpt-4o",
        worker_model="gemini-2.5-flash",
    )
    ```

---

## What You Gain

- **Brain-inspired architecture** - synaptic weights, prediction/surprise, plasticity
- **Goal tracking** - every response verified against the original goal
- **Loop prevention** - state hashing + drift detection, automatic
- **Provider agnostic** - swap OpenAI, Gemini, Anthropic, or local in one line
- **On-prem deployment** - zero required external dependencies
- **Production readiness scoring** - `engine.readiness_score()` before deploy
- **No execution limits** - bring your own keys, no 50-runs/month ceiling
- **Enterprise features** - multi-tenant isolation, audit logging, compliance

## Migration Steps

1. **Install** - `pip install cortex-engine[openai]`
2. **Replace imports** - swap `from crewai import ...` with `import cortex`
3. **Convert tools** - rename `@tool` to `@tool(description="...")`, body stays the same
4. **Replace Crew/Agent/Task** - use the Engine/Agent/Session pattern shown above
5. **Add async** - wrap entry point in `async def main()` with `asyncio.run(main())`
6. **Readiness check** - run `engine.readiness_score()` to validate before deploying

## FAQ

**Can I use CrewAI tools directly?**
:   CrewAI tools are regular Python functions. Copy the function body into a corteX
    `@tool()` decorated function - no wrapper changes needed.

**Does corteX support hierarchical processes?**
:   Yes. Use A2A (Agent-to-Agent protocol) to connect agents that delegate to each other.

**What about CrewAI's memory feature?**
:   corteX provides four memory tiers out of the box (working, short-term, long-term,
    episodic) managed by MemoryFabric. No extra configuration needed.

**Is corteX async-only?**
:   Async by default for I/O. Wrap calls in `asyncio.run()` for synchronous scripts.
