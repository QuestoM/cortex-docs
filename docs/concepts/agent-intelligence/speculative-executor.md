# Speculative Executor

The Speculative Executor pre-compiles context for the predicted next step before it is actually needed. When the prediction is correct, the agent saves a full context-compilation round-trip. When wrong, the pre-computed context is simply discarded with minimal waste. Inspired by CPU speculative execution and the brain's predictive coding model (cerebellum forward models).

## How It Works

At the end of each step, the executor evaluates whether the next step is predictable enough to justify pre-computation. It combines three confidence signals:

1. **Prediction confidence** from the brain's prediction engine (base signal)
2. **Goal progress bonus** - further along tasks are more predictable (+15% max)
3. **Historical calibration** - adjusts based on past hit/miss ratio (after 5+ attempts)
4. **Description specificity** - longer, more detailed descriptions earn a small bonus

If the combined confidence exceeds the minimum threshold (default 0.6), the executor builds a pre-compiled context dictionary containing step metadata, goal state, completed steps, available tools, and a ready-to-use prompt fragment.

On the next actual step, `validate_speculation()` checks whether the predicted step ID matches the actual one. A hit records the time savings; a miss is logged and discarded.

## Key Features

- **Zero LLM calls** - pre-compiles context only, never invokes models speculatively
- **Confidence gating** - only speculates when confidence exceeds a configurable threshold
- **Automatic calibration** - hit rate history adjusts future confidence estimates
- **Action type prediction** - classifies the predicted step as `code_generation`, `validation`, `analysis`, `search`, `debugging`, or `general` using keyword matching
- **Full statistics** via `get_stats()` - hit rate, total savings in milliseconds, average confidence
- **Bounded history** - capped at 200 entries to prevent unbounded memory growth

## Integration

The Speculative Executor works closely with:

- **Goal Tracker** - provides `goal_progress` and `goal_description` for context compilation
- **Progress Estimator** - velocity data helps predict how many steps remain
- **Brain Prediction Engine** - supplies the base `prediction_confidence` signal
- **Session Orchestrator** - validates speculation at the start of each step and uses pre-compiled context on hits

## Usage Example

```python
from corteX.engine.speculative_executor import SpeculativeExecutor

executor = SpeculativeExecutor(min_confidence=0.6)

# After completing step "parse_input", speculate on next step
spec = executor.speculate(
    current_step_id="step_1",
    next_step_id="step_2",
    next_step_description="Generate SQL migration for users table",
    prediction_confidence=0.75,
    goal_progress=0.4,
    goal_description="Build user management API",
    completed_steps=["step_1"],
    available_tools=["code_edit", "file_read"],
)

if spec:
    print(spec.predicted_action)  # "code_generation"
    print(spec.precompiled_context["prompt_fragment"])

# When the actual next step arrives, validate
is_hit = executor.validate_speculation(
    actual_step_id="step_2",
    context_compilation_ms=45.0,
)
# is_hit = True -> saved 45ms

stats = executor.get_stats()
print(f"Hit rate: {stats['hit_rate']:.0%}")
print(f"Total savings: {stats['total_savings_ms']:.0f}ms")
```

## See Also

- [Progress Estimator](progress-estimator.md) - velocity-based completion prediction
- [Brain-to-Behavior Loop](../behavior-loop.md) - how speculation fits into the overall turn cycle
- [Decision Log](decision-log.md) - speculation hits/misses can be logged as decisions
