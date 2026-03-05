# Provider Health Monitor

The Provider Health Monitor tracks the operational health of every LLM provider in real time using sliding-window metrics, latency percentiles, and per-provider circuit breakers. When a provider starts failing, the circuit breaker opens to prevent cascading failures, and the monitor provides a health-ranked failover order so the orchestrator can route to healthy alternatives.

## How It Works

Every LLM call result is recorded via `record_call(provider, success, latency_ms, error)`. The monitor maintains two sliding windows per provider:

- **1-minute window** (60s) - for real-time success rate and latency percentiles
- **15-minute window** (900s) - for trend-level success rate

A provider is considered **healthy** when:
- Circuit breaker is not OPEN
- 1-minute success rate >= 80%
- p99 latency <= 30,000ms
- (New providers with < 3 calls are assumed healthy)

### Circuit Breaker

Each provider has its own circuit breaker with three states:

| State | Behavior |
|-------|----------|
| **CLOSED** | Normal operation - all calls allowed |
| **OPEN** | Provider is failing - calls blocked. Transitions to HALF_OPEN after exponential backoff (`base_s * 2^(trips-1)`) |
| **HALF_OPEN** | Probe mode - allows calls. After N consecutive successes (default 3), transitions back to CLOSED |

The breaker opens after 5 consecutive failures (configurable). Each re-trip doubles the recovery wait time.

## Key Features

- **Per-provider sliding windows** using `collections.deque` for efficient O(n) computation
- **Latency percentiles** - p50 and p99 computed from sorted 1-minute window
- **Circuit breaker with exponential backoff** - prevents hammering a failing provider
- **Health-ranked failover ordering** - `get_failover_order()` sorts providers by health, then success rate, then latency
- **ProviderHealth dataclass** - complete snapshot including `success_rate_1m`, `success_rate_15m`, `latency_p50_ms`, `latency_p99_ms`, `circuit_state`, `last_error`, `is_healthy`
- **Quick health check** - `is_healthy(provider)` and `should_allow_call(provider)` for fast routing decisions
- **Bounded memory** - max 2,000 call records per provider

## Integration

Provider Health feeds into the broader corteX routing system:

- **Model Router** - uses `should_allow_call()` before attempting a provider and `get_failover_order()` for fallback selection
- **Model Mosaic** - assigns mosaic stages to healthy providers
- **A/B Test Manager** - health data prevents sending test traffic to failing providers
- **Decision Log** - provider health state is captured in `brain_signals` for model selection decisions
- **Session Orchestrator** - the `run()` loop catches LLM failures and uses health data for retry decisions

## Usage Example

```python
from corteX.engine.provider_health import ProviderHealthMonitor

monitor = ProviderHealthMonitor()

# Record successful calls
monitor.record_call("openai", success=True, latency_ms=450)
monitor.record_call("openai", success=True, latency_ms=520)
monitor.record_call("gemini", success=True, latency_ms=380)

# Record a failure
monitor.record_call("openai", success=False, latency_ms=30000,
                     error="RateLimitError: 429")

# Check health
health = monitor.get_health("openai")
print(f"Healthy: {health.is_healthy}")
print(f"Success rate (1m): {health.success_rate_1m:.0%}")
print(f"p99 latency: {health.latency_p99_ms:.0f}ms")
print(f"Circuit: {health.circuit_state.value}")

# Get failover order (healthiest first)
order = monitor.get_failover_order(["openai", "gemini", "anthropic"])
print(f"Preferred order: {order}")

# Check circuit breaker before calling
if monitor.should_allow_call("openai"):
    response = await call_openai(...)
```

## See Also

- [Model Mosaic](model-mosaic.md) - multi-model patterns use health for stage assignment
- [A/B Test Manager](ab-test-manager.md) - health-aware test traffic routing
- [Decision Log](decision-log.md) - provider selection audit trail
- [Brain-to-Behavior Loop](../behavior-loop.md) - how health data flows through the turn pipeline
