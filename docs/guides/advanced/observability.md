# Monitor Your Agent

Access every stats method corteX exposes, configure logging, and export metrics for production monitoring.

---

## Response metadata

Every call to `session.run()` returns a `Response` with rich metadata:

```python
import cortex

engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    orchestrator_model="gpt-4o",
    worker_model="gpt-4o-mini",
)

agent = engine.create_agent(
    name="assistant",
    system_prompt="You are a helpful assistant.",
    goal_tracking=True,
    weight_config=cortex.WeightConfig(autonomy=0.7),
)

session = agent.start_session(user_id="user_123")
response = await session.run("Explain microservices architecture.")

# Core metadata
print(f"Model:        {response.metadata.model_used}")
print(f"Tokens:       {response.metadata.tokens_used}")
print(f"Latency:      {response.metadata.latency_ms:.0f}ms")
print(f"Tools called: {response.metadata.tools_called}")
print(f"Goal progress:{response.metadata.goal_progress}")
print(f"Drift score:  {response.metadata.drift_score}")
```

## Session stats reference

corteX exposes a comprehensive set of stats methods on every session. Here is the complete list, organized by subsystem.

### Core brain stats

```python
# Weight snapshot -- current effective weights after plasticity adjustments
weights = session.get_weights()

# Goal tracking -- progress toward declared goals
progress = session.get_goal_progress()

# Calibration -- how well confidence matches actual outcomes
calibration = session.get_calibration_report()

# Dual-process routing -- System 1 vs System 2 usage
dual_process = session.get_dual_process_stats()
```

### Tool stats

```python
# Tool reputation -- trust scores, call counts, quarantine status
reputation = session.get_reputation_stats()

# Active modulations -- current activate/silence overrides
modulations = session.get_active_modulations()
```

### Context engine stats

```python
# Context stats -- token usage by tier, eviction counts
context = session.get_context_stats()
```

### Phase 2 (P2) subsystem stats

```python
# Cortical columns -- column-level processing stats
columns = session.get_column_stats()

# Resource allocation -- how compute budget is distributed
resources = session.get_resource_map()

# Attention -- what the agent is focusing on
attention = session.get_attention_stats()
```

### Phase 3 (P3) subsystem stats

```python
# Concept graph -- learned concept relationships
concepts = session.get_concept_graph_stats()

# Territory map -- semantic territory organization
territories = session.get_territory_map()

# Active modulations -- P3-level modulation state
modulations = session.get_active_modulations()
```

## Build a monitoring dashboard

Collect stats after each interaction and log them for analysis:

```python
import json
import logging

logger = logging.getLogger("cortex.monitor")
logging.basicConfig(level=logging.INFO)


async def monitored_run(session, message: str):
    """Run a message and log all stats."""
    response = await session.run(message)

    stats = {
        "model": response.metadata.model_used,
        "tokens": response.metadata.tokens_used,
        "latency_ms": response.metadata.latency_ms,
        "tools_called": response.metadata.tools_called,
        "goal_progress": response.metadata.goal_progress,
        "drift_score": response.metadata.drift_score,
        "weights": session.get_weights(),
        "calibration": session.get_calibration_report(),
        "dual_process": session.get_dual_process_stats(),
        "reputation": session.get_reputation_stats(),
        "context": session.get_context_stats(),
    }

    logger.info("turn_stats=%s", json.dumps(stats))
    return response
```

!!! tip
    In production, send these stats to your observability platform (Datadog, Prometheus, OpenTelemetry) instead of logging to stdout.

## Key metrics to watch

| Metric | Source | Alert when |
|---|---|---|
| Latency | `response.metadata.latency_ms` | Sustained > 5000ms |
| Drift score | `response.metadata.drift_score` | > 0.5 for multiple turns |
| Token usage | `response.metadata.tokens_used` | Approaching model context limit |
| Tool failures | `get_reputation_stats()` | Any tool enters quarantine |
| Calibration | `get_calibration_report()` | Confidence consistently misaligned |
| Goal progress | `get_goal_progress()` | Stalled for > 5 turns |

## Session close stats

When you close a session, it returns aggregate statistics:

```python
stats = await session.close()
print(f"Session stats: {stats}")
```

This includes total tokens, total turns, average latency, and final weight state.

## Complete example

```python
import asyncio
import json
import cortex


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": "sk-..."}},
        orchestrator_model="gpt-4o",
        worker_model="gpt-4o-mini",
    )

    agent = engine.create_agent(
        name="monitored_agent",
        system_prompt="You are a helpful assistant.",
        goal_tracking=True,
        weight_config=cortex.WeightConfig(autonomy=0.7, formality=0.5),
    )

    session = agent.start_session(user_id="user_123")

    # Interaction 1
    r = await session.run("What are design patterns?")
    print(f"[Turn 1] Model: {r.metadata.model_used}, "
          f"Tokens: {r.metadata.tokens_used}, "
          f"Latency: {r.metadata.latency_ms:.0f}ms")

    # Interaction 2
    r = await session.run("Explain the Observer pattern with an example.")
    print(f"[Turn 2] Model: {r.metadata.model_used}, "
          f"Tokens: {r.metadata.tokens_used}, "
          f"Latency: {r.metadata.latency_ms:.0f}ms")

    # Full stats dump
    print("\n--- Session Stats ---")
    print(f"Weights:       {session.get_weights()}")
    print(f"Goal progress: {session.get_goal_progress()}")
    print(f"Calibration:   {session.get_calibration_report()}")
    print(f"Dual process:  {session.get_dual_process_stats()}")
    print(f"Context:       {session.get_context_stats()}")
    print(f"Columns:       {session.get_column_stats()}")
    print(f"Resources:     {session.get_resource_map()}")
    print(f"Attention:     {session.get_attention_stats()}")
    print(f"Concepts:      {session.get_concept_graph_stats()}")
    print(f"Territories:   {session.get_territory_map()}")
    print(f"Modulations:   {session.get_active_modulations()}")

    close_stats = await session.close()
    print(f"\nClose stats: {close_stats}")


asyncio.run(main())
```

!!! note
    Not all stats methods return data on every turn. Some subsystems (concept graphs, territory maps) require multiple interactions before they produce meaningful output.

---

## Next steps

- [Tune Agent Weights](../config/weight-tuning.md) -- adjust the weights you are monitoring.
- [Run What-If Simulations](what-if-simulation.md) -- test changes before they affect metrics.
- [Persist Weights Across Sessions](weight-persistence.md) -- save good configurations when metrics look healthy.
- [Manage Tool Reputation](../tools/tool-reputation.md) -- dig into tool-level metrics.
