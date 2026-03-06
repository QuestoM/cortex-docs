# Approach Evaluator

`corteX.engine.approach_evaluator`

Proactive strategy selection - the "Three.js Principle." Before and during execution, evaluates whether there is a fundamentally better approach (10x improvement, not marginal). Only recommends switching when the paradigm-level gain clearly exceeds switching cost.

Inspired by prefrontal cortex (strategy selection), anterior cingulate cortex (effort monitoring, conflict detection), dopamine reward prediction error (stagnation signals), and cognitive flexibility (task-set switching when current approach fails).

This component sits ABOVE existing brain components. It consumes signals from PredictionEngine (surprise), GoalTracker (drift/stall), and ProgressEstimator (velocity decline) to decide when the current approach is fundamentally inadequate - not just struggling, but wrong.

---

## EvaluationPhase

Enum for when the evaluation is happening.

```python
class EvaluationPhase(str, Enum):
    PRE_EXECUTION = "pre_execution"    # Before starting work
    MONITORING = "monitoring"           # During execution
    RE_EVALUATION = "re_evaluation"     # Reconsidering after stagnation
```

---

## Approach

A known approach for solving a category of problems.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | required | Unique identifier within category |
| `category` | `str` | required | Task category (e.g. "visualization") |
| `description` | `str` | `""` | Human-readable description |
| `strengths` | `List[str]` | `[]` | What this approach excels at |
| `limitations` | `List[str]` | `[]` | Known weaknesses |
| `scale_limit` | `Optional[int]` | `None` | Maximum scale this approach handles well |
| `complexity_cost` | `float` | `0.5` | Cost to adopt (`0.0` = trivial, `1.0` = very complex) |
| `switching_cost` | `float` | `0.5` | Cost to switch to this approach mid-execution |

---

## ApproachScore

Scored assessment of an approach for a specific context.

| Attribute | Type | Description |
|-----------|------|-------------|
| `approach` | `Approach` | The approach being scored |
| `fitness` | `float` | How well it fits the task (`0.0` to `1.0`) |
| `paradigm_advantage` | `float` | How much better than baseline (`0.0` to `1.0`) |
| `risk` | `float` | Risk of adopting this approach (`0.0` to `1.0`) |
| `explanation` | `str` | Human-readable score breakdown |

**Ranking formula:** Approaches are sorted by `fitness - risk * 0.3` (descending).

---

## EvaluationResult

Result of evaluating approaches for a task.

| Attribute | Type | Description |
|-----------|------|-------------|
| `phase` | `EvaluationPhase` | When the evaluation happened |
| `category` | `str` | Task category evaluated |
| `recommended` | `Optional[ApproachScore]` | Best-scoring approach |
| `alternatives` | `List[ApproachScore]` | Other approaches, ranked |
| `current_approach` | `Optional[str]` | Name of currently active approach |
| `switch_benefit` | `float` | Estimated improvement from switching |
| `timestamp` | `float` | Unix timestamp (auto-populated) |

---

## SwitchRecommendation

Recommendation on whether to switch approaches mid-execution.

| Attribute | Type | Description |
|-----------|------|-------------|
| `should_switch` | `bool` | Whether switching is recommended |
| `reason` | `str` | Human-readable explanation of the decision |
| `current_approach` | `str` | Name of the current approach |
| `recommended_approach` | `Optional[str]` | Name of recommended alternative (if switching) |
| `benefit_cost_ratio` | `float` | Computed benefit/cost ratio |
| `steps_invested` | `int` | Number of steps completed in current approach |
| `progress_achieved` | `float` | Total progress so far |
| `stagnation_score` | `float` | Current stagnation level (`0.0` = healthy, `1.0` = stuck) |

---

## ApproachEvaluator

Evaluates and monitors approach selection. Pre-execution: scores known approaches. During execution: monitors progress/effort ratio and detects stagnation. Re-evaluation: recommends switching only when benefit >> cost. Hysteresis prevents oscillation via commitment periods and rising thresholds.

### Constructor

```python
ApproachEvaluator(
    switch_threshold: float = 3.0,
    min_commitment_steps: int = 3,
    stagnation_ratio: float = 0.02,
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `switch_threshold` | `float` | `3.0` | Benefit/cost ratio must exceed this to recommend switching |
| `min_commitment_steps` | `int` | `3` | Minimum steps before re-evaluation is allowed |
| `stagnation_ratio` | `float` | `0.02` | Progress/effort below this triggers stagnation |

### Methods

#### `register_approach(approach: Approach) -> None`

Register a known approach for a task category. If an approach with the same name already exists in the category, it is replaced. Maximum 200 approaches per category.

#### `list_approaches(category: str) -> List[Approach]`

List all known approaches for a category. Returns empty list if category is unknown.

#### `evaluate(category: str, context: Optional[Dict[str, Any]] = None, current_approach: Optional[str] = None) -> EvaluationResult`

Evaluate all known approaches for a task category. Scores each approach based on:

- **Scale fitness**: Context scale vs approach `scale_limit`
- **Keyword matching**: Context values matched against `strengths` and `limitations`
- **Paradigm advantage**: How much better than the 0.5 baseline
- **Risk**: Derived from `complexity_cost`

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `category` | `str` | Task category to evaluate |
| `context` | `Optional[Dict]` | Task context (keys: `scale`, `point_count`, `data_size` for scale checks) |
| `current_approach` | `Optional[str]` | Name of currently active approach (triggers re-evaluation mode) |

**Returns:** `EvaluationResult` with the best recommendation and ranked alternatives.

#### `set_current(approach_name: str, category: str) -> None`

Set the currently active approach. Clears progress history for fresh monitoring.

#### `record_step(progress: float, effort: float = 1.0) -> None`

Record a step's progress and effort for stagnation detection. Called after each action/iteration.

| Parameter | Type | Description |
|-----------|------|-------------|
| `progress` | `float` | Progress made in this step (e.g. `0.1` = 10% of goal) |
| `effort` | `float` | Effort invested (default `1.0` = one unit) |

#### `check_switch() -> SwitchRecommendation`

Check whether the current approach should be switched. Implements hysteresis:

1. **Commitment period**: Must complete `min_commitment_steps * (1 + switch_count)` steps before re-evaluating
2. **Stagnation check**: Recent progress/effort ratio must be below `stagnation_ratio`
3. **Alternative scoring**: Re-evaluates approaches, computes benefit/cost ratio
4. **Rising threshold**: Each previous switch raises the bar by 50% (`threshold * (1 + 0.5 * switch_count)`)

**Returns:** `SwitchRecommendation` with the decision and reasoning.

#### `accept_switch(new_approach: str) -> None`

Accept a switch recommendation. Increments switch counter (raising future thresholds), sets new approach as current, and clears progress history.

#### `get_stats() -> Dict[str, Any]`

Returns evaluator statistics for dashboard/monitoring:

```python
{
    "current_approach": str | None,
    "current_category": str | None,
    "switch_count": int,
    "steps_in_current": int,
    "stagnation_score": float,
    "total_categories": int,
    "total_approaches": int,
    "evaluations_performed": int,
}
```

### Example

```python
from corteX.engine.approach_evaluator import (
    Approach,
    ApproachEvaluator,
)

evaluator = ApproachEvaluator()

# Register approaches for a task category
evaluator.register_approach(Approach(
    name="canvas_2d", category="visualization",
    description="Draw points on HTML5 Canvas 2D",
    strengths=["simple", "no deps"],
    limitations=["slow >10K points"],
    scale_limit=10_000,
))
evaluator.register_approach(Approach(
    name="threejs", category="visualization",
    description="WebGL via Three.js for GPU-accelerated rendering",
    strengths=["handles 1M+ points", "GPU-accelerated"],
    limitations=["larger bundle", "learning curve"],
    scale_limit=10_000_000,
))

# Pre-execution: which approach fits best?
result = evaluator.evaluate("visualization", context={"point_count": 100_000})
best = result.recommended
print(f"Recommended: {best.approach.name} (fitness={best.fitness:.2f})")
# -> Recommended: threejs (fitness=0.70)

# Set the chosen approach and monitor progress
evaluator.set_current("canvas_2d", "visualization")

# Simulate stagnation: low progress, high effort
for _ in range(5):
    evaluator.record_step(progress=0.01, effort=2.0)

# Check if we should switch
rec = evaluator.check_switch()
if rec.should_switch:
    print(f"Switch to {rec.recommended_approach} (ratio={rec.benefit_cost_ratio:.2f})")
    evaluator.accept_switch(rec.recommended_approach)
```
