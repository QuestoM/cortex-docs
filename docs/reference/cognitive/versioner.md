# Context Versioner API Reference

## Module: `corteX.engine.cognitive.versioner`

Context versioning with causal diff for failure diagnosis. Records the state of assembled context at each decision point. When a mistake occurs, diffs the context between the failure and last success to identify what information was missing. Tracks quality trends over time.

## Classes

### `ContextVersion`

**Type**: `@dataclass`

Record of context state at a specific decision point.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `version_id` | `str` | Format: `v_{step}_{hash}` |
| `step_number` | `int` | Execution step when recorded |
| `decision_made` | `str` | The decision made at this step |
| `context_hash` | `str` | SHA-256 hash of assembled content (first 16 chars) |
| `item_ids_present` | `List[str]` | IDs of all items in context |
| `quality_scores` | `Dict[str, float]` | Quality dimension scores (grs, dpr, overall, etc.) |
| `outcome` | `str` | Outcome of the decision (set after the fact) |
| `outcome_success` | `Optional[bool]` | Whether the outcome was successful |
| `total_tokens` | `int` | Total token count of the context |
| `timestamp` | `float` | Unix timestamp of recording |

---

### `CausalDiff`

**Type**: `@dataclass`

Diagnosis of what context differed between a success and failure.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `success_version_id` | `str` | Version ID of the successful step |
| `failure_version_id` | `str` | Version ID of the failed step |
| `missing_items` | `List[str]` | Items present in success but absent in failure |
| `extra_items` | `List[str]` | Items present in failure but absent in success |
| `quality_delta` | `Dict[str, float]` | Quality score differences (success - failure) |
| `suggested_boosts` | `List[str]` | Items to boost for recovery (up to 10) |
| `diagnosis` | `str` | Human-readable diagnosis summary |

---

### `ContextVersioner`

Tracks context evolution for causal failure diagnosis. Records what information was available at each decision point.

#### Constructor

```python
ContextVersioner(max_versions: int = 500)
```

**Parameters**:

- `max_versions` (int, default=500): Maximum version history. Oldest versions are evicted when exceeded.

#### Methods

##### `record`

```python
def record(
    self, step_number: int,
    assembled_context: List[Dict[str, str]],
    quality: Dict[str, float],
    decision: str = ""
) -> ContextVersion
```

Record a context snapshot at a decision point.

**Parameters**:

- `step_number` (`int`): Current execution step.
- `assembled_context` (`List[Dict[str, str]]`): The messages that were assembled.
- `quality` (`Dict[str, float]`): Quality scores dict (grs, overall, dpr, etc.).
- `decision` (`str`): The decision made at this step.

**Returns**: `ContextVersion` -- The recorded version.

##### `record_outcome`

```python
def record_outcome(self, step_number: int, outcome: str, success: bool) -> None
```

Record the outcome of a decision at a specific step (after execution).

##### `get_version`

```python
def get_version(self, step_number: int) -> Optional[ContextVersion]
```

Retrieve the context version for a specific step.

##### `causal_diff`

```python
def causal_diff(self, success_step: int, failure_step: int) -> Optional[CausalDiff]
```

Diff context between a success and failure step. Identifies missing items, extra items, quality deltas, and generates a diagnosis.

**Parameters**:

- `success_step` (`int`): Step number of a successful outcome.
- `failure_step` (`int`): Step number of a failed outcome.

**Returns**: `Optional[CausalDiff]` -- Diagnosis, or `None` if versions are missing.

**Diagnosis includes**:

- Count of items present in success but missing in failure
- Count of extra items in failure context
- Whether goal retention (GRS) was higher during success
- Whether decision preservation (DPR) was higher during success
- Overall quality delta

##### `diagnose_failure`

```python
def diagnose_failure(self, failure_step: int) -> Optional[CausalDiff]
```

Auto-diagnose: find the nearest prior success and diff against the failure. Searches backward through version history.

**Parameters**:

- `failure_step` (`int`): Step number of the failure.

**Returns**: `Optional[CausalDiff]` -- Diagnosis against the most recent success, or `None` if no prior success exists.

##### `get_history`

```python
def get_history(self, last_n: int = 10) -> List[ContextVersion]
```

Get the most recent context versions.

##### `get_quality_trend`

```python
def get_quality_trend(self, last_n: int = 20) -> List[Dict[str, Any]]
```

Quality trend over recent versions. Returns list of dicts with `step`, `quality`, `success`, `tokens` keys.

##### `get_success_rate`

```python
def get_success_rate(self, last_n: int = 20) -> float
```

Compute success rate over recent steps with recorded outcomes. Returns 1.0 if no outcomes are recorded.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: `Dict[str, Any]` with keys `total_versions`, `with_outcome`, `successes`, `success_rate`, `max_versions`.

---

## Example

```python
from corteX.engine.cognitive.versioner import ContextVersioner

versioner = ContextVersioner(max_versions=500)

# Record context at each step
versioner.record(
    step_number=1,
    assembled_context=[
        {"content": "Goal: Build REST API"},
        {"content": "Schema defined in models.py"},
    ],
    quality={"grs": 0.9, "dpr": 0.8, "overall": 0.85},
    decision="Generate endpoint code",
)
versioner.record_outcome(1, "Generated 3 endpoints", success=True)

versioner.record(
    step_number=2,
    assembled_context=[
        {"content": "Goal: Build REST API"},
        # Missing: schema definition
    ],
    quality={"grs": 0.7, "dpr": 0.5, "overall": 0.55},
    decision="Add validation",
)
versioner.record_outcome(2, "Validation failed -- missing schema", success=False)

# Auto-diagnose the failure
diff = versioner.diagnose_failure(failure_step=2)
if diff:
    print(diff.diagnosis)
    # "1 items present in success but missing at failure.
    #  Decision preservation was higher during success."
    print(f"Missing items: {diff.missing_items}")
    print(f"Suggested boosts: {diff.suggested_boosts}")

# Track quality trends
trend = versioner.get_quality_trend(last_n=10)
print(f"Success rate: {versioner.get_success_rate():.0%}")
```

---

## Performance Notes

- Context hashing uses SHA-256 (first 16 chars) for fast comparison
- Token estimation uses `len(content) // 4`
- Version history is bounded and evicts oldest entries automatically
- All methods are synchronous -- no I/O

---

## See Also

- [Cognitive Context Pipeline](./cognitive-pipeline.md) -- Uses versioning in Phase 8
- [Context Quality Engine](./context-quality.md) -- Provides quality scores for versioning
- [Active Forgetting Engine](./active-forgetting.md) -- Can use version history for reversal
