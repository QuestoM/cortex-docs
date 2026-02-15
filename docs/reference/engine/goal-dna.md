# Goal DNA API Reference

## Module: `corteX.engine.goal_dna`

Compact token-set fingerprint for O(1) drift detection. Extracts key tokens from goal text and uses Jaccard similarity to detect when agent actions diverge from the original goal. No LLM calls needed -- pure algorithmic comparison in <0.1ms per check.

Brain analogy: Working memory maintenance (dorsolateral prefrontal cortex). The goal persists as a stable activation pattern, and each action is compared against this pattern for conflict detection. Drift history enables trend analysis.

---

## Classes

### `DriftSeverity`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `NONE` | No drift detected |
| `LOW` | Minor divergence |
| `MODERATE` | Significant divergence |
| `HIGH` | Serious drift |
| `CRITICAL` | Extreme drift from goal |

### `DriftEvent`

**Type**: `@dataclass`

A recorded drift event with context.

| Attribute | Type | Description |
|-----------|------|-------------|
| `step_number` | `int` | Step where drift occurred |
| `similarity` | `float` | Jaccard similarity score |
| `action_text` | `str` | Description of the action (truncated to 200 chars) |
| `severity` | `DriftSeverity` | Classified severity |
| `timestamp` | `float` | When the event was recorded |

### `DriftTrend`

**Type**: `@dataclass`

Analysis of drift history over time.

| Attribute | Type | Description |
|-----------|------|-------------|
| `direction` | `str` | `"improving"`, `"worsening"`, or `"stable"` |
| `average_similarity` | `float` | Mean similarity over window |
| `min_similarity` | `float` | Minimum similarity in window |
| `max_similarity` | `float` | Maximum similarity in window |
| `consecutive_drifts` | `int` | Current consecutive drift count |
| `total_drift_events` | `int` | Total drift events recorded |
| `trend_slope` | `float` | Linear regression slope (positive = improving) |

---

### `GoalDNA`

Token-set fingerprint of the goal for O(1) drift checks.

#### Constructor

```python
GoalDNA(
    goal: str,
    drift_threshold: float = 0.15,
    consecutive_limit: int = 3,
    history_size: int = 200,
)
```

**Parameters**:

- `goal` (str): The goal text to fingerprint
- `drift_threshold` (float): Similarity below this = drift. Default: 0.15
- `consecutive_limit` (int): Consecutive drift steps before `is_drifting()`. Default: 3
- `history_size` (int): Maximum history entries. Default: 200

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `goal_text` | `str` | Original goal text |
| `tokens` | `FrozenSet[str]` | Content tokens extracted from goal |
| `trigrams` | `FrozenSet[str]` | Character trigrams from goal |
| `drift_threshold` | `float` | Current drift threshold |
| `consecutive_drift_count` | `int` | Steps below threshold |
| `total_checks` | `int` | Total checks performed |

#### Methods

##### `similarity`

```python
def similarity(self, action_text: str) -> float
```

Compute Jaccard similarity between goal DNA and action text. Uses weighted fusion: **70% token-level + 30% trigram-level**. Returns 0.5 when no signal available (empty inputs).

**Token extraction**: Lowercased words > 2 chars, stop words removed, supports underscored identifiers.

##### `check_drift`

```python
def check_drift(self, step_number: int, action_text: str) -> Optional[DriftEvent]
```

Check if an action is drifting from the goal. Records the check in history. Returns a `DriftEvent` if drift is detected (similarity below threshold), `None` otherwise.

##### `is_drifting`

```python
def is_drifting(self) -> bool
```

Returns `True` if consecutive drift count exceeds the `consecutive_limit`, indicating sustained drift from the goal.

##### `get_trend`

```python
def get_trend(self, window: int = 10) -> DriftTrend
```

Analyze drift trend over recent history using linear regression slope. Direction thresholds: slope > 0.02 = improving, slope < -0.02 = worsening.

##### `get_drift_events` / `get_summary` / `reset_consecutive`

Get all drift events, state summary, or reset the consecutive counter (e.g., after replan).

---

## Severity Classification

| Similarity Range | Severity |
|-----------------|----------|
| >= threshold | NONE |
| >= threshold * 0.7 | LOW |
| >= threshold * 0.4 | MODERATE |
| >= threshold * 0.2 | HIGH |
| < threshold * 0.2 | CRITICAL |

---

## Usage Example

```python
from corteX.engine.goal_dna import GoalDNA

dna = GoalDNA("Build a REST API for user management")

# Check related action -- high similarity
sim = dna.similarity("Creating database migration for users")
# sim ~ 0.3 (related to goal)

# Check unrelated action -- low similarity
sim2 = dna.similarity("Researching quantum computing papers")
# sim2 ~ 0.0 (unrelated -- drift!)

# Drift detection in agent loop
for step_num, action in enumerate(agent_actions):
    event = dna.check_drift(step_num, action)
    if event:
        print(f"Drift at step {step_num}: {event.severity.value}")

    if dna.is_drifting():
        print("Sustained drift detected! Triggering replan...")
        dna.reset_consecutive()

# Trend analysis
trend = dna.get_trend(window=10)
print(f"Direction: {trend.direction}, slope: {trend.trend_slope}")
```

---

## See Also

- [Drift Engine API](./drift-engine.md)
- [Goal Reminder API](./goal-reminder.md)
- [Goal Tree API](./goal-tree.md)
