# Goal Reminder Injector

The Goal Reminder Injector prevents the LLM from "forgetting" the original goal during long agent sessions by injecting compact, progressively compressed goal reminders into the LLM context. Like the brain's dorsolateral prefrontal cortex rehearsing goal representations in working memory, the injector periodically reminds the model what it is working toward, what pitfalls to avoid, and what approaches have already been tried.

## How It Works

The injector uses progressive detail levels based on the turn number:

| Turn Range | Mode | Token Cost | Content |
|------------|------|-----------|---------|
| 1-5 | `FULL` | ~80-120 tokens | Goal, progress (step/total + summary), current sub-goal, pitfalls to avoid, tried approaches, drift warning |
| 6-15 | `COMPACT` | ~40-60 tokens | Goal, progress (step/total + percentage), current sub-goal, drift warning, most recent pitfall |
| 16+ | `ULTRA_COMPACT` | ~20-30 tokens | Goal, progress percentage, drift warning if active |

This progressive compression mirrors how humans summarize goals over time - full detail when starting, essentials once the task is well understood.

### Drift Warnings

When the drift score exceeds the warning threshold (default 0.3) or sustained drift is detected (`is_drift_active`), a drift warning is appended to the reminder text. In FULL mode this includes the score and a refocus instruction; in ULTRA_COMPACT mode it is a terse `[DRIFT: 0.45!]` flag.

### Accumulated Context

The injector accumulates pitfalls (things to avoid) and tried approaches (past strategies) through `record_pitfall()` and `record_tried_approach()`. These deduplicate automatically and are pruned to configurable maximums (default 5 each). They appear in FULL reminders to prevent the agent from repeating mistakes.

## Key Features

- **Three verbosity modes** - full, compact, ultra-compact with configurable turn cutoffs
- **Progressive compression** - detail decreases as the session progresses
- **Drift warning injection** - automatic warning when drift score exceeds threshold
- **Pitfall tracking** - record and surface known pitfalls to avoid
- **Tried approach tracking** - prevent the agent from retrying failed strategies
- **Token-efficient** - estimated at ~4 chars per token, reminders stay under 120 tokens even in full mode
- **Context-based building** - `build_from_context(ReminderContext)` accepts a structured context object for easy integration
- **Statistics** - `get_stats()` reports total reminders generated, pitfall count, tried approach count
- **Bounded history** - reminder history capped at 50 entries

## Integration

The Goal Reminder Injector connects with:

- **Drift Engine** - drift score and `is_drift_active` flag trigger drift warnings in reminders
- **Goal Tracker** - provides `GoalProgress` with step numbers, progress percentage, and sub-goal focus
- **Session prepare pipeline** - the injector builds a reminder during `_prepare_turn()`, and the text is injected into the system prompt before the LLM call
- **Model Mosaic** - goal context from the reminder can be passed into `build_mosaic_prompts()` to keep all mosaic stages aligned
- **Loop Detector** - tried approaches from loop detection can be fed into the injector

## Usage Example

```python
from corteX.engine.goal_reminder import GoalReminderInjector, GoalProgress

injector = GoalReminderInjector(
    full_cutoff=5,
    compact_cutoff=15,
    drift_warning_threshold=0.3,
)

injector.set_goal("Build a REST API for user management with JWT auth")

# Record context as it accumulates
injector.record_pitfall("Don't use deprecated ORM methods")
injector.record_tried_approach("Tried raw SQL - too error-prone")

# Build reminder for turn 3 (full mode)
reminder = injector.build_reminder(
    progress=GoalProgress(
        current_step=3, total_steps=10,
        progress_pct=0.3, summary="DB schema complete",
        current_sub_goal="Implement JWT token generation",
    ),
    drift_score=0.1,
    turn_number=3,
)
print(reminder.text)
# [GOAL: Build a REST API...] [PROGRESS: 3/10 - DB schema complete]
# [FOCUS: Implement JWT token generation]
# [AVOID: Don't use deprecated ORM methods]
# [TRIED: Tried raw SQL - too error-prone]

print(f"Mode: {reminder.mode}")           # "full"
print(f"~{reminder.token_estimate} tokens")  # ~95

# Turn 20 - ultra compact
reminder = injector.build_reminder(
    progress=GoalProgress(progress_pct=0.8),
    drift_score=0.5,
    turn_number=20,
    is_drift_active=True,
)
print(reminder.text)
# [GOAL: Build a REST API...] [PROGRESS: 80%] [DRIFT: 0.50!]
```

## See Also

- [Goal DNA](goal-dna.md) - the fingerprinting system that detects drift
- [Drift Engine](../anti-drift/drift-engine.md) - five-signal drift scoring that triggers reminders
- [Goal Tree](goal-tree.md) - hierarchical sub-goal tracking
- [Brain-to-Behavior Loop](../behavior-loop.md) - where reminder injection happens in the pipeline
