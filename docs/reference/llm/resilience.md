# Resilience Primitives API Reference

## Module: `corteX.core.llm.resilience`

Lightweight resilience primitives for LLM routing: circuit breaker and rate limiter. Zero external dependencies.

## Enums

### `CircuitState`

```python
class CircuitState(str, Enum)
```

| Value | Description |
|-------|-------------|
| `CLOSED` | Normal operation -- requests flow through |
| `OPEN` | Failing -- requests are rejected immediately |
| `HALF_OPEN` | Probing -- exactly one request is allowed to test recovery |

---

## Classes

### `CircuitBreaker`

Per-provider circuit breaker. After `failure_threshold` consecutive failures, the circuit opens for `recovery_timeout` seconds. During that window all calls are rejected immediately. After the timeout, the circuit moves to half-open and allows exactly one probe request. A success resets to closed; a failure reopens.

#### Constructor

```python
CircuitBreaker(
    failure_threshold: int = 3,
    recovery_timeout: float = 30.0,
)
```

**Parameters**:

- `failure_threshold` (int, default=3): Number of consecutive failures that trips the circuit open
- `recovery_timeout` (float, default=30.0): Seconds to wait before allowing a probe request

#### Methods

##### `is_available`

```python
def is_available(self, provider_key: str) -> bool
```

Check whether a provider is allowed to receive requests. Returns `True` if the circuit is CLOSED or HALF_OPEN (after timeout), `False` if OPEN.

##### `record_success`

```python
def record_success(self, provider_key: str) -> None
```

Record a successful call. Resets consecutive failure count and closes the circuit.

##### `record_failure`

```python
def record_failure(self, provider_key: str) -> None
```

Record a failed call. Increments the failure counter. If the circuit is HALF_OPEN, it reopens immediately. If consecutive failures reach the threshold, the circuit opens.

##### `get_state`

```python
def get_state(self, provider_key: str) -> CircuitState
```

Return the current circuit state for observability/debugging.

##### `reset`

```python
def reset(self, provider_key: str) -> None
```

Manually reset a circuit to its initial state (e.g., after a configuration change).

**Example**:

```python
from corteX.core.llm.resilience import CircuitBreaker

cb = CircuitBreaker(failure_threshold=3, recovery_timeout=30.0)

# Normal operation
assert cb.is_available("openai") is True

# Simulate failures
cb.record_failure("openai")
cb.record_failure("openai")
cb.record_failure("openai")  # Circuit opens

assert cb.is_available("openai") is False

# After recovery_timeout seconds, circuit goes half-open
# A successful probe closes it
cb.record_success("openai")
assert cb.is_available("openai") is True
```

---

### `RateLimiter`

Per-provider sliding-window rate limiter. Tracks request timestamps over a 60-second window and enforces a configurable RPM (requests per minute) limit. When the limit is reached, `acquire()` returns `False` so the router can proactively switch to a backup provider instead of waiting for a 429 response.

#### Constructor

```python
RateLimiter(default_rpm: int = 60)
```

**Parameters**:

- `default_rpm` (int, default=60): Default requests-per-minute limit for all providers

#### Methods

##### `set_limit`

```python
def set_limit(self, provider_key: str, rpm: int) -> None
```

Configure RPM limit for a specific provider.

##### `get_limit`

```python
def get_limit(self, provider_key: str) -> int
```

Return configured RPM for a provider. Falls back to `default_rpm`.

##### `acquire`

```python
def acquire(self, provider_key: str) -> bool
```

Try to acquire a request slot. Returns `True` if the request is allowed, `False` if the rate limit would be exceeded. Automatically prunes timestamps older than 60 seconds.

##### `remaining`

```python
def remaining(self, provider_key: str) -> int
```

Return how many requests are still available in the current 60-second window.

**Example**:

```python
from corteX.core.llm.resilience import RateLimiter

rl = RateLimiter(default_rpm=60)
rl.set_limit("gemini", 25)  # Gemini tier-1: 25 RPM

# Before each request
if rl.acquire("gemini"):
    # Proceed with request
    pass
else:
    # Switch to alternative provider
    print(f"Rate limit reached, {rl.remaining('gemini')} slots left")
```

---

## Integration with LLMRouter

Both primitives are automatically instantiated and used by `LLMRouter`:

```python
class LLMRouter:
    def __init__(self, circuit_failure_threshold=3, circuit_recovery_timeout=30.0, default_rpm=60):
        self._circuit_breaker = CircuitBreaker(
            failure_threshold=circuit_failure_threshold,
            recovery_timeout=circuit_recovery_timeout,
        )
        self._rate_limiter = RateLimiter(default_rpm=default_rpm)
```

During `generate()`:

1. **Rate limiter** is checked proactively before sending the request. If the limit is approached, the router switches to an alternative provider
2. **Circuit breaker** is checked during model selection. Providers with open circuits are skipped
3. On success: `record_success()` is called on the circuit breaker
4. On failure: `record_failure()` is called, and the router falls back to alternatives

---

## See Also

- [LLM Router API](./router.md) - Uses both primitives for resilient routing
- [Multi-Provider Failover Tutorial](../../tutorials/multi-provider.md)
