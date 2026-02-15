# Recovery Engine API Reference

## Module: `corteX.engine.recovery`

Intelligent error recovery with classification, retry strategies, exponential backoff, and backtracking. Classifies errors into four types and computes recovery strategies without calling external services.

Brain analogy: Thalamic filtering (classification), cerebellum (timing/backoff), prefrontal cortex (replan/backtrack), amygdala (abort threshold).

---

## Classes

### `ErrorClass`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `TRANSIENT` | Rate limit, network timeout, temporary failure |
| `PERMANENT` | Tool not found, invalid args, unsupported operation |
| `CONTEXT` | Context overflow, corrupted state |
| `FATAL` | Provider unavailable, key revoked, auth failure |

### `RecoveryAction`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `RETRY` | Retry same action with backoff |
| `RETRY_DIFFERENT` | Retry with different model/params |
| `REPLAN` | Generate a new plan from scratch |
| `SKIP` | Skip this step, continue pipeline |
| `ESCALATE` | Ask user for help |
| `ABORT` | Stop execution entirely |

### `RecoveryStrategy`

**Type**: `@dataclass`

A concrete strategy returned by the engine for the orchestrator.

| Attribute | Type | Description |
|-----------|------|-------------|
| `action` | `RecoveryAction` | Recovery action to take |
| `max_retries` | `int` | Maximum retry attempts (default: 3) |
| `backoff_base_ms` | `float` | Base backoff in milliseconds (default: 1000.0) |
| `backoff_multiplier` | `float` | Exponential multiplier (default: 2.0) |
| `alternative_model` | `Optional[str]` | Alternative model to try |
| `adjusted_params` | `Optional[Dict[str, Any]]` | Adjusted parameters for retry |
| `reason` | `str` | Human-readable reason for this strategy |

### `ErrorRecord`

**Type**: `@dataclass`

A recorded error for history tracking and pattern detection.

| Attribute | Type | Description |
|-----------|------|-------------|
| `error_class` | `ErrorClass` | Classification of the error |
| `error_message` | `str` | Error message text |
| `tool_name` | `Optional[str]` | Tool that caused the error |
| `model_name` | `Optional[str]` | Model that caused the error |
| `timestamp` | `float` | When the error occurred |
| `retry_count` | `int` | Number of retries so far |
| `recovered` | `bool` | Whether recovery succeeded |

---

### `RecoveryEngine`

Intelligent error recovery with classification, backoff, and backtracking.

#### Constructor

```python
RecoveryEngine(max_consecutive_errors: int = 5)
```

**Parameters**:

- `max_consecutive_errors` (int): Abort threshold for consecutive errors. Default: 5

#### Methods

##### `classify_error`

```python
def classify_error(
    self, error: Exception, context: Optional[Dict[str, Any]] = None,
) -> ErrorClass
```

Classify error via `isinstance` first, then pattern matching against error message text. Uses four pattern tables: transient (rate limit, timeout, 429, 503), permanent (not found, invalid, 400), context (token limit, overflow), fatal (auth, 401, 403).

##### `get_strategy`

```python
def get_strategy(
    self, error_class: ErrorClass, retry_count: int,
    has_alternative_model: bool = False,
) -> RecoveryStrategy
```

Compute recovery strategy based on error class and retry count.

**Decision matrix**:

| Error Class | retry_count < 3 | Retries exhausted (alt model) | Retries exhausted (no alt) |
|-------------|-----------------|-------------------------------|----------------------------|
| `TRANSIENT` | RETRY with backoff | RETRY_DIFFERENT | ESCALATE |
| `CONTEXT` | RETRY_DIFFERENT (compact) | REPLAN | ESCALATE |
| `PERMANENT` | RETRY_DIFFERENT (if alt) | SKIP | ESCALATE |
| `FATAL` | ABORT | ABORT | ABORT |

##### `compute_backoff_ms`

```python
def compute_backoff_ms(
    self, retry_count: int, base_ms: float = 1000.0, multiplier: float = 2.0,
) -> float
```

Exponential backoff: `base_ms * (multiplier ^ retry_count) + jitter(0-500ms)`.

##### `record_error`

```python
def record_error(
    self, error_class: ErrorClass, error_message: str,
    tool_name: Optional[str] = None, model_name: Optional[str] = None,
) -> ErrorRecord
```

Record an error and return the record. Increments consecutive error counter. History capped at 200 entries.

##### `record_recovery`

```python
def record_recovery(self, error_record: ErrorRecord) -> None
```

Mark an error as recovered and reset the consecutive error counter.

##### `should_abort`

```python
def should_abort(self) -> bool
```

Returns `True` if consecutive errors exceed `max_consecutive_errors`.

##### `get_error_summary`

```python
def get_error_summary(self) -> str
```

Human-readable error summary for LLM context injection. Shows last 5 errors and any repeated tool failure patterns (3+ failures).

##### `get_stats` / `reset`

Statistics and state reset methods.

---

## Usage Example

```python
from corteX.engine.recovery import RecoveryEngine, ErrorClass

recovery = RecoveryEngine(max_consecutive_errors=5)

try:
    result = await call_llm(prompt)
except Exception as e:
    error_class = recovery.classify_error(e)
    record = recovery.record_error(error_class, str(e), model_name="gpt-4")

    strategy = recovery.get_strategy(error_class, retry_count=0, has_alternative_model=True)

    if strategy.action.value == "retry":
        backoff = recovery.compute_backoff_ms(retry_count=0)
        await asyncio.sleep(backoff / 1000)
        # Retry the call...
        recovery.record_recovery(record)
    elif strategy.action.value == "abort":
        raise

    if recovery.should_abort():
        print("Too many consecutive errors, aborting")
```

---

## See Also

- [Agent Loop API](./agent-loop.md)
- [Provider Health API](./provider-health.md)
