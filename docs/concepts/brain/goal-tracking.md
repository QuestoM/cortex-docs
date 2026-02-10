# Goal Tracking

The Goal Tracker monitors whether the agent is making progress toward its stated objective. It detects goal drift (the agent going off-topic), execution loops (repeating the same action), and alignment failures (tool choices that do not serve the goal).

## What It Does

The Goal Tracker maintains three key metrics in the Weight Engine's `goal_alignment` category:

| Metric | Range | Meaning |
|--------|-------|---------|
| `current_progress` | 0.0 - 1.0 | How far along the goal is |
| `drift_score` | 0.0 - 1.0 | How far the agent has strayed from the goal |
| `loop_risk` | 0.0 - 1.0 | Probability that the agent is stuck in a loop |

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Prefrontal Goal Maintenance"
    The prefrontal cortex (PFC) maintains goal representations in working memory and continuously monitors whether ongoing actions are aligned with those goals. When a mismatch is detected -- for example, you find yourself browsing social media instead of working on a report -- the PFC generates a conflict signal that redirects attention.

    The dorsolateral PFC specifically supports "goal maintenance under interference," which is exactly what the Goal Tracker does: it keeps the agent on track even when intermediate steps introduce distractions or dead ends.

    The Goal Tracker's drift detection mirrors the ACC's (Anterior Cingulate Cortex) error monitoring function: it continuously compares "what we are doing" against "what we should be doing" and escalates when the gap is too large.

## How It Works

### Goal Alignment Weights

Goal alignment is stored in the Weight Engine and updated through the standard weight update mechanism:

```python
from corteX.engine.weights import WeightEngine, WeightUpdate, WeightCategory

engine = WeightEngine()

# Goal alignment is one of the 7 weight categories
engine.goal_alignment
# {"current_progress": 0.0, "drift_score": 0.0, "loop_risk": 0.0}

# Update progress after a successful step
engine.apply_update(WeightUpdate(
    category=WeightCategory.GOAL_ALIGNMENT,
    key="current_progress",
    delta=0.15,
    reason="Sub-goal completed: API endpoint implemented",
))
```

### Drift Detection

Drift is detected by comparing the current activity against the stated goal. When drift exceeds the threshold, the DualProcessRouter is notified to escalate to System 2:

```python
from corteX.engine.game_theory import DualProcessRouter, EscalationContext

router = DualProcessRouter(drift_threshold=0.4)

# High drift triggers System 2 (full replanning)
context = EscalationContext(goal_drift=0.6)
process = router.route(context)
# ProcessType.SYSTEM2 -- agent needs to replan
```

### Loop Prevention

The loop risk metric increases when the agent repeats similar actions without progress. This is tracked through the Weight Engine's update history:

```python
# The orchestrator checks loop risk before each step
loop_risk = engine.goal_alignment["loop_risk"]
if loop_risk > 0.7:
    # Break the loop: try a different tool, ask the user, or replan
    pass
```

### Integration with Context Engine

The Goal Tracker feeds the `TaskState` in the Cortical Context Engine, which persists structured goal state across compression cycles:

```python
from corteX.engine.context import TaskState

state = TaskState(
    current_goal="Build a REST API with authentication",
    sub_goals=[
        {"goal": "Set up project structure", "status": "done"},
        {"goal": "Implement login endpoint", "status": "in_progress"},
        {"goal": "Add JWT validation", "status": "pending"},
    ],
    progress_percentage=35.0,
)
```

## When It Activates

- **Every step**: Progress and drift metrics are updated based on the step outcome
- **After tool execution**: Loop risk is evaluated by checking if the same tool was called with similar arguments
- **Before replanning**: Drift score is passed to the DualProcessRouter's EscalationContext
- **At compression cycles**: The TaskState is updated with current goal alignment state

## API Reference

```python
from corteX.engine.weights import WeightEngine, WeightCategory, WeightUpdate
from corteX.engine.context import TaskState
from corteX.engine.game_theory import DualProcessRouter, EscalationContext
```
