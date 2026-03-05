# Progress Estimator

The Progress Estimator tracks step-by-step progress toward goal completion using velocity-based prediction. It computes sliding-window velocity (progress per step), token velocity (progress per 1,000 tokens), acceleration (velocity trend), and confidence intervals - giving the agent and developer a clear picture of how much work remains and whether the task is on track.

## How It Works

Each completed step reports its progress contribution (a float between 0 and 1) and token cost. The estimator maintains a sliding window (default 20 steps) and computes:

1. **Velocity** - average progress delta per step over the window
2. **Token velocity** - progress per 1,000 tokens over the window
3. **Acceleration** - difference between average velocity in the first half vs. second half of recorded velocities. Positive means speeding up, negative means slowing down.
4. **Estimated remaining steps** - `(1.0 - progress) / velocity`
5. **Confidence interval** - based on standard deviation of recent progress deltas

The status is assessed by comparing estimated total steps against the budget:

| Condition | Status |
|-----------|--------|
| Total estimate <= 70% of budget | `AHEAD_OF_SCHEDULE` |
| Total estimate <= 120% of budget | `ON_TRACK` |
| Velocity positive but slowing | `AT_RISK` |
| Velocity near zero | `STALLED` |
| Progress = 1.0 | `COMPLETE` |
| No steps taken yet | `JUST_STARTED` |

## Key Features

- **Six status levels** via `ProgressStatus` enum: `ON_TRACK`, `AT_RISK`, `STALLED`, `AHEAD_OF_SCHEDULE`, `JUST_STARTED`, `COMPLETE`
- **Trend detection** - `get_trend()` returns "improving", "stable", or "declining" based on acceleration thresholds
- **Completion prediction** - `predict_completion()` estimates total steps needed to finish
- **Confidence intervals** - variance-based bounds on remaining steps
- **Token-aware** - tracks both step velocity and token velocity for cost-conscious estimation
- **Human-readable explanations** - each `ProgressEstimate` includes an `explanation` string summarizing the status
- **Bounded memory** - history capped at 500 records via `deque(maxlen=...)`

## Integration

The Progress Estimator feeds into multiple corteX subsystems:

- **Adaptive Budget** - velocity data drives budget expansion/contraction decisions
- **Speculative Executor** - goal progress informs speculation confidence
- **Goal Tracker** - progress deltas come from goal verification
- **Drift Engine** - stalling progress is one of the five drift signals
- **Session Response Metadata** - `goal_progress` is surfaced to the developer

## Usage Example

```python
from corteX.engine.progress_estimator import ProgressEstimator

estimator = ProgressEstimator(velocity_window=10)

# Record progress for each step
estimator.record_step(step_number=1, progress_delta=0.05, tokens_used=1200)
estimator.record_step(step_number=2, progress_delta=0.08, tokens_used=900)
estimator.record_step(step_number=3, progress_delta=0.06, tokens_used=1100)

# Get full estimate
estimate = estimator.estimate(total_steps_budget=30, total_tokens_budget=50000)
print(f"Status: {estimate.status.value}")          # "on_track"
print(f"Velocity: {estimate.velocity:.4f}/step")   # ~0.063
print(f"Trend: {estimate.trend}")                  # "stable"
print(f"Steps remaining: {estimate.estimated_steps_remaining}")
print(f"Confidence: {estimate.confidence_interval}")
print(estimate.explanation)

# Quick completion prediction
total_steps = estimator.predict_completion()
print(f"Predicted total steps to finish: {total_steps}")
```

## See Also

- [Adaptive Budget](../anti-drift/adaptive-budget.md) - uses velocity for dynamic budget management
- [Speculative Executor](speculative-executor.md) - uses progress for speculation confidence
- [Goal Tree](../goal-intelligence/goal-tree.md) - hierarchical progress tracking
- [Brain-to-Behavior Loop](../behavior-loop.md) - where progress estimation fits in the turn cycle
