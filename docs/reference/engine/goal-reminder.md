# Goal Reminder Injector API Reference

## Module: `corteX.engine.goal_reminder`

Injects compact goal reminders into LLM context to prevent the LLM from "forgetting" the original goal during long sessions. Progressive detail levels compress the reminder as the conversation grows. Cost: ~50-100 tokens per injection.

Brain analogy: Dorsolateral prefrontal cortex (dlPFC) maintaining goal in working memory. Periodic rehearsal of goal representation to prevent decay -- like an internal voice repeating "remember, we're building the API." Progressive compression mirrors how humans summarize goals over time.

---

## Classes

### `ReminderMode`

**Type**: `str`

| Value | Turn Range | Token Budget |
|-------|------------|-------------|
| `"full"` | Turns 1-5 | ~80-120 tokens |
| `"compact"` | Turns 6-15 | ~40-60 tokens |
| `"ultra_compact"` | Turns 16+ | ~20-30 tokens |

### `GoalProgress`

**Type**: `@dataclass`

Current progress state for reminder construction.

| Attribute | Type | Description |
|-----------|------|-------------|
| `current_step` | `int` | Current step number |
| `total_steps` | `int` | Total planned steps |
| `progress_pct` | `float` | Progress percentage [0.0, 1.0] |
| `summary` | `str` | Progress summary text |
| `current_sub_goal` | `str` | Active sub-goal description |

### `ReminderContext`

**Type**: `@dataclass`

Full context needed to build a goal reminder.

| Attribute | Type | Description |
|-----------|------|-------------|
| `original_goal` | `str` | The original goal text |
| `progress` | `GoalProgress` | Current progress state |
| `drift_score` | `float` | Current drift score |
| `known_pitfalls` | `List[str]` | Things to avoid |
| `tried_approaches` | `List[str]` | Approaches already tried |
| `turn_number` | `int` | Current turn number |
| `is_drift_active` | `bool` | Whether sustained drift is active |

### `GoalReminder`

**Type**: `@dataclass`

A constructed goal reminder ready for injection.

| Attribute | Type | Description |
|-----------|------|-------------|
| `text` | `str` | Formatted reminder text |
| `mode` | `str` | `"full"`, `"compact"`, or `"ultra_compact"` |
| `token_estimate` | `int` | Estimated token count |
| `turn_number` | `int` | Turn this reminder was built for |
| `includes_drift_warning` | `bool` | Whether drift warning is included |
| `timestamp` | `float` | Creation timestamp |

---

### `GoalReminderInjector`

Builds and injects compact goal reminders into LLM context.

#### Constructor

```python
GoalReminderInjector(
    full_cutoff: int = 5,
    compact_cutoff: int = 15,
    max_pitfalls: int = 5,
    max_tried: int = 5,
    drift_warning_threshold: float = 0.3,
)
```

**Parameters**:

- `full_cutoff` (int): Last turn for full reminders. Default: 5
- `compact_cutoff` (int): Last turn for compact reminders. Default: 15
- `max_pitfalls` (int): Maximum pitfalls to include. Default: 5
- `max_tried` (int): Maximum tried approaches. Default: 5
- `drift_warning_threshold` (float): Drift score triggering warning. Default: 0.3

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `goal` | `str` | Current goal being tracked |
| `pitfalls` | `List[str]` | Known pitfalls |
| `tried_approaches` | `List[str]` | Approaches tried |
| `total_reminders` | `int` | Total reminders generated |

#### Methods

##### `set_goal`

```python
def set_goal(self, goal: str) -> None
```

Set or update the goal. Clears all accumulated pitfalls, tried approaches, and reminder history.

##### `record_pitfall` / `record_tried_approach`

```python
def record_pitfall(self, pitfall: str) -> None
def record_tried_approach(self, approach: str) -> None
```

Record events for future reminder inclusion. Duplicates are ignored. Lists auto-pruned at 2x max capacity.

##### `get_mode`

```python
def get_mode(self, turn_number: int) -> str
```

Determine reminder mode based on turn number.

##### `build_reminder`

```python
def build_reminder(
    self, progress: Optional[GoalProgress] = None,
    drift_score: float = 0.0, turn_number: int = 0,
    is_drift_active: bool = False,
) -> GoalReminder
```

Build a goal reminder appropriate for the current turn. Returns empty reminder if no goal is set.

**Full mode output**: `[GOAL: ...] [PROGRESS: 3/10 - 30% complete] [FOCUS: ...] [AVOID: ...] [TRIED: ...] [DRIFT WARNING: ...]`

**Compact mode**: `[GOAL: ...] [PROGRESS: 3/10 - 30%] [FOCUS: ...] [DRIFT: 0.35 - refocus]`

**Ultra-compact mode**: `[GOAL: ...] [PROGRESS: 30%] [DRIFT: 0.35!]`

##### `build_from_context`

```python
def build_from_context(self, ctx: ReminderContext) -> GoalReminder
```

Build a reminder from a full `ReminderContext` object. Merges pitfalls and tried approaches.

##### `get_stats`

Returns: goal, total_reminders, pitfalls_count, tried_approaches_count, history_size.

---

## Usage Example

```python
from corteX.engine.goal_reminder import GoalReminderInjector, GoalProgress

injector = GoalReminderInjector()
injector.set_goal("Build a REST API for user management")

# Record context
injector.record_pitfall("Don't use deprecated ORM methods")
injector.record_tried_approach("Tried SQLAlchemy, too complex")

# Build reminder for turn 3 (full mode)
reminder = injector.build_reminder(
    progress=GoalProgress(current_step=3, total_steps=10, progress_pct=0.3),
    drift_score=0.1,
    turn_number=3,
)
print(reminder.text)
# [GOAL: Build a REST API...] [PROGRESS: 3/10 - 30% complete]
# [AVOID: Don't use deprecated ORM methods] [TRIED: Tried SQLAlchemy...]

# Later turns get more compact
reminder_late = injector.build_reminder(
    progress=GoalProgress(current_step=8, total_steps=10, progress_pct=0.8),
    drift_score=0.0,
    turn_number=20,
)
print(reminder_late.mode)  # "ultra_compact"
print(f"Cost: ~{reminder_late.token_estimate} tokens")
```

---

## See Also

- [Goal DNA API](./goal-dna.md)
- [Goal Tree API](./goal-tree.md)
- [Agent Loop API](./agent-loop.md)
