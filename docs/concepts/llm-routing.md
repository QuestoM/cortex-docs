# Multi-Model Routing

corteX treats LLM model selection as a strategic optimization problem, not a static configuration. The system learns which models perform best for which task types and routes accordingly -- using Nash Equilibrium dynamics for stable allocation and Bayesian weight tracking for continuous improvement.

## What It Does

The multi-model routing system:

1. **Tracks model performance** per task type (coding, reasoning, summarization, etc.)
2. **Finds stable routing assignments** using iterated best-response dynamics (Nash Equilibrium)
3. **Selects fast vs. slow processing paths** using Kahneman-style System 1/System 2 routing
4. **Applies minimax reasoning** for high-stakes decisions where worst-case matters more than average-case

## Why: The Decision Theory Inspiration

!!! note "Decision Theory: Nash Equilibrium and Dual Process Theory"
    The routing system draws on two foundational frameworks:

    **Nash Equilibrium (1950)**: In a multi-player game, a Nash Equilibrium is a state where no player can improve their outcome by unilaterally changing their strategy. When multiple models are available, the system finds a stable assignment where each model handles the task types it is best suited for -- and no reassignment would improve overall performance.

    **Kahneman's Dual Process Theory (2011)**: Daniel Kahneman's research shows that human cognition operates in two modes: System 1 (fast, automatic, heuristic) and System 2 (slow, deliberate, analytical). Most decisions use System 1; the brain escalates to System 2 only when it detects conflict, novelty, or high stakes. The DualProcessRouter implements exactly this pattern.

    **Von Neumann's Minimax Theorem (1928)**: For high-stakes decisions, the system shifts from expected-value maximization to worst-case minimization. This mirrors how humans become more risk-averse as stakes increase -- consistent with Kahneman and Tversky's Prospect Theory.

## How It Works

### Model Performance Tracking

The `ModelSelectionWeights` tier tracks which models perform best for each task type:

```python
from corteX.engine.weights import ModelSelectionWeights

model_weights = ModelSelectionWeights()

# Initialize model scores per task type
model_weights.set_initial("coding", "gemini-pro", 0.8)
model_weights.set_initial("coding", "gemini-flash", 0.6)
model_weights.set_initial("summarization", "gemini-flash", 0.9)
model_weights.set_initial("reasoning", "gemini-pro", 0.85)

# Update after observing performance (learning rate 0.04)
model_weights.update("coding", "gemini-pro", delta=0.1)  # Good result
model_weights.update("coding", "gemini-flash", delta=-0.05)  # Mediocre result

# Query best model for a task
scores = model_weights.get_scores("coding")
# {"gemini-pro": 0.804, "gemini-flash": 0.598}
```

The system tracks 7 default task types:

| Task Type | Description |
|-----------|-------------|
| `planning` | Multi-step plan generation |
| `coding` | Code generation and modification |
| `summarization` | Content condensation |
| `validation` | Checking correctness |
| `conversation` | Free-form dialogue |
| `tool_use` | Tool invocation and orchestration |
| `reasoning` | Logic and analysis |

### Nash Routing Optimizer

The `NashRoutingOptimizer` finds stable model-to-task assignments using iterated best-response dynamics:

```python
from corteX.engine.game_theory import NashRoutingOptimizer

optimizer = NashRoutingOptimizer(
    models=["gemini-flash", "gemini-pro"],
    task_types=["coding", "summarization", "reasoning"],
)

# Record observed performance
optimizer.record_utility(
    model="gemini-flash",
    task_type="summarization",
    quality=0.9,
    latency_ms=200,
    cost=0.01,
)
optimizer.record_utility(
    model="gemini-pro",
    task_type="coding",
    quality=0.85,
    latency_ms=1500,
    cost=0.05,
)

# Run iterated best-response to find Nash Equilibrium
optimizer.iterate(steps=10)

# Query the best model for a task
best = optimizer.get_best_model("coding")
# "gemini-pro"

# Get ranked list with scores
ranked = optimizer.get_assignment("summarization")
# [("gemini-flash", 0.72), ("gemini-pro", 0.31)]
```

#### How It Converges

The optimizer uses a three-step process:

1. **Utility Observation**: For each model-task pair, track a running utility score:
   ```
   Utility = quality * speed_factor - cost
   speed_factor = 1.0 / max(latency_seconds, 0.1)
   ```
   Utilities update via exponential moving average (alpha = 0.15).

2. **Best Response**: Each model's strategy allocates probability proportional to its utility across task types. Models naturally concentrate on tasks where they have comparative advantage.

3. **Iteration**: Running best-response for each model in sequence converges toward a Nash Equilibrium -- a stable assignment where no single model benefits from switching tasks.

### Dual Process Router (System 1 / System 2)

The `DualProcessRouter` decides whether to use fast heuristic processing (System 1) or full deliberative reasoning (System 2):

```python
from corteX.engine.game_theory import DualProcessRouter, EscalationContext

router = DualProcessRouter(
    surprise_threshold=0.6,
    agreement_threshold=0.4,
    novelty_threshold=0.7,
    safety_threshold=0.8,
    drift_threshold=0.4,
)

# Most decisions use System 1 (fast path)
context = EscalationContext(
    surprise_magnitude=0.2,
    population_agreement=0.85,
    task_novelty=0.1,
)
process = router.route(context)
# ProcessType.SYSTEM1

# But high surprise triggers System 2 (slow path)
context = EscalationContext(
    surprise_magnitude=0.8,  # Prediction was wrong
    population_agreement=0.3,  # Evaluators disagree
)
process = router.route(context)
# ProcessType.SYSTEM2
# router.last_escalation_reasons:
#   ["high_surprise(0.80)", "low_agreement(0.30)"]
```

#### Seven Escalation Triggers

Any one of these conditions escalates from System 1 to System 2:

| Trigger | Threshold | What It Means |
|---------|-----------|---------------|
| **High surprise** | > 0.6 | Prediction engine was wrong; deeper analysis needed |
| **Low agreement** | < 0.4 | Population evaluators disagree; uncertain situation |
| **High novelty** | > 0.7 | No cached pattern available; unfamiliar task |
| **High safety concern** | > 0.8 | Enterprise risk too high for heuristics |
| **User request** | boolean | User explicitly asked for careful analysis |
| **Previous error** | boolean | Last step produced an error; need careful recovery |
| **Goal drift** | > 0.4 | Agent is going off-track from the stated goal |

!!! note "Brain Science: Anterior Cingulate Cortex"
    The DualProcessRouter mirrors the role of the Anterior Cingulate Cortex (ACC), which detects conflict between automatic responses and required controlled processing. When the ACC detects a mismatch -- such as the Stroop effect, where the word "RED" is printed in blue ink -- it triggers prefrontal cortex engagement for deliberate processing.

    In corteX, high surprise, low agreement, and goal drift all represent "cognitive conflict" that triggers the shift from automatic to controlled processing.

#### What Each Path Does

| Aspect | System 1 (Fast) | System 2 (Slow) |
|--------|-----------------|-----------------|
| Tool selection | PopulationDecoder vote | Full LLM reasoning |
| Model choice | Speed-optimized (e.g., Flash) | Quality-optimized (e.g., Pro) |
| Temperature | Lower | Higher |
| Planning | None (cached pattern) | Multi-step GoalTracker |
| Prompt | Simple, concise | Complex, detailed |

### Minimax Safety Guard

For high-stakes enterprise decisions, the `MinimaxSafetyGuard` shifts from expected-value maximization to worst-case minimization:

```python
from corteX.engine.game_theory import MinimaxSafetyGuard

guard = MinimaxSafetyGuard(risk_threshold=0.7)

# Register worst-case losses for each action
guard.register_worst_case("fast_model", max_loss=0.6)  # Might hallucinate
guard.register_worst_case("slow_model", max_loss=0.1)  # Very reliable
guard.register_worst_case("no_action", max_loss=0.3)   # Safe but unhelpful

# Low stakes: maximize expected gain
action = guard.select(
    candidates=["fast_model", "slow_model", "no_action"],
    expected_gains={"fast_model": 0.9, "slow_model": 0.7, "no_action": 0.0},
    enterprise_safety=0.3,  # Low concern
)
# "fast_model" (highest expected gain wins)

# High stakes: minimize worst-case loss
action = guard.select(
    candidates=["fast_model", "slow_model", "no_action"],
    expected_gains={"fast_model": 0.9, "slow_model": 0.7, "no_action": 0.0},
    enterprise_safety=0.95,  # Very high concern
)
# "slow_model" (lowest worst-case loss wins)
```

The transition is gradual, not binary:

```
score = (1 - safety_weight) * expected_gain - safety_weight * worst_loss
safety_weight = min(1.0, (enterprise_safety - risk_threshold) / 0.3)
```

Below the risk threshold (default 0.7), pure expected-value maximization applies. Above it, the safety weight increases linearly until at safety = 1.0, the decision is purely minimax.

### Monitoring

Track routing statistics to understand how the system is behaving:

```python
# Dual process stats
stats = router.get_stats()
# {
#   "system1_count": 847,
#   "system2_count": 153,
#   "system2_ratio": 0.153,
#   "total_decisions": 1000,
# }

# Nash optimizer state
optimizer_state = optimizer.to_dict()
# Includes strategies and utilities for all model-task combinations
```

A healthy system2_ratio is typically 10-20%. If it exceeds 30%, the system may be encountering too many novel or problematic situations. If it is below 5%, escalation thresholds may be too high.

## When It Activates

- **Before every LLM call**: The DualProcessRouter decides fast vs. slow path
- **During tool selection**: ModelSelectionWeights inform which model handles the task
- **At session consolidation**: NashRoutingOptimizer runs `iterate()` to update strategies
- **For high-stakes decisions**: MinimaxSafetyGuard overrides normal selection when enterprise safety is elevated
- **After each task completion**: `record_utility()` updates the optimizer with observed performance

## API Reference

```python
from corteX.engine.game_theory import (
    DualProcessRouter,
    ProcessType,
    EscalationContext,
    NashRoutingOptimizer,
    MinimaxSafetyGuard,
)
from corteX.engine.weights import ModelSelectionWeights
```
