# Adaptive Budget

The Adaptive Budget engine dynamically manages step and token budgets based on real-time velocity. When the agent makes good progress, the budget expands. When it stalls, the budget tightens. It also detects stuck states (zero progress for N consecutive steps) and enforces hard caps to prevent runaway execution. Historical velocity profiles improve initial budget estimates for recurring task types.

## How It Works

After every step, `update()` receives the current progress (0.0-1.0) and tokens consumed. The engine then:

1. **Computes velocity** - sliding-window average of progress deltas (default window: 10 steps)
2. **Compares to expected velocity** - either from a historical `TaskTypeProfile` or a default of 0.03 per step
3. **Makes a budget decision** based on the comparison:

| Condition | Decision | Effect |
|-----------|----------|--------|
| At hard cap (steps or tokens exhausted) | `HARD_CAP` | Forced stop |
| Near soft cap (80%) with low velocity | `SOFT_CAP` | Warning, prepare to stop |
| Zero progress for N steps (default 5) | `STUCK` | Alert, consider replanning |
| Velocity > 1.5x expected | `EXTEND` | Add 3 steps, increase token budget 10% |
| Velocity < 0.3x expected | `TIGHTEN` | Remove 2 steps (floor: current + 2) |
| Otherwise | `HOLD` | No change |

Budget expansion is capped at 3x the initial step budget to prevent unbounded growth.

### Historical Profiling

When a task completes, `record_task_completion()` updates the class-level `TaskTypeProfile` with an exponential moving average (alpha=0.2) of velocity, token velocity, and completion steps. Future tasks of the same type get better initial estimates via `estimate_budget(task_type)`.

Profiles can be persisted across sessions using `export_profiles()` and `import_profiles()`.

## Key Features

- **Five budget decisions** via `BudgetDecision` enum: `EXTEND`, `HOLD`, `TIGHTEN`, `STUCK`, `SOFT_CAP`, `HARD_CAP`
- **Velocity-based expansion/contraction** - budget adjusts dynamically based on measured progress rate
- **Soft cap warning** at 80% utilization - gives the agent a chance to wrap up
- **Hard cap enforcement** at 100% - prevents runaway execution
- **Stuck detection** - consecutive zero-progress steps trigger `STUCK` decision
- **Task type profiles** - class-level historical data for better initial estimates
- **Profile persistence** - `export_profiles()` / `import_profiles()` for cross-session learning
- **Token-aware** - tracks both step budget and token budget independently
- **Full state snapshot** via `BudgetState` dataclass including remaining steps, tokens, velocity, estimated steps to finish

## Integration

Adaptive Budget is deeply integrated into the corteX turn pipeline:

- **Session `run()` loop** - `AdaptiveBudget.estimate_budget()` sets initial `max_tool_rounds`, and `update()` is called after each tool execution round. `HARD_CAP` and `STUCK` decisions break the tool loop.
- **Goal Tracker** - provides the progress value passed to `update()`
- **Progress Estimator** - velocity data is complementary; the budget engine has its own sliding window optimized for decision-making
- **Drift Engine** - token consumption from the budget feeds into the drift engine's token budget ratio signal
- **Decision Log** - budget adjustment decisions are recorded as `DecisionType.BUDGET_ADJUSTMENT`

## Usage Example

```python
from corteX.engine.adaptive_budget import AdaptiveBudget

budget = AdaptiveBudget(
    step_budget=30,
    token_budget=50000,
    task_type="code_generation",
)

# Update after each step
state = budget.update(progress=0.1, tokens_used=1500)
print(f"Decision: {state.decision.value}")     # "hold"
print(f"Steps remaining: {state.steps_remaining}")
print(f"Velocity: {state.velocity:.4f}")

# Good progress -> budget extends
state = budget.update(progress=0.25, tokens_used=1200)
print(f"Decision: {state.decision.value}")     # "extend"
print(f"Step budget: {state.step_budget}")      # 33 (was 30)

# Task completes -> update profile
budget.record_task_completion()

# Persist profiles
profiles = AdaptiveBudget.export_profiles()
# ... save to disk ...

# Later, in a new session
AdaptiveBudget.import_profiles(profiles)
initial = AdaptiveBudget.estimate_budget("code_generation")
print(f"Estimated budget: {initial} steps")
```

## See Also

- [Progress Estimator](../agent-intelligence/progress-estimator.md) - complementary velocity tracking
- [Drift Engine](drift-engine.md) - token budget ratio feeds drift detection
- [Loop Detector](loop-detector.md) - stuck detection is complementary to loop detection
- [Brain-to-Behavior Loop](../behavior-loop.md) - how budget decisions control the tool loop
