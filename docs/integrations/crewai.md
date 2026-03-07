# CrewAI Integration

Add corteX intelligence to any CrewAI Crew. Your multi-agent crew keeps handling delegation, LLM calls, and tool routing - corteX adds a brain that monitors the entire crew's progress, detects loops, and learns from outcomes.

## Installation

```bash
pip install cortex-ai[crewai]
```

## Quick Start

```python
from corteX.integrations.crewai import CortexCrewBrain

# Wrap your existing CrewAI crew
brain = CortexCrewBrain(
    crew=my_crew,
    goal="Write a comprehensive market analysis report",
)

# Run with brain monitoring
result = await brain.run()

print(result["output"])
print(f"Goal progress: {result['goal_progress']:.0%}")
print(f"Drift: {result['drift_score']:.2f}")
print(f"Agent loops: {result['agent_loops']}")
```

## How It Works

`CortexCrewBrain` adds intelligence without changing your crew:

1. **Event listener injection** - A `CortexCrewCallbackHandler` is attached to the crew's event listeners. It captures agent transitions, tool calls, task completions, and errors.

2. **Task-aware goal tracking** - The brain extracts task descriptions from your crew and registers them as goal plan steps. Progress is tracked per-task.

3. **Multi-agent loop detection** - Beyond single-agent loops, the brain detects circular delegation patterns where agents keep passing work back and forth.

4. **Agent transition tracking** - Every handoff between CrewAI agents is recorded, letting you see the delegation chain and identify bottlenecks.

## Configuration

```python
brain = CortexCrewBrain(
    crew=my_crew,
    goal="Write a comprehensive market analysis report",

    # Brain mode: "full" (default, 30+ components) or "lite" (3 core)
    brain_mode="full",

    # Max steps before forced stop
    max_steps=50,
)
```

## CrewAI-Specific Metrics

The CrewAI wrapper tracks metrics unique to multi-agent crews:

```python
result = await brain.run()

# Standard brain metrics
print(result["goal_progress"])    # Overall crew progress
print(result["drift_score"])      # How far the crew wandered
print(result["loop_detected"])    # Any loops detected

# CrewAI-specific metrics
print(result["agent_loops"])      # Circular delegation detected
print(result["agent_transitions"])  # Number of agent handoffs
print(result["steps"])            # Total steps across all agents
print(result["tool_calls"])       # Total tool invocations
```

### Brain State with Agent Context

```python
state = brain.brain_state

# Includes all standard brain metrics plus:
print(state["agent_loops"])       # CrewAI agent loop flag
```

## Accessing Brain Components

```python
# Direct brain access
brain.brain                    # IntegrationBrain instance
brain.weight_engine            # WeightEngine
brain.goal_tracker             # GoalTracker
brain.callback_handler         # CortexCrewCallbackHandler

# Full component access (in full mode)
brain.brain.get_component("prediction")
brain.brain.get_component("bayesian_selector")
brain.brain.get_component("reputation")
```

## Synchronous Usage

```python
result = brain.run_sync(inputs={"topic": "cloud security market"})
```

## Full Example

```python
from crewai import Agent, Task, Crew
from corteX.integrations.crewai import CortexCrewBrain

# 1. Define your CrewAI agents and tasks (as you normally would)
researcher = Agent(
    role="Market Researcher",
    goal="Find market data and trends",
    backstory="Expert in market analysis",
    llm="gpt-4o",
)

writer = Agent(
    role="Report Writer",
    goal="Write clear, data-driven reports",
    backstory="Expert business writer",
    llm="gpt-4o",
)

research_task = Task(
    description="Research the cloud security market size and trends for 2026",
    expected_output="Bullet-point summary with data sources",
    agent=researcher,
)

writing_task = Task(
    description="Write a 2-page market analysis based on the research",
    expected_output="Professional market analysis report",
    agent=writer,
)

crew = Crew(agents=[researcher, writer], tasks=[research_task, writing_task])

# 2. Wrap with corteX brain
brain = CortexCrewBrain(
    crew=crew,
    goal="Produce an accurate market analysis of cloud security",
)

# 3. Run with intelligence
result = await brain.run()

print(result["output"])
print(f"Progress: {result['goal_progress']:.0%}")
print(f"Agent transitions: {result['agent_transitions']}")
print(f"Circular delegation: {result['agent_loops']}")

# 4. Check which tools the crew learned to prefer
weights = brain.weight_engine.snapshot()
print(weights)
```

## Key Differences from LangChain Integration

| Feature | LangChain (`CortexBrain`) | CrewAI (`CortexCrewBrain`) |
|---------|--------------------------|---------------------------|
| Input | Single agent executor | Multi-agent crew |
| Goal source | User input text | Crew task descriptions |
| Loop detection | Single-agent loops | Single + multi-agent loops |
| Agent tracking | N/A | Agent transitions, delegation chains |
| Run method | `brain.run("input text")` | `brain.run(inputs={...})` |
| Kickoff | `executor.ainvoke()` | `crew.kickoff_async()` |
