# Active Forgetting Engine API Reference

## Module: `corteX.engine.cognitive.active_forgetting`

Deliberate memory removal for improved agent performance. Not all forgetting is loss: removing outdated, contradicted, or poisonous memories improves reliability. Five triggers: Contradiction, Staleness, Error Poisoning, Redundancy, Goal Divergence. Each generates a `ForgettingEvent` with provenance for potential reversal.

## Classes

### `ForgettingTrigger`

**Type**: `str, Enum`

Reasons for deliberate forgetting.

| Value | Description |
|-------|-------------|
| `CONTRADICTION` | New fact supersedes existing memory |
| `STALENESS` | Exponential decay for unaccessed memories |
| `ERROR_POISONING` | Memory present during consecutive failures |
| `REDUNDANCY` | Near-duplicate content detected |
| `GOAL_DIVERGENCE` | Content drifting from current goal |

---

### `ForgettingEvent`

**Type**: `@dataclass`

Record of a deliberate forgetting action.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_id` | `str` | ID of the memory being forgotten |
| `trigger` | `ForgettingTrigger` | Reason for forgetting |
| `old_importance` | `float` | Importance before forgetting |
| `new_importance` | `float` | Importance after forgetting |
| `reason` | `str` | Human-readable explanation |
| `step_number` | `int` | Step when forgetting occurred |
| `superseded_by` | `Optional[str]` | ID of the superseding item (if applicable) |
| `reversible` | `bool` | Whether the forgetting can be undone (default `True`) |
| `timestamp` | `float` | Unix timestamp of the event |

---

### `MemoryItem`

**Type**: `@dataclass`

Minimal memory item interface for forgetting evaluation.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_id` | `str` | Unique memory identifier |
| `content` | `str` | Text content of the memory |
| `importance` | `float` | Current importance score |
| `step_created` | `int` | Step when the memory was created |
| `last_accessed_step` | `int` | Step when last accessed |
| `metadata` | `Dict[str, Any]` | Additional metadata |

---

### `ActiveForgettingEngine`

Deliberate memory removal engine with five forgetting triggers.

#### Constructor

```python
ActiveForgettingEngine(
    staleness_half_life: int = 200,
    contradiction_decay: float = 0.05,
    poison_threshold: int = 3,
    redundancy_threshold: float = 0.85,
    divergence_threshold: float = 0.05
)
```

**Parameters**:

- `staleness_half_life` (int, default=200): Steps until importance decays by half.
- `contradiction_decay` (float, default=0.05): Importance reduction per contradiction.
- `poison_threshold` (int, default=3): Consecutive failures before an item is considered poisonous.
- `redundancy_threshold` (float, default=0.85): Jaccard similarity threshold for duplicate detection.
- `divergence_threshold` (float, default=0.05): Minimum goal proximity below which items are divergent.

#### Methods

##### `evaluate`

```python
def evaluate(
    self, item: MemoryItem, step_number: int,
    error_context: Optional[Dict[str, Any]] = None
) -> Optional[ForgettingEvent]
```

Evaluate a single item for all forgetting triggers. Checks staleness first (most common), then error poisoning.

**Parameters**:

- `item` (`MemoryItem`): Item to evaluate.
- `step_number` (`int`): Current execution step.
- `error_context` (`Optional[Dict[str, Any]]`): Dict with `is_failure` key.

**Returns**: `Optional[ForgettingEvent]` -- Event if the item should be forgotten, `None` otherwise.

##### `detect_contradiction`

```python
def detect_contradiction(self, item: MemoryItem, new_fact: str) -> bool
```

Check if a new fact contradicts an existing memory item. Uses negation pattern matching and value override detection.

**Contradiction patterns detected**:

- Direct negation: `"not X"`, `"no longer X"`, `"instead of X"`
- Value override: `"X is Y"` (old) vs `"X is Z"` (new)
- Change statements: `"changed from X to Y"`

##### `compute_staleness`

```python
def compute_staleness(self, item: MemoryItem, step_number: int) -> float
```

Compute staleness decay using exponential decay: `1 - exp(-decay_rate * age)`.

**Returns**: `float` -- Staleness from 0.0 (fresh) to 1.0 (completely stale).

##### `detect_poisoning`

```python
def detect_poisoning(self, item: MemoryItem, consecutive_failures: int) -> bool
```

Check if an item is correlated with repeated failures.

**Returns**: `bool` -- `True` if failures >= `poison_threshold`.

##### `detect_redundancy`

```python
def detect_redundancy(self, item_a: MemoryItem, item_b: MemoryItem) -> float
```

Compute content similarity using word-level Jaccard similarity.

**Returns**: `float` -- Similarity from 0.0 (no overlap) to 1.0 (identical).

##### `apply_forgetting`

```python
def apply_forgetting(self, event: ForgettingEvent, items: Dict[str, MemoryItem]) -> None
```

Apply a forgetting event: reduce importance and mark metadata.

##### `record_failure`

```python
def record_failure(self, active_item_ids: List[str]) -> List[ForgettingEvent]
```

Record a failure: increment failure counts for all active items. Returns events for items exceeding the poison threshold.

##### `record_success`

```python
def record_success(self, active_item_ids: List[str]) -> None
```

Reset failure counts for items present during success (decrements by 1).

##### `scan_redundancy`

```python
def scan_redundancy(self, items: List[MemoryItem]) -> List[ForgettingEvent]
```

Find near-duplicate memories using MD5 content hashing. Lower-importance duplicates are marked for forgetting.

##### `scan_goal_divergence`

```python
def scan_goal_divergence(
    self, items: List[MemoryItem], goal: str,
    step_number: int, min_age: int = 100
) -> List[ForgettingEvent]
```

Find items whose goal proximity has dropped below the divergence threshold. Only evaluates items older than `min_age` steps.

##### `get_forgetting_stats`

```python
def get_forgetting_stats(self) -> Dict[str, Any]
```

**Returns**: `Dict[str, Any]` with keys `total_forgotten`, `by_trigger` (dict), `active_poison_watches`.

---

## Staleness Decay Formula

```
decay_rate = ln(2) / staleness_half_life
staleness = 1 - exp(-decay_rate * age)
```

Where `age = step_number - max(last_accessed_step, step_created)`.

Items with staleness > 0.95 are automatically marked for forgetting.

---

## Example

```python
from corteX.engine.cognitive.active_forgetting import (
    ActiveForgettingEngine, MemoryItem,
)

engine = ActiveForgettingEngine(
    staleness_half_life=200,
    poison_threshold=3,
)

item = MemoryItem(
    item_id="old_config",
    content="Database port is 5432",
    importance=0.6,
    step_created=10,
    last_accessed_step=15,
)

# Check staleness at step 500
event = engine.evaluate(item, step_number=500)
if event:
    print(f"Forget: {event.trigger.value} -- {event.reason}")

# Detect contradiction
is_contradiction = engine.detect_contradiction(
    item, "Database port is 3306"
)
# True -- "port is 5432" vs "port is 3306"

# Track error poisoning
engine.record_failure(["old_config", "other_item"])
engine.record_failure(["old_config", "other_item"])
events = engine.record_failure(["old_config"])  # 3rd failure
# events contains ForgettingEvent for "old_config"

# Scan for duplicates
items = [
    MemoryItem("a", "some content", 0.8),
    MemoryItem("b", "some content", 0.3),  # duplicate of "a"
]
redundant = engine.scan_redundancy(items)
```

---

## See Also

- [Memory Crystallizer](./crystallizer.md) -- Preserves successful patterns before forgetting
- [Context Versioner](./versioner.md) -- Tracks what was in context when forgetting happened
- [Entanglement Graph](./entanglement.md) -- Consults entanglement before eviction
