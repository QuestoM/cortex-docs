# Speculative Executor API Reference

## Module: `corteX.engine.speculative_executor`

Pre-compiles context for the predicted next step. Correct speculation saves a full round-trip; wrong speculation is discarded with zero cost. Does NOT execute LLM calls -- only prepares context that the orchestrator can use if speculation is correct.

Brain analogy: Predictive coding (pre-activation), cerebellum (forward models).

---

## Classes

### `SpeculativeResult`

**Type**: `@dataclass`

Pre-computed context for a predicted next step.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `next_step_id` | `str` | ID of the predicted next step |
| `next_step_description` | `str` | Description of the predicted step |
| `precompiled_context` | `Dict[str, Any]` | Pre-compiled context dict for the step |
| `confidence` | `float` | Speculation confidence [0.0, 1.0] |
| `predicted_action` | `str` | Predicted action type (e.g., `"code_generation"`, `"analysis"`) |
| `created_at` | `float` | Timestamp of creation |
| `was_used` | `Optional[bool]` | Whether this speculation was used (None=pending) |
| `savings_ms` | `float` | Time saved if speculation was correct |

---

### `SpeculativeExecutor`

Pre-computes context for the most likely next step.

#### Constructor

```python
SpeculativeExecutor(
    min_confidence: float = 0.6,
    max_history: int = 200,
)
```

**Parameters**:

- `min_confidence` (float): Minimum confidence to speculate. Below this threshold, speculation is not worth the compute. Default: 0.6
- `max_history` (int): Maximum speculations to keep in history. Default: 200

#### Methods

##### `speculate`

```python
def speculate(
    self,
    current_step_id: str,
    next_step_id: Optional[str],
    next_step_description: str = "",
    prediction_confidence: float = 0.0,
    goal_progress: float = 0.0,
    goal_description: str = "",
    completed_steps: Optional[List[str]] = None,
    available_tools: Optional[List[str]] = None,
) -> Optional[SpeculativeResult]
```

Speculate on the next step and pre-compile its context. Returns `None` if confidence is below threshold or no next step is available.

**Parameters**:

- `current_step_id` (str): ID of the currently executing step
- `next_step_id` (Optional[str]): ID of the predicted next step
- `next_step_description` (str): Human-readable description of the next step
- `prediction_confidence` (float): Base prediction confidence [0.0, 1.0]
- `goal_progress` (float): Current goal progress [0.0, 1.0]
- `goal_description` (str): Description of the current goal
- `completed_steps` (Optional[List[str]]): IDs of completed steps
- `available_tools` (Optional[List[str]]): Names of available tools

**Returns**: `Optional[SpeculativeResult]` -- Pre-compiled context if confidence is sufficient, `None` otherwise.

##### `validate_speculation`

```python
def validate_speculation(
    self,
    actual_step_id: str,
    speculation: Optional[SpeculativeResult] = None,
    context_compilation_ms: float = 0.0,
) -> bool
```

Check if the speculation was correct. Returns `True` if hit (actual step matches predicted step).

**Parameters**:

- `actual_step_id` (str): The step that actually executed
- `speculation` (Optional[SpeculativeResult]): Specific speculation to validate (uses current if None)
- `context_compilation_ms` (float): Time saved if speculation was correct

**Returns**: `bool` -- `True` if speculation was correct (hit), `False` if miss.

##### `precompile_context`

```python
def precompile_context(
    self,
    next_step_id: str,
    next_step_description: str,
    goal_description: str = "",
    goal_progress: float = 0.0,
    completed_steps: Optional[List[str]] = None,
    available_tools: Optional[List[str]] = None,
    current_step_id: str = "",
) -> Dict[str, Any]
```

Pre-compile context for the predicted next step. Returns a dict that can be used directly by the orchestrator if the speculation turns out to be correct.

**Returns**: `Dict[str, Any]` with keys: `step_id`, `step_description`, `goal`, `goal_progress`, `completed_count`, `recent_completed`, `available_tools`, `preceding_step_id`, `compiled_at`, `is_speculative`, `prompt_fragment`.

##### `get_current_speculation`

```python
def get_current_speculation(self) -> Optional[SpeculativeResult]
```

Get the current pending speculation, if any.

##### `discard_current`

```python
def discard_current(self) -> None
```

Discard the current speculation without recording it in history.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: `Dict` with `total_speculations`, `total_hits`, `total_misses`, `hit_rate`, `total_savings_ms`, `avg_confidence`, `history_size`, `has_pending`.

---

## Usage Example

```python
from corteX.engine.speculative_executor import SpeculativeExecutor

executor = SpeculativeExecutor(min_confidence=0.6)

# Speculate on next step
result = executor.speculate(
    current_step_id="step_1",
    next_step_id="step_2",
    next_step_description="Write unit tests for the API",
    prediction_confidence=0.8,
    goal_progress=0.3,
    goal_description="Build a REST API",
)

if result:
    print(f"Speculated: {result.predicted_action} (conf={result.confidence})")

    # Later, validate
    hit = executor.validate_speculation("step_2", context_compilation_ms=50.0)
    if hit:
        # Use precompiled context -- saved 50ms
        context = result.precompiled_context

stats = executor.get_stats()
print(f"Hit rate: {stats['hit_rate']:.1%}")
```

---

## Action Type Prediction

The executor predicts the action type from step description keywords:

| Action Type | Keywords |
|-------------|----------|
| `code_generation` | write, create, implement, code, build |
| `validation` | test, verify, check, validate |
| `analysis` | read, analyze, review, inspect |
| `search` | search, find, look, browse |
| `debugging` | fix, debug, resolve, repair |
| `general` | (default) |

---

## See Also

- [Prediction & Surprise Concept](../../concepts/brain/prediction.md)
- [Progress Estimator API](./progress-estimator.md)
