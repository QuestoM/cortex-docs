# Predictive Pre-Loader API Reference

## Module: `corteX.engine.cognitive.predictive_loader`

CPU-cache-inspired context prefetching. Predicts what context will be needed 2-3 turns ahead and pre-loads it into a prefetch buffer. Uses entity co-occurrence, plan lookahead, and error pattern matching -- no LLM calls. Tracks hit/miss rates for self-tuning.

## Classes

### `PrefetchCandidate`

**Type**: `@dataclass`

A context item predicted to be needed in upcoming steps.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_id` | `str` | Predicted item identifier |
| `predicted_relevance` | `float` | Relevance score (0.0-1.0) |
| `source_signal` | `str` | Signal that triggered the prediction: `"plan_lookahead"`, `"co_occurrence"`, or `"error_pattern"` |
| `resolution` | `int` | Default resolution level for prefetch (default 2 = R2 Compact) |

---

### `PredictivePreLoader`

CPU-cache-inspired context prefetching for upcoming turns.

#### Constructor

```python
PredictivePreLoader(
    buffer_size: int = 30,
    lookahead_steps: int = 3,
    min_relevance: float = 0.3
)
```

**Parameters**:

- `buffer_size` (int, default=30): Maximum items in the prefetch buffer.
- `lookahead_steps` (int, default=3): How many plan steps ahead to look.
- `min_relevance` (float, default=0.3): Minimum relevance to keep a candidate.

#### Methods

##### `prefetch`

```python
def prefetch(
    self, step_number: int,
    plan_state: Optional[Dict[str, Any]],
    current_context: List[Dict[str, str]]
) -> List[PrefetchCandidate]
```

Predict and pre-load context for the next 2-3 turns. Generates candidates from three signals, filters by minimum relevance, and adds top candidates to the buffer.

**Parameters**:

- `step_number` (`int`): Current execution step.
- `plan_state` (`Optional[Dict[str, Any]]`): Plan dict with `steps` (list of dicts with `description`) and `current_step_index` keys.
- `current_context` (`List[Dict[str, str]]`): Recent messages with `content` key.

**Returns**: `List[PrefetchCandidate]` -- Candidates generated (sorted by descending relevance).

**Prediction signals**:

| Signal | Source | Relevance | Description |
|--------|--------|-----------|-------------|
| Plan lookahead | `plan_state.steps` | 0.8 - (offset * 0.1) | Extracts entities from upcoming plan steps |
| Co-occurrence | Entity history | 0.5 | Entities that historically appear together |
| Error pattern | Error patterns DB | 0.7 | Matches known error signatures to pre-load resolutions |

##### `check_hits`

```python
def check_hits(self, current_items: List[str]) -> List[str]
```

Check which prefetched items are now relevant (correctly predicted).

**Parameters**:

- `current_items` (`List[str]`): Item IDs currently in active context.

**Returns**: `List[str]` -- Item IDs that were correctly predicted (hits).

##### `promote`

```python
def promote(self, item_id: str) -> bool
```

Move an item from prefetch buffer to active context.

**Parameters**:

- `item_id` (`str`): Item to promote.

**Returns**: `bool` -- `True` if the item was in the buffer.

##### `register_content`

```python
def register_content(self, item_id: str, content: str) -> None
```

Populate content for a buffered item (called by the pipeline after prefetch).

##### `update_cooccurrence`

```python
def update_cooccurrence(self, entities: List[str]) -> None
```

Update the entity co-occurrence index from observed context.

##### `record_error_resolution`

```python
def record_error_resolution(self, error_signature: str, resolution: str) -> None
```

Record a known error pattern and its resolution for future prefetching.

##### `get_status`

```python
def get_status(self) -> Dict[str, Any]
```

Prefetch performance statistics.

**Returns**: `Dict[str, Any]` with keys `buffer_size`, `total_prefetches`, `hit_count`, `miss_count`, `hit_rate`, `miss_rate`.

---

## Example

```python
from corteX.engine.cognitive.predictive_loader import PredictivePreLoader

loader = PredictivePreLoader(buffer_size=30, lookahead_steps=3)

# Record known error pattern
loader.record_error_resolution("ModuleNotFoundError", "pip install missing_module")

# Update entity co-occurrence from observed context
loader.update_cooccurrence(["models.py", "database", "User"])

# Prefetch based on plan and current context
candidates = loader.prefetch(
    step_number=5,
    plan_state={
        "steps": [
            {"description": "Read config.yaml"},
            {"description": "Parse database schema"},
            {"description": "Generate migration for User model"},
        ],
        "current_step_index": 0,
    },
    current_context=[
        {"content": "Working on models.py for User entity"},
    ],
)

# Check hits after next step
hits = loader.check_hits(["plan_schema_3"])

# Get performance stats
stats = loader.get_status()
print(f"Hit rate: {stats['hit_rate']:.1%}")
```

---

## Buffer Management

- Stale entries are evicted when `current_step > predicted_need_step + 3`
- Promoted items are also removed from the buffer
- Buffer is bounded by `buffer_size` (default 30)
- Only the top candidates (by relevance) are added to the buffer

---

## Performance Notes

- All prediction is heuristic-based -- no LLM calls
- Entity extraction uses lightweight regex patterns
- Co-occurrence predictions are capped at 20 per step
- Error pattern matching scans only the last 5 messages

---

## See Also

- [Cognitive Context Pipeline](./cognitive-pipeline.md) -- Uses prefetch in Phase 7
- [Entanglement Graph](./entanglement.md) -- Entity co-occurrence analysis
- [Context Pyramid](./pyramid.md) -- Resolution levels for prefetched items
