# Game Theory Integration

The Game Theory module provides strategic decision-making primitives for multi-agent, multi-tool environments. Two key components -- Nash Routing and Shapley Attribution -- optimize model allocation and fairly assign credit across tool pipelines.

## What It Does

| Component | Game Theory Concept | Purpose |
|-----------|-------------------|---------|
| **NashRoutingOptimizer** | Nash Equilibrium | Find stable model-to-task assignments via best-response dynamics |
| **ShapleyAttributor** | Shapley Value | Fair credit assignment when multiple tools contribute to an outcome |
| **MinimaxSafetyGuard** | Minimax Theorem | Risk minimization for high-stakes enterprise decisions |
| **TruthfulScoringMechanism** | Mechanism Design (VCG) | Incentivize honest capability reporting from tools |

All components apply **soft biases**, not hard overrides. They compose with other brain components (weights, population coding, calibration) through the standard signal aggregation pipeline.

## Nash Routing

For models `M = {m1, ..., mn}` and task types `T = {t1, ..., tk}`, the optimizer finds a stable strategy where no model benefits from unilateral deviation:

```python
from corteX.engine.game_theory import NashRoutingOptimizer

optimizer = NashRoutingOptimizer(
    models=["gemini-3-pro", "gemini-3-flash", "local-llama"],
    task_types=["coding", "reasoning", "conversation"],
)

# Record observed performance
optimizer.record_utility("gemini-3-pro", "reasoning",
                         quality=0.95, latency_ms=2000, cost=0.01)
optimizer.record_utility("gemini-3-flash", "conversation",
                         quality=0.85, latency_ms=300, cost=0.001)

# Run iterated best-response to approach equilibrium
optimizer.iterate(steps=10)

# Query optimal assignment
best = optimizer.get_best_model("reasoning")  # "gemini-3-pro"
ranked = optimizer.get_assignment("conversation")
# [("gemini-3-flash", 0.72), ("gemini-3-pro", 0.45), ...]
```

The optimizer runs **periodically** (e.g., at session consolidation), not on every step. Utility is computed as `quality * speed_factor - cost` with EMA smoothing.

## Shapley Attribution

When a pipeline uses multiple tools (search -> code_interpreter -> calculator), Shapley values answer: "How much did each tool contribute?"

```python
from corteX.engine.game_theory import ShapleyAttributor

attributor = ShapleyAttributor()

# Record coalition values (what outcomes each subset achieved)
attributor.record_coalition_value({"search"}, 0.3)
attributor.record_coalition_value({"code_interpreter"}, 0.4)
attributor.record_coalition_value({"search", "code_interpreter"}, 0.85)
attributor.record_coalition_value({"search", "code_interpreter", "calculator"}, 0.95)

# Compute fair credit
credit = attributor.compute({"search", "code_interpreter", "calculator"})
# {"search": 0.28, "code_interpreter": 0.42, "calculator": 0.25}
```

Properties guaranteed by Shapley values: efficiency (credits sum to total), symmetry (equal players get equal credit), and dummy player (non-contributors get zero).

For small tool sets (N <= 8), exact computation is used. For larger sets, Monte Carlo approximation samples random permutations.

## Minimax Safety

High-stakes enterprise decisions blend expected-value maximization with worst-case minimization:

```python
from corteX.engine.game_theory import MinimaxSafetyGuard

guard = MinimaxSafetyGuard(risk_threshold=0.7)
guard.register_worst_case("delete_records", max_loss=0.95)
guard.register_worst_case("read_records", max_loss=0.05)

action = guard.select(
    candidates=["delete_records", "read_records"],
    expected_gains={"delete_records": 0.8, "read_records": 0.6},
    enterprise_safety=0.9,  # High stakes
)
# "read_records" -- minimax avoids the high-loss action
```

## When It Activates

- **Nash Routing**: Periodically at session consolidation, not per-step
- **Shapley Attribution**: After multi-tool pipelines complete, to update tool weights
- **Minimax Safety**: Before executing actions flagged as high enterprise risk
- **All components**: Produce soft bias signals that compose with existing brain weights

## API Reference

```python
from corteX.engine.game_theory import (
    NashRoutingOptimizer,
    ShapleyAttributor,
    MinimaxSafetyGuard,
    TruthfulScoringMechanism,
    DualProcessRouter,
    ReputationSystem,
)
```
