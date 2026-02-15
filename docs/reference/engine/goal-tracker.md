# Goal Tracker

`corteX.engine.goal_tracker`

Tracks progress toward the original goal across all agent steps. Detects loops through state hashing and drift through alignment scoring. The "anterior cingulate cortex" of corteX -- monitors for errors, conflict, and deviation from the intended plan.

---

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `DRIFT_WARNING` | `0.3` | Warn when drift score exceeds this threshold |
| `DRIFT_CRITICAL` | `0.6` | Force replan when drift exceeds this threshold |
| `LOOP_THRESHOLD` | `3` | Number of similar state hashes before declaring a loop |
| `PROGRESS_STALL_TURNS` | `5` | Turns without progress before intervention |

---

## GoalState

Snapshot of goal tracking at a point in time.

| Attribute | Type | Description |
|-----------|------|-------------|
| `original_goal` | `str` | The original user goal |
| `current_step` | `int` | Current step index |
| `total_steps_planned` | `int` | Number of steps in the plan |
| `progress` | `float` | Progress toward goal (`0.0` to `1.0`) |
| `drift_score` | `float` | How far off track (`0.0` = on track, `1.0` = off) |
| `loop_detected` | `bool` | Whether a loop has been detected |
| `loop_count` | `int` | Total number of loops detected |
| `stall_turns` | `int` | Consecutive turns without progress |
| `timestamp` | `float` | Unix timestamp (auto-populated) |

---

## StepVerification

Result of verifying a single step against the goal.

| Attribute | Type | Description |
|-----------|------|-------------|
| `aligned` | `bool` | Whether this step advances the goal |
| `alignment_score` | `float` | Alignment quality (`0.0` to `1.0`) |
| `drift_delta` | `float` | Change in drift since last step |
| `progress_delta` | `float` | Change in progress since last step |
| `reasoning` | `str` | Human-readable explanation of the verdict |
| `recommended_action` | `str` | One of: `"continue"`, `"adjust"`, `"replan"`, `"abort"` |

---

## GoalTracker

The main goal tracking class. Maintains the original goal, execution plan, progress metrics, loop detection state, and verification history.

### Constructor

```python
GoalTracker(goal: str)
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `goal` | `str` | The original user goal to track against |

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `original_goal` | `str` | The goal string set at construction |

### Methods

#### `set_plan(steps: List[str]) -> None`

Sets the execution plan steps. Resets the step counter to `0`.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `steps` | `List[str]` | Ordered list of plan step descriptions |

#### `get_state() -> GoalState`

Returns the current `GoalState` snapshot with progress, drift, loop detection, and stall information.

#### `async verify_step(step_description: str, step_output: str, llm_verify_fn: Optional[Any] = None) -> StepVerification`

Verifies that a completed step advances toward the original goal. Performs loop detection via state hashing, heuristic keyword alignment, and optional LLM-based semantic verification.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `step_description` | `str` | What the step intended to do |
| `step_output` | `str` | What actually happened |
| `llm_verify_fn` | `Optional[Callable]` | Async callable `(prompt: str) -> str` for LLM verification |

**Returns:** `StepVerification` with alignment score and recommended action.

**Behavior:**

1. Hashes the state and checks for loops (3+ identical hashes triggers loop detection)
2. Computes heuristic alignment via keyword overlap with the goal
3. If alignment < `0.7` and `llm_verify_fn` is provided, performs LLM verification (weighted 70% LLM / 30% heuristic)
4. Updates drift and progress scores
5. Returns a recommended action based on thresholds

#### `reset_loop_detection() -> None`

Clears all state hashes and resets the loop flag. Call after replanning.

#### `get_summary() -> Dict[str, Any]`

Returns a summary dict with: `goal`, `progress`, `drift`, `steps_completed`, `steps_planned`, `loops_detected`, `stall_turns`, `elapsed_seconds`, `verifications`, `avg_alignment`.

### Recommended Actions

The tracker recommends one of four actions based on current state:

| Action | Condition |
|--------|-----------|
| `"replan"` | Loop detected, drift >= `0.6`, or stall >= 5 turns |
| `"adjust"` | Drift >= `0.3` or alignment < `0.5` |
| `"abort"` | Alignment < `0.3` (no loop/drift override) |
| `"continue"` | Everything is on track |

### Example

```python
from corteX.engine.goal_tracker import GoalTracker

tracker = GoalTracker("Build a REST API for user management")
tracker.set_plan(["Design schema", "Implement endpoints", "Write tests"])

# After each agent step
verification = await tracker.verify_step(
    step_description="Creating database migration",
    step_output="Created users table with id, email, name columns",
    llm_verify_fn=my_llm_function,  # optional
)

if verification.recommended_action == "replan":
    tracker.reset_loop_detection()
    # ... generate new plan ...
elif verification.recommended_action == "continue":
    pass  # proceed to next step

# Check progress
state = tracker.get_state()
print(f"Progress: {state.progress:.0%}, Drift: {state.drift_score:.2f}")
```
