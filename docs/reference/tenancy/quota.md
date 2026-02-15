# Quota Tracker API Reference

## Module: `corteX.tenancy.quota`

Per-tenant resource quota enforcement. Implements token budgets, RPM sliding windows, concurrent session limits, and tool-call-per-turn caps. All decisions are one of: `ALLOW`, `SOFT_LIMIT` (warning), or `HARD_LIMIT` (reject).

## Classes

### `QuotaDecision`

**Type**: `str, Enum`

Outcome of a quota check.

| Value | Description |
|-------|-------------|
| `ALLOW` | Within limits, proceed normally |
| `SOFT_LIMIT` | At 80%+ of limit, proceed with warning |
| `HARD_LIMIT` | At or over limit, reject the request |

---

### `QuotaEnvelope`

**Type**: `@dataclass`

Quota limits for a single tenant.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `tokens_per_day` | `int` | `1,000,000` | Daily token budget |
| `tokens_per_session` | `int` | `100,000` | Per-session token limit |
| `requests_per_minute` | `int` | `60` | RPM limit (60-second sliding window) |
| `requests_per_day` | `int` | `10,000` | Daily request limit |
| `concurrent_sessions` | `int` | `50` | Maximum simultaneous sessions |
| `max_tool_calls_per_turn` | `int` | `10` | Tool calls per agent turn |

---

### `QuotaTracker`

Runtime quota enforcement for a single tenant. Thread-safe with `threading.Lock`.

#### Constructor

```python
QuotaTracker(envelope: QuotaEnvelope)
```

**Parameters**:

- `envelope` (`QuotaEnvelope`): The quota limits to enforce.

#### Methods

##### `acquire_tokens`

```python
def acquire_tokens(self, estimated: int, session_id: str = "") -> QuotaDecision
```

Attempt to consume estimated tokens. Checks both daily and per-session limits.

**Parameters**:

- `estimated` (`int`): Number of tokens to consume.
- `session_id` (`str`): Optional session ID for per-session tracking.

**Returns**: `QuotaDecision`

**Decision logic**:

| Condition | Decision |
|-----------|----------|
| Would exceed daily limit | `HARD_LIMIT` |
| Would exceed session limit | `HARD_LIMIT` |
| At 80%+ of daily limit | `SOFT_LIMIT` |
| Otherwise | `ALLOW` |

##### `acquire_request`

```python
def acquire_request(self) -> QuotaDecision
```

Attempt one LLM request. Uses a 60-second sliding window for RPM enforcement.

**Decision logic**:

| Condition | Decision |
|-----------|----------|
| Current RPM >= limit | `HARD_LIMIT` |
| Daily requests >= limit | `HARD_LIMIT` |
| RPM at 80%+ of limit | `SOFT_LIMIT` |
| Otherwise | `ALLOW` |

##### `acquire_session`

```python
def acquire_session(self) -> QuotaDecision
```

Attempt to start a new session.

**Decision logic**:

| Condition | Decision |
|-----------|----------|
| Active sessions >= limit | `HARD_LIMIT` |
| Active sessions at 80%+ | `SOFT_LIMIT` |
| Otherwise | `ALLOW` |

##### `release_session`

```python
def release_session(self, session_id: str = "") -> None
```

Release a session slot and clear per-session token tracking.

##### `get_usage`

```python
def get_usage(self) -> Dict[str, Any]
```

Return current usage statistics.

**Returns**: `Dict[str, Any]` with keys:

| Key | Description |
|-----|-------------|
| `tokens_used_today` | Tokens consumed today |
| `tokens_limit_day` | Daily token limit |
| `tokens_remaining_day` | Remaining daily tokens |
| `requests_today` | Requests made today |
| `requests_limit_day` | Daily request limit |
| `current_rpm` | Current requests per minute |
| `rpm_limit` | RPM limit |
| `active_sessions` | Currently active sessions |
| `session_limit` | Session concurrency limit |

##### `reset_daily`

```python
def reset_daily(self) -> None
```

Manually reset daily counters (tokens, requests, per-session tracking).

---

## Auto-Reset Behavior

Daily counters automatically reset when the UTC day changes. This is checked on every `acquire_tokens` and `acquire_request` call.

---

## Soft Limit Threshold

The soft limit threshold is 80% of each resource limit. When usage crosses this threshold but is still below the hard limit, the tracker returns `SOFT_LIMIT` to allow the runtime to warn the user or adjust behavior.

---

## Example

```python
from corteX.tenancy.quota import QuotaTracker, QuotaEnvelope, QuotaDecision

envelope = QuotaEnvelope(
    tokens_per_day=500_000,
    tokens_per_session=50_000,
    requests_per_minute=30,
    concurrent_sessions=10,
)

tracker = QuotaTracker(envelope)

# Acquire a session
decision = tracker.acquire_session()
assert decision == QuotaDecision.ALLOW

# Acquire tokens
decision = tracker.acquire_tokens(2000, session_id="sess_001")
if decision == QuotaDecision.HARD_LIMIT:
    print("Token budget exceeded!")
elif decision == QuotaDecision.SOFT_LIMIT:
    print("Approaching token limit")

# Acquire a request
decision = tracker.acquire_request()
if decision == QuotaDecision.HARD_LIMIT:
    print("RPM limit reached, retry later")

# Check usage
usage = tracker.get_usage()
print(f"Tokens remaining: {usage['tokens_remaining_day']:,}")
print(f"Current RPM: {usage['current_rpm']}")
print(f"Active sessions: {usage['active_sessions']}")

# Release session
tracker.release_session("sess_001")
```

---

## Thread Safety

All public methods acquire a `threading.Lock` before modifying state. The `QuotaTracker` is safe to use from multiple threads (e.g., concurrent FastAPI request handlers).

---

## See Also

- [Tenant Manager](./manager.md) -- Creates quota trackers per tenant
- [Tenant DNA](./dna.md) -- Usage patterns inform quota sizing
- [Cost Predictor](../observability/cost-predictor.md) -- Budget prediction before execution
