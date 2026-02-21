# Adaptive Budget Engine API Reference

## Module: `corteX.engine.adaptive_budget`

Dynamic step/token budget management with velocity-based expansion and contraction. Expands budget when making good progress, contracts when stalling, and detects stuck states. Tracks historical velocities per task type for improved future estimates.

Brain analogy: Autonomic nervous system (energy regulation), hypothalamus (homeostatic control), dopaminergic reward (velocity = reward prediction error).

---

## Classes

### `BudgetDecision`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `EXTEND` | Extend budget (velocity > 1.5x expected) |
| `HOLD` | Maintain current budget |
| `TIGHTEN` | Reduce budget (velocity < 0.3x expected) |
| `STUCK` | No progress for N consecutive steps |
| `SOFT_CAP` | Approaching budget limit (>= 80%) |
| `HARD_CAP` | At budget limit (100%) |

### `BudgetState`

**Type**: `@dataclass`

Current budget snapshot.

| Attribute | Type | Description |
|-----------|------|-------------|
| `step_budget` | `int` | Current step budget |
| `token_budget` | `int` | Current token budget |
| `steps_taken` | `int` | Steps executed so far |
| `tokens_consumed` | `int` | Tokens consumed so far |
| `progress` | `float` | Current progress [0.0, 1.0] |
| `velocity` | `float` | Progress per step (moving average) |
| `token_velocity` | `float` | Progress per 1000 tokens |
| `decision` | `BudgetDecision` | Current budget decision |
| `steps_remaining` | `int` | Steps left in budget |
| `tokens_remaining` | `int` | Tokens left in budget |
| `is_near_soft_cap` | `bool` | >= 80% utilization |
| `is_at_hard_cap` | `bool` | >= 100% utilization |
| `estimated_steps_to_finish` | `Optional[int]` | Projected steps needed |
| `explanation` | `str` | Human-readable explanation |

### `VelocityRecord`

**Type**: `@dataclass`

| Attribute | Type | Description |
|-----------|------|-------------|
| `progress_delta` | `float` | Progress change in this step |
| `tokens_consumed` | `int` | Tokens used in this step |
| `timestamp` | `float` | When recorded |

### `TaskTypeProfile`

**Type**: `@dataclass`

Historical velocity profile for a task type (class-level shared state).

| Attribute | Type | Description |
|-----------|------|-------------|
| `task_type` | `str` | Task category |
| `avg_velocity` | `float` | Average progress per step |
| `avg_token_velocity` | `float` | Average progress per 1000 tokens |
| `sample_count` | `int` | Number of completed tasks |
| `total_steps` | `int` | Total steps across all tasks |
| `total_tokens` | `int` | Total tokens across all tasks |
| `avg_completion_steps` | `float` | Average steps to completion |

---

### `AdaptiveBudget`

Dynamic budget management with velocity-based expansion/contraction.

#### Constructor

```python
AdaptiveBudget(
    step_budget: int = 50,
    token_budget: int = 100000,
    soft_cap_ratio: float = 0.8,
    velocity_window: int = 10,
    stuck_threshold: int = 5,
    task_type: str = "general",
)
```

**Parameters**:

- `step_budget` (int): Initial step budget. Default: 50 (minimum: 5)
- `token_budget` (int): Initial token budget. Default: 100000 (minimum: 1000)
- `soft_cap_ratio` (float): Warning threshold. Default: 0.8 (range: 0.5-0.95)
- `velocity_window` (int): Steps for moving average. Default: 10
- `stuck_threshold` (int): Zero-progress steps before stuck. Default: 5
- `task_type` (str): Task category for historical profiling. Default: "general"

#### Methods

##### `update`

```python
def update(
    self, progress: float, tokens_used: int = 0,
    force_decision: Optional[BudgetDecision] = None,
) -> BudgetState
```

Update budget after each step. Returns `BudgetState` with the budget decision.

**Decision logic**:

| Condition | Decision |
|-----------|----------|
| At hard cap (100%) | HARD_CAP |
| Near soft cap + slow | SOFT_CAP |
| Zero progress for N steps | STUCK |
| Velocity > 1.5x expected | EXTEND (+3 steps, +10% tokens) |
| Velocity < 0.3x expected | TIGHTEN (-2 steps) |
| Normal | HOLD |

**Budget limits**: Maximum expansion = 3x initial. Minimum = current steps + 2.

##### `record_task_completion`

```python
def record_task_completion(self) -> None
```

Record completion for historical profiling. Updates `TaskTypeProfile` with exponential moving average (alpha=0.2).

##### `estimate_budget` (classmethod)

```python
@classmethod
def estimate_budget(cls, task_type: str, default_steps: int = 50) -> int
```

Estimate initial budget from historical data. Requires 3+ samples. Returns `avg_completion_steps * 1.3`.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Get budget statistics. Returns dict with keys: `step_budget`, `initial_step_budget`, `token_budget`, `steps_taken`, `tokens_consumed`, `progress`, `current_velocity`, `zero_velocity_streak`, `budget_utilization`, `decision_history` (last 10), `task_type`, `task_profile_exists`.

##### `reset`

```python
def reset(self) -> None
```

Reset budget state for a new task. Restores initial step and token budgets, clears all counters, velocities, and decision history.

---

## Usage Example

```python
from corteX.engine.adaptive_budget import AdaptiveBudget

budget = AdaptiveBudget(step_budget=30, token_budget=50000, task_type="api_build")

# After each step
state = budget.update(progress=0.1, tokens_used=2000)
print(f"Decision: {state.decision.value}")
print(f"Remaining: {state.steps_remaining} steps, {state.tokens_remaining} tokens")

state = budget.update(progress=0.3, tokens_used=3000)
# Good velocity -> may EXTEND

if state.decision.value == "stuck":
    print("Agent is stuck, consider replanning")

if state.is_at_hard_cap:
    print("Budget exhausted, must stop")

# Record for future estimates
budget.record_task_completion()

# Future tasks benefit from history
estimated = AdaptiveBudget.estimate_budget("api_build")
print(f"Estimated budget for api_build: {estimated} steps")
```

---

## See Also

- [Drift Engine API](./drift-engine.md)
- [Loop Detector API](./loop-detector.md)
- [Agent Loop API](./agent-loop.md)
