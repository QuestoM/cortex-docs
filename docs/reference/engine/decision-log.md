# Decision Log API Reference

## Module: `corteX.engine.decision_log`

Audit trail for every agent branch point. Records decisions, alternatives considered, reasoning, outcomes, and brain signals. Enables enterprise compliance, post-hoc analysis, and "what-if" reasoning about alternative paths.

Brain analogy: Episodic memory (hippocampus) for decision recall, orbitofrontal cortex (outcome evaluation), prefrontal cortex (reasoning).

---

## Classes

### `DecisionType`

**Type**: `str, Enum`

Categories of decisions the agent makes.

| Value | Description |
|-------|-------------|
| `MODEL_SELECTION` | Which LLM model to use |
| `TOOL_SELECTION` | Which tool to invoke |
| `PLAN_STEP` | Planning step decision |
| `DRIFT_RESPONSE` | How to respond to goal drift |
| `ESCALATION` | Whether to escalate to user |
| `DELEGATION` | Whether to delegate to sub-agent |
| `BUDGET_ADJUSTMENT` | Budget expansion/contraction |
| `LOOP_RECOVERY` | How to recover from detected loop |
| `PATTERN_SELECTION` | Which mosaic pattern to use |

### `OutcomeRating`

**Type**: `str, Enum`

Post-hoc evaluation of a decision's outcome.

| Value | Description |
|-------|-------------|
| `EXCELLENT` | Optimal outcome |
| `GOOD` | Satisfactory outcome |
| `NEUTRAL` | No significant impact |
| `POOR` | Suboptimal outcome |
| `FAILURE` | Decision led to failure |
| `UNKNOWN` | Not yet evaluated |

---

### `DecisionEntry`

**Type**: `@dataclass`

A single recorded decision point.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `step_number` | `int` | *(required)* | Step at which decision was made |
| `decision_type` | `DecisionType` | *(required)* | Category of decision |
| `description` | `str` | *(required)* | What was being decided |
| `chosen` | `str` | *(required)* | The chosen alternative |
| `alternatives` | `List[str]` | `[]` | Other options that were considered |
| `reasoning` | `str` | `""` | Why this choice was made |
| `confidence` | `float` | `0.5` | Decision confidence [0.0, 1.0] |
| `outcome` | `OutcomeRating` | `OutcomeRating.UNKNOWN` | Post-hoc outcome evaluation |
| `outcome_detail` | `str` | `""` | Detail about the outcome |
| `latency_ms` | `float` | `0.0` | Decision latency |
| `cost_tokens` | `int` | `0` | Tokens consumed by this decision |
| `model_used` | `str` | `""` | Model that made the decision |
| `brain_signals` | `Dict[str, float]` | `{}` | Brain signals at decision time |
| `timestamp` | `float` | `time.time()` | Unix timestamp. Auto-set on creation. |
| `tags` | `List[str]` | `[]` | Arbitrary tags for filtering |

Only `step_number`, `decision_type`, `description`, and `chosen` are required. All other fields have sensible defaults and can be omitted when creating entries.

---

### `DecisionLog`

Immutable append-only log of agent decisions. Every branch point is recorded: model selection, tool choice, plan steps, drift responses, escalations. Supports filtering, pattern analysis, regret detection, and export.

#### Constructor

```python
DecisionLog(max_size: int = 1000)
```

**Parameters**:

- `max_size` (int): Maximum log entries. Oldest entries are evicted when full. Minimum: 50. Default: 1000

#### Properties

##### `size -> int`

Number of entries currently in the log.

```python
log = DecisionLog()
print(log.size)  # 0
log.record(entry)
print(log.size)  # 1
```

#### Methods

##### `record`

```python
def record(self, entry: DecisionEntry) -> int
```

Append a decision entry. Returns the index of the recorded entry.

##### `update_outcome`

```python
def update_outcome(self, index: int, outcome: OutcomeRating, detail: str = "") -> bool
```

Update the outcome of a previously recorded decision. Returns `True` if updated successfully.

##### `get_entry`

```python
def get_entry(self, index: int) -> Optional[DecisionEntry]
```

Get a single entry by index. Returns `None` if the index is out of range.

##### `get_history`

```python
def get_history(self, last_n: int = 20) -> List[DecisionEntry]
```

Get the most recent N decision entries.

##### `get_by_type`

```python
def get_by_type(self, decision_type: DecisionType) -> List[DecisionEntry]
```

Get all entries of a specific decision type.

##### `get_by_step`

```python
def get_by_step(self, step_number: int) -> List[DecisionEntry]
```

Get all decisions made at a specific step number. Useful for understanding all branch points at a particular execution step.

##### `get_regrets`

```python
def get_regrets(self) -> List[DecisionEntry]
```

Get decisions where the outcome was `POOR` or `FAILURE`. These are candidates for "what-if" analysis.

##### `get_high_confidence_failures`

```python
def get_high_confidence_failures(self) -> List[DecisionEntry]
```

Get decisions with confidence >= 0.7 but POOR/FAILURE outcome. These indicate calibration issues.

##### `analyze_patterns`

```python
def analyze_patterns(self) -> Dict[str, Any]
```

Analyze decision patterns across the log. Returns `Dict` with `total_decisions`, `rated_decisions`, `overall_success_rate`, `regret_count`, `calibration_failure_count`, `per_type` stats, `top_choices`, and `avg_confidence`.

##### `export`

```python
def export(self, format: str = "json") -> str
```

Export the decision log as a JSON string.

##### `clear`

```python
def clear(self) -> None
```

Clear all entries from the log.

---

## Usage Example

```python
from corteX.engine.decision_log import DecisionLog, DecisionEntry, DecisionType, OutcomeRating

log = DecisionLog()

# Record a decision (only required fields + reasoning)
idx = log.record(DecisionEntry(
    step_number=1,
    decision_type=DecisionType.MODEL_SELECTION,
    description="Select model for code generation",
    chosen="gemini-3-pro",
    alternatives=["gemini-3-flash", "gemini-2.5-pro"],
    reasoning="High complexity task requires strong model",
    confidence=0.85,
))

# Later, record outcome
log.update_outcome(idx, OutcomeRating.GOOD, "Code was correct")

# Retrieve specific entries
entry = log.get_entry(0)         # by index
step_decisions = log.get_by_step(1)  # all decisions at step 1

# Check log size
print(f"Log contains {log.size} entries")

# Analyze patterns
patterns = log.analyze_patterns()
print(f"Success rate: {patterns['overall_success_rate']:.0%}")

# Find calibration issues
failures = log.get_high_confidence_failures()
for f in failures:
    print(f"Overconfident: {f.description} (conf={f.confidence})")

# Clear the log
log.clear()
```

---

## See Also

- [Observability Concept](../../concepts/observability.md)
- [A/B Test Manager API](./ab-test-manager.md)
