# Provider Health Monitor API Reference

## Module: `corteX.engine.provider_health`

Sliding-window health tracking per LLM provider. Tracks success rates, latency percentiles, and circuit breaker state. Supports health-based failover ordering.

Brain analogy: Interoception (body state monitoring), autonomic nervous system (health regulation), immune system (circuit breaker = fever response).

---

## Classes

### `CircuitState`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `CLOSED` | Healthy -- all requests allowed |
| `HALF_OPEN` | Probing -- limited requests to test recovery |
| `OPEN` | Unhealthy -- requests blocked until backoff expires |

### `ProviderHealth`

**Type**: `@dataclass`

Health snapshot for a single provider.

| Attribute | Type | Description |
|-----------|------|-------------|
| `provider` | `str` | Provider name |
| `success_rate_1m` | `float` | Success rate over last 1 minute |
| `success_rate_15m` | `float` | Success rate over last 15 minutes |
| `latency_p50_ms` | `float` | Median latency (ms) |
| `latency_p99_ms` | `float` | 99th percentile latency (ms) |
| `error_rate_1m` | `float` | Error rate over last 1 minute |
| `circuit_state` | `CircuitState` | Current circuit breaker state |
| `last_error` | `Optional[str]` | Most recent error message |
| `is_healthy` | `bool` | Overall health assessment |
| `total_calls` | `int` | Total calls recorded |
| `consecutive_failures` | `int` | Current failure streak |

---

### `ProviderHealthMonitor`

Monitors LLM provider health with sliding-window metrics. Uses `collections.deque` for efficient O(n) window computation. Supports circuit breaker per provider and health-based failover.

#### Constructor

```python
ProviderHealthMonitor(
    window_1m: int = 60,
    window_15m: int = 900,
    max_records: int = 2000,
)
```

**Parameters**:

- `window_1m` (int): 1-minute window in seconds. Default: 60
- `window_15m` (int): 15-minute window in seconds. Default: 900
- `max_records` (int): Maximum call records per provider. Default: 2000

#### Methods

##### `record_call`

```python
def record_call(self, provider: str, success: bool, latency_ms: float, error: Optional[str] = None) -> None
```

Record the result of a call to a provider. Automatically creates circuit breaker on first call to new provider.

##### `get_health`

```python
def get_health(self, provider: str) -> ProviderHealth
```

Get current health snapshot for a provider. Healthy when: success_rate_1m >= 0.8, p99 latency <= 30s, circuit not OPEN, and at least 3 total calls.

##### `get_all_health`

```python
def get_all_health(self) -> Dict[str, ProviderHealth]
```

Get health snapshots for all known providers.

##### `should_allow_call`

```python
def should_allow_call(self, provider: str) -> bool
```

Check if circuit breaker allows a call. Uses exponential backoff with `base_s * 2^(trips-1)` for recovery probing.

##### `is_healthy`

```python
def is_healthy(self, provider: str) -> bool
```

Quick health check for a provider. Returns the `is_healthy` field from `get_health()`.

##### `get_failover_order`

```python
def get_failover_order(self, providers: Optional[List[str]] = None) -> List[str]
```

Get providers ordered by health (healthiest first). Sorts by: is_healthy (desc), success_rate_1m (desc), latency_p50_ms (asc).

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Get monitor-level statistics. Returns dict with keys: `known_providers` (list of provider names), `total_calls` (dict of provider -> call count), `circuit_states` (dict of provider -> circuit state value).

---

## Circuit Breaker Behavior

The circuit breaker uses exponential backoff recovery:

1. **CLOSED**: All requests pass through. Failures increment counter.
2. **OPEN**: After 5 consecutive failures. Blocks all requests. Waits `30s * 2^(trips-1)` before probing.
3. **HALF_OPEN**: Allows probe requests. After 3 consecutive successes, returns to CLOSED. Any failure returns to OPEN.

---

## Usage Example

```python
from corteX.engine.provider_health import ProviderHealthMonitor

monitor = ProviderHealthMonitor()

# Record calls
monitor.record_call("openai", success=True, latency_ms=850)
monitor.record_call("openai", success=True, latency_ms=920)
monitor.record_call("gemini", success=False, latency_ms=5000, error="rate limit")

# Check health
health = monitor.get_health("openai")
print(f"OpenAI: healthy={health.is_healthy}, p50={health.latency_p50_ms}ms")

# Get failover order
order = monitor.get_failover_order(["openai", "gemini", "anthropic"])
print(f"Failover order: {order}")

# Check circuit breaker
if monitor.should_allow_call("gemini"):
    # Safe to call gemini
    pass
```

---

## See Also

- [Multi-Provider Failover Tutorial](../../tutorials/multi-provider.md)
- [A/B Test Manager API](./ab-test-manager.md)
