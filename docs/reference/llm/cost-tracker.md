# Cost Tracker API Reference

## Module: `corteX.core.llm.cost_tracker`

Per-session, per-task, and per-tenant token budget enforcement. Tracks LLM call costs, enforces budget limits, and detects cost anomalies.

Brain analogy: The hypothalamus regulating metabolic energy expenditure.

## Enums

### `BudgetLevel`

```python
class BudgetLevel(str, Enum)
```

| Value | Description |
|-------|-------------|
| `OK` | Within budget |
| `SOFT_LIMIT` | Approaching budget limit (warning issued) |
| `HARD_LIMIT` | Budget exceeded (request blocked) |

---

## Data Classes

### `CostRecord`

**Type**: `@dataclass`

A single LLM call cost record.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `provider` | `str` | -- | Provider name (e.g., `"openai"`) |
| `model` | `str` | -- | Model ID |
| `input_tokens` | `int` | -- | Input tokens consumed |
| `output_tokens` | `int` | -- | Output tokens generated |
| `cost_usd` | `float` | -- | Estimated cost in USD |
| `session_id` | `str` | -- | Session identifier |
| `task_id` | `Optional[str]` | `None` | Task identifier |
| `tenant_id` | `Optional[str]` | `None` | Tenant identifier |
| `timestamp` | `float` | `time.time()` | Unix timestamp of the call |
| `metadata` | `Dict[str, Any]` | `{}` | Additional metadata |

### `BudgetPolicy`

**Type**: `@dataclass`

Budget limits for a scope (session or tenant).

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `soft_limit_usd` | `float` | `inf` | Warning threshold in USD |
| `hard_limit_usd` | `float` | `inf` | Hard cap in USD (blocks requests) |
| `daily_limit_usd` | `float` | `inf` | Per-day budget for tenants |
| `warn_at_percent` | `float` | `0.8` | Percentage of daily limit that triggers warning |

#### Methods

- `is_unlimited() -> bool`: Returns `True` if all limits are infinite.

### `BudgetCheck`

**Type**: `@dataclass`

Result of a budget check.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `level` | `BudgetLevel` | OK, SOFT_LIMIT, or HARD_LIMIT |
| `current_cost_usd` | `float` | Current total spend |
| `limit_usd` | `float` | Applicable limit |
| `remaining_usd` | `float` | Budget remaining |
| `message` | `str` | Human-readable status message |

### `CostAnomaly`

**Type**: `@dataclass`

A detected cost anomaly (3x+ deviation from rolling average).

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `record` | `CostRecord` | The anomalous call record |
| `expected_cost_usd` | `float` | Rolling average cost |
| `actual_cost_usd` | `float` | Actual call cost |
| `ratio` | `float` | Actual / expected ratio |
| `message` | `str` | Description of the anomaly |

### `CostSummary`

**Type**: `@dataclass`

Aggregated cost statistics.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `total_cost_usd` | `float` | Total spend |
| `total_input_tokens` | `int` | Total input tokens |
| `total_output_tokens` | `int` | Total output tokens |
| `call_count` | `int` | Number of LLM calls |
| `cost_by_model` | `Dict[str, float]` | Spend per model |
| `cost_by_provider` | `Dict[str, float]` | Spend per provider |

---

## Classes

### `CostTracker`

Track and enforce token budgets across sessions, tasks, and tenants.

#### Constructor

```python
CostTracker()
```

Creates a new tracker with no records and no budget policies.

#### Methods

##### `record_call`

```python
def record_call(
    self, provider: str, model: str, input_tokens: int,
    output_tokens: int, cost_usd: float, session_id: str,
    task_id: Optional[str] = None, tenant_id: Optional[str] = None,
    metadata: Optional[Dict[str, Any]] = None,
) -> CostRecord
```

Record a completed LLM call. Maintains a rolling window of 50 records per model for anomaly detection.

##### `set_session_budget`

```python
def set_session_budget(self, session_id: str, policy: BudgetPolicy) -> None
```

Configure budget for a session.

##### `set_tenant_budget`

```python
def set_tenant_budget(self, tenant_id: str, policy: BudgetPolicy) -> None
```

Configure budget for a tenant.

##### `check_budget`

```python
def check_budget(
    self, session_id: str, tenant_id: Optional[str] = None,
    estimated_cost: float = 0.0,
) -> BudgetCheck
```

Check if a call is within budget. Checks session budget first, then tenant daily budget.

**Example**:

```python
from corteX.core.llm.cost_tracker import CostTracker, BudgetPolicy, BudgetLevel

tracker = CostTracker()
tracker.set_session_budget("session-1", BudgetPolicy(
    soft_limit_usd=0.50,
    hard_limit_usd=1.00,
))

# Before each LLM call
check = tracker.check_budget("session-1", estimated_cost=0.02)
if check.level == BudgetLevel.HARD_LIMIT:
    raise RuntimeError(check.message)
elif check.level == BudgetLevel.SOFT_LIMIT:
    print(f"Warning: {check.message}")
```

##### `get_session_cost`

```python
def get_session_cost(self, session_id: str) -> float
```

Total cost for a session in USD.

##### `get_task_cost`

```python
def get_task_cost(self, task_id: str) -> float
```

Total cost for a task in USD.

##### `get_tenant_cost`

```python
def get_tenant_cost(self, tenant_id: str) -> float
```

Total cost for a tenant (all time).

##### `get_tenant_daily_cost`

```python
def get_tenant_daily_cost(self, tenant_id: str) -> float
```

Cost for a tenant in the current UTC day.

##### `get_session_summary`

```python
def get_session_summary(self, session_id: str) -> CostSummary
```

Get aggregated cost summary for a session.

##### `get_anomalies`

```python
def get_anomalies(self, session_id: Optional[str] = None) -> List[CostAnomaly]
```

Detect cost anomalies. A call is anomalous if its cost is 3x or more the rolling average for that model (minimum 5 calls required).

##### `clear_records`

```python
def clear_records(self, session_id: Optional[str] = None) -> int
```

Clear records. Returns count removed. If `session_id` is provided, only clears that session.

---

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `_ANOMALY_THRESHOLD` | `3.0` | Cost ratio threshold for anomaly detection |
| `_MIN_CALLS_FOR_ANOMALY` | `5` | Minimum calls before anomaly detection activates |
| `_ROLLING_WINDOW` | `50` | Rolling window size for cost history per model |

---

## See Also

- [LLM Router API](./router.md) - Integrates CostTracker for budget enforcement
- [Model Registry API](./registry.md) - Provides cost estimation per model
