# Degradation Chain API Reference

## Module: `corteX.resilience.degradation`

Graceful degradation chain for agent resilience. Formalizes a 5-level fallback system: when an agent's primary capability fails, it degrades through ordered levels rather than crashing or retrying endlessly.

Brain analogy: Cognitive load causes natural degradation - first dropping nuance, then habits (cached behavior), then reflexes (templates), then freeze/flee. This mirrors how the brain sheds non-essential functions under stress.

---

## Classes

### `DegradationLevel`

**Type**: `IntEnum`

Ordered degradation levels from full capability to honest failure.

| Value | Int | Description |
|-------|-----|-------------|
| `FULL` | 1 | All tools, best model, full context. |
| `REDUCED_TOOLS` | 2 | Non-essential tools disabled, keep core model. |
| `CACHED` | 3 | Use memory/cached responses, simpler model. |
| `TEMPLATE` | 4 | Pre-built response templates for common cases. |
| `HONEST_FAILURE` | 5 | Graceful failure with explanation of what was tried. |

---

### `DegradationState`

**Type**: `@dataclass`

Current degradation state of an agent.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `level` | `DegradationLevel` | (required) | Current degradation level. |
| `reason` | `str` | (required) | Why the agent is at this level. |
| `since` | `float` | (required) | Monotonic timestamp when this level was entered. |
| `attempts_at_level` | `int` | `0` | Number of attempts made at this level. |
| `recovery_attempts` | `int` | `0` | Total recovery attempts across the session. |

---

### `FallbackAction`

**Type**: `@dataclass`

Configuration for behavior at a specific degradation level.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `level` | `DegradationLevel` | (required) | Which level this config applies to. |
| `description` | `str` | (required) | Human-readable description of this level's behavior. |
| `disable_tools` | `List[str]` | `[]` | Tool names to disable at this level. Use `["all"]` to disable all. |
| `fallback_model` | `Optional[str]` | `None` | Alternative model to use (e.g. `"fast"`). |
| `use_cache` | `bool` | `False` | Whether to prefer cached/memory responses. |
| `template_key` | `Optional[str]` | `None` | Template key for pre-built responses. |
| `max_retries` | `int` | `2` | Retries allowed before degrading further. |

---

### `DegradationChain`

**Type**: `class`

5-level graceful degradation chain. When errors occur, the chain drops through progressively simpler modes of operation. Complements `AgentCircuitBreaker`, `FailureBoundary`, and `CascadeDetector` by adding a higher-level coordination layer for individual agent behavior.

#### Constructor

```python
DegradationChain(
    fallbacks: Optional[List[FallbackAction]] = None,
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `fallbacks` | `Optional[List[FallbackAction]]` | `None` | Custom fallback actions per level. Uses built-in 5-level defaults if not provided. |

**Default fallback chain:**

| Level | Tools disabled | Model | Cache | Max retries |
|-------|---------------|-------|-------|-------------|
| FULL | none | best | no | 3 |
| REDUCED_TOOLS | browser, code_interpreter, web_search | best | no | 2 |
| CACHED | browser, code_interpreter, web_search | fast | yes | 2 |
| TEMPLATE | browser, code_interpreter, web_search, llm | - | yes | 1 |
| HONEST_FAILURE | all | - | - | 0 |

### Methods

#### `degrade(reason) -> DegradationState`

Drop one level in the chain. If already at `HONEST_FAILURE`, increments `attempts_at_level` and returns current state.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `reason` | `str` | (required) | Why degradation is happening. |

**Returns:** New `DegradationState` after transition.

---

#### `recover() -> DegradationState`

Try to recover one level up. If already at `FULL`, returns current state unchanged.

**Returns:** New `DegradationState` after recovery.

---

#### `full_recovery() -> DegradationState`

Reset to `FULL` capability regardless of current level.

**Returns:** New `DegradationState` at FULL level.

---

#### `should_degrade(error) -> bool`

Determine if an error should trigger degradation.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `error` | `Exception` | (required) | The error that occurred. |

**Returns:** `True` if degradation should happen.

**Rules:**
- Permanent errors (`PermissionError`, `NotImplementedError`) always trigger degradation.
- Transient errors (`ConnectionError`, `TimeoutError`, `OSError`, etc.) trigger degradation only after `max_retries` at the current level are exhausted.
- At `HONEST_FAILURE` level, always returns `False`.

---

#### `record_attempt() -> None`

Record an attempt at the current degradation level. Increments `attempts_at_level`.

---

#### `current_level() -> DegradationLevel`

Get current degradation level.

---

#### `get_active_config() -> Dict[str, Any]`

Get config adjustments for current degradation level.

**Returns:** Dict with keys `level`, `level_name`, `description`, `disable_tools`, `fallback_model`, `use_cache`, `template_key`, `max_retries`.

---

#### `get_state_summary() -> Dict[str, Any]`

Return state summary for dashboard/debugging.

**Returns:** Dict with keys `current_level`, `current_level_name`, `reason`, `elapsed_seconds`, `attempts_at_level`, `recovery_attempts`, `active_config`, `history_length`, `fallback_description`.

---

#### `get_history() -> List[Dict[str, Any]]`

Return degradation history (up to 50 entries) for diagnostics.

---

### Properties

#### `state -> DegradationState`

Current degradation state (read-only snapshot).

---

## Usage Example

```python
from corteX.resilience.degradation import DegradationChain

chain = DegradationChain()

# Normal operation
print(f"Level: {chain.current_level().name}")  # FULL

# Error occurs - check if we should degrade
try:
    result = call_external_api()
except ConnectionError as e:
    chain.record_attempt()
    if chain.should_degrade(e):
        state = chain.degrade(reason=str(e))
        config = chain.get_active_config()
        print(f"Degraded to: {config['level_name']}")
        print(f"Disabled tools: {config['disable_tools']}")

# When things stabilize
chain.recover()

# Emergency: reset to full
chain.full_recovery()

# Dashboard integration
summary = chain.get_state_summary()
```

---

## See Also

- [Circuit Breaker API](../engine/circuit-breaker.md)
- [Cascade Detector API](../engine/cascade-detector.md)
- [Production Readiness API](./readiness.md)
