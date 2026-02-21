# Progress Estimator API Reference

## Module: `corteX.engine.progress_estimator`

Velocity-based progress prediction with trend analysis. Tracks step-by-step progress deltas, computes sliding-window velocity, detects acceleration/deceleration, and provides confidence intervals.

Brain analogy: Cerebellum (timing/prediction), dopamine system (progress = reward).

---

## Classes

### `ProgressStatus`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `ON_TRACK` | Progressing within budget |
| `AT_RISK` | May exceed budget |
| `STALLED` | Zero or near-zero velocity |
| `AHEAD_OF_SCHEDULE` | Progressing faster than budget allows |
| `JUST_STARTED` | No data yet |
| `COMPLETE` | Task complete |

### `ProgressEstimate`

**Type**: `@dataclass`

Snapshot of estimated progress toward goal completion.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `estimated_steps_remaining` | `Optional[int]` | Estimated steps to complete (None if unable) |
| `estimated_tokens_remaining` | `Optional[int]` | Estimated tokens to complete |
| `velocity` | `float` | Progress per step (sliding window) |
| `token_velocity` | `float` | Progress per 1000 tokens |
| `acceleration` | `float` | Change in velocity (positive = speeding up) |
| `status` | `ProgressStatus` | Current progress assessment |
| `confidence_interval` | `Tuple[float, float]` | (low, high) bounds for remaining steps |
| `current_progress` | `float` | Current progress [0.0, 1.0] |
| `steps_taken` | `int` | Total steps completed |
| `tokens_used` | `int` | Total tokens consumed |
| `trend` | `str` | `"improving"`, `"stable"`, or `"declining"` |
| `explanation` | `str` | Human-readable explanation |

---

### `ProgressEstimator`

Estimates remaining work based on velocity and acceleration.

#### Constructor

```python
ProgressEstimator(
    velocity_window: int = 20,
    max_history: int = 500,
)
```

**Parameters**:

- `velocity_window` (int): Sliding window size for velocity computation. Default: 20
- `max_history` (int): Maximum step records to keep. Default: 500

#### Methods

##### `record_step`

```python
def record_step(self, step_number: int, progress_delta: float, tokens_used: int = 0) -> None
```

Record a completed step's progress contribution. `progress_delta` is clamped to [0.0, 1.0].

##### `estimate`

```python
def estimate(
    self,
    current_progress: Optional[float] = None,
    total_steps_budget: int = 50,
    total_tokens_budget: int = 100000,
) -> ProgressEstimate
```

Compute a full progress estimate based on current velocity, acceleration, and trend.

##### `get_trend`

```python
def get_trend(self) -> str
```

Returns `"improving"` (acceleration > 0.005), `"declining"` (< -0.005), or `"stable"`.

##### `predict_completion`

```python
def predict_completion(self, current_velocity: Optional[float] = None) -> Optional[int]
```

Estimate total steps needed to complete. Returns `None` if velocity is near zero.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Get estimator statistics. Returns dict with keys: `total_steps`, `total_tokens`, `current_progress`, `velocity`, `acceleration`, `trend`, `history_size`.

##### `reset`

```python
def reset(self) -> None
```

Reset all state for a new task.

---

## Usage Example

```python
from corteX.engine.progress_estimator import ProgressEstimator

estimator = ProgressEstimator(velocity_window=10)

# Record steps as they complete
estimator.record_step(1, progress_delta=0.1, tokens_used=2000)
estimator.record_step(2, progress_delta=0.12, tokens_used=1800)
estimator.record_step(3, progress_delta=0.08, tokens_used=2200)

# Get estimate
est = estimator.estimate(total_steps_budget=20)
print(f"Status: {est.status.value}")
print(f"Steps remaining: {est.estimated_steps_remaining}")
print(f"Velocity: {est.velocity:.4f}/step, Trend: {est.trend}")
print(f"CI: {est.confidence_interval}")
```

---

## See Also

- [Goal Tracking Concept](../../concepts/brain/goal-tracking.md)
- [Adaptive Budget API](./adaptive-budget.md)
