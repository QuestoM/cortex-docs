# Approach Evaluator

The Approach Evaluator implements proactive strategy selection - the ability to ask "Is there a fundamentally better way to do this?" before and during task execution. We call this the **"Three.js Principle"**: the insight that AI agents systematically fail to evaluate whether their current approach is the right one, even when a 10x better alternative exists.

## What It Does

The Approach Evaluator operates at a **meta-cognitive level** above other brain components. While the Prediction Engine evaluates action outcomes and the Goal Tracker detects drift, the Approach Evaluator evaluates the *strategy itself*:

```
Pre-Execution:  Evaluate known approaches --> Select best fit
During:         Monitor progress/effort ratio --> Detect stagnation
Re-Evaluation:  Score alternatives --> Recommend switch (if 10x better)
Hysteresis:     Prevent oscillation --> Rising thresholds per switch
```

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Prefrontal Strategy Selection"
    The prefrontal cortex doesn't just execute plans - it **selects between competing strategies** before committing to action. The anterior cingulate cortex (ACC) continuously monitors the effort-to-reward ratio and signals when the current strategy is yielding diminishing returns.

    Cognitive flexibility research shows that the brain has a built-in "task-set switching" mechanism. When the current approach fails repeatedly, the ACC's conflict detection signal triggers the prefrontal cortex to consider alternative task-sets. Critically, this switching has a real **cost** (switch cost), which the brain accounts for - it doesn't switch impulsively.

    The Approach Evaluator models this: minimum commitment periods (like motor planning costs), rising thresholds for each switch (like the brain's increasing reluctance to switch after multiple failures), and stagnation detection (like the ACC's effort monitoring).

## How It Works

### 1. Register Known Approaches

Populate the evaluator with domain-specific approaches:

```python
from corteX.engine.approach_evaluator import Approach, ApproachEvaluator

evaluator = ApproachEvaluator()

evaluator.register_approach(Approach(
    name="canvas_2d",
    category="visualization",
    description="HTML5 Canvas 2D rendering",
    strengths=["simple", "no dependencies"],
    limitations=["slow beyond 10K points"],
    scale_limit=10_000,
    complexity_cost=0.2,
    switching_cost=0.1,
))

evaluator.register_approach(Approach(
    name="threejs",
    category="visualization",
    description="WebGL via Three.js for GPU-accelerated rendering",
    strengths=["handles 1M+ points", "GPU-accelerated"],
    limitations=["larger bundle", "steeper learning curve"],
    scale_limit=10_000_000,
    complexity_cost=0.6,
    switching_cost=0.4,
))
```

### 2. Pre-Execution Evaluation

Before starting a task, evaluate which approach fits best:

```python
result = evaluator.evaluate(
    category="visualization",
    context={"point_count": 100_000},
)

print(result.recommended.approach.name)  # "threejs"
print(result.recommended.fitness)        # 0.7
print(result.recommended.paradigm_advantage)  # 0.4
```

### 3. Monitor Progress During Execution

Track effort and progress to detect stagnation:

```python
evaluator.set_current("canvas_2d", "visualization")

# Record each step's progress and effort
evaluator.record_step(progress=0.05, effort=1.0)  # Some progress
evaluator.record_step(progress=0.01, effort=1.0)  # Slowing down
evaluator.record_step(progress=0.001, effort=1.0) # Stagnating...
```

### 4. Check for Switch Recommendations

The evaluator will recommend switching only when the benefit clearly outweighs the cost:

```python
rec = evaluator.check_switch()

if rec.should_switch:
    print(f"Switch to: {rec.recommended_approach}")
    print(f"Benefit/cost ratio: {rec.benefit_cost_ratio:.2f}")
    evaluator.accept_switch(rec.recommended_approach)
else:
    print(f"Continue: {rec.reason}")
```

## Integration with Session

The Approach Evaluator is automatically available on every Session:

```python
session = agent.create_session(user_id="user-1")

# Register domain approaches
session.register_approach(Approach(
    name="polling", category="realtime",
    strengths=["simple"], limitations=["high latency"],
    scale_limit=100, switching_cost=0.1,
))
session.register_approach(Approach(
    name="websocket", category="realtime",
    strengths=["low latency", "efficient"],
    scale_limit=100_000, switching_cost=0.3,
))

# Evaluate before task
result = session.evaluate_approach("realtime", context={"connections": 5000})

# Monitor during execution
session.set_current_approach("polling", "realtime")
session.record_approach_progress(progress=0.1, effort=1.0)

# Check switch
rec = session.check_approach_switch()
```

## LLM Context Injection

When an approach is active, its state is automatically injected into the LLM prompt via the BrainStateInjector. If stagnation is detected, the LLM sees:

```
### Approach: canvas_2d (visualization)
Steps in current approach: 8
WARNING: Stagnation detected (0.85) - consider fundamentally different approach
```

This makes the agent **self-aware** of its strategy effectiveness.

## Hysteresis: Preventing Oscillation

The evaluator includes biological-inspired hysteresis to prevent thrashing:

1. **Minimum commitment**: At least N steps before re-evaluation (default: 3)
2. **Rising thresholds**: Each switch increases the threshold for the next one
3. **Sunk cost accounting**: Progress already made is factored into switching cost
4. **Benefit/cost ratio**: Must exceed threshold (default: 3x) to recommend switch

This ensures switches happen only for **paradigm-level improvements**, not marginal gains.

## API Reference

| Method | Description |
|--------|-------------|
| `register_approach(approach)` | Register a known approach for a category |
| `evaluate(category, context)` | Score all approaches for a task |
| `set_current(name, category)` | Set the active approach |
| `record_step(progress, effort)` | Record progress for monitoring |
| `check_switch()` | Check if switching is recommended |
| `accept_switch(name)` | Accept and execute a switch |
| `get_stats()` | Get evaluator statistics |
