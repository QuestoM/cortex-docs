# Tenant DNA API Reference

## Module: `corteX.tenancy.dna`

Learned usage profile per tenant. A compact representation of how each tenant uses corteX. Persists across sessions, enabling smarter model routing, pre-warming, and anomaly detection from the very first turn of a new session. Updated via exponential moving average (EMA) after each session.

## Classes

### `TenantDNA`

**Type**: `@dataclass`

Compact learned profile for a tenant.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `avg_task_complexity` | `float` | `0.5` | 0 = simple, 1 = complex |
| `preferred_model_tier` | `str` | `"worker"` | `"orchestrator"` or `"worker"` |
| `avg_tokens_per_task` | `int` | `2000` | Running EMA of tokens used per task |
| `tool_usage_frequency` | `Dict[str, float]` | `{}` | Tool name to usage frequency (0-1) |
| `peak_hours` | `List[int]` | `[]` | UTC hours when the tenant is most active |
| `risk_profile` | `float` | `0.3` | 0 = conservative, 1 = aggressive |
| `common_intents` | `List[str]` | `[]` | Most frequent intent categories |
| `last_updated` | `float` | `time.time()` | Unix timestamp of last update |

#### Methods

##### `update_from_session`

```python
def update_from_session(
    self, session_metrics: Dict[str, Any],
    alpha: float = 0.2
) -> None
```

Update DNA using exponential moving average from session metrics.

**Parameters**:

- `session_metrics` (`Dict[str, Any]`): Session metrics dict with optional keys:
    - `task_complexity` (`float`, 0-1)
    - `model_tier` (`str`)
    - `tokens_used` (`int`)
    - `tools_used` (`Dict[str, int]` -- tool name to call count)
    - `hour_utc` (`int`, 0-23)
    - `risk_score` (`float`, 0-1)
    - `intent` (`str`)
- `alpha` (`float`, default=0.2): EMA smoothing factor. Smaller = slower adaptation. Clamped to [0.01, 1.0].

**Update rules**:

| Metric | Method |
|--------|--------|
| Task complexity | EMA |
| Model tier | Majority wins (last value) |
| Tokens per task | EMA |
| Tool usage frequency | EMA per tool, pruned at 50 |
| Peak hours | Append (unique), keep last 12 |
| Risk profile | EMA |
| Common intents | Append (dedup), keep last 20 |

##### `to_dict` / `from_dict`

```python
def to_dict(self) -> Dict[str, Any]

@classmethod
def from_dict(cls, data: Dict[str, Any]) -> TenantDNA
```

Serialize/deserialize the DNA profile.

---

### `TenantDNAStore`

In-memory store for `TenantDNA` objects, keyed by tenant ID.

#### Constructor

```python
TenantDNAStore()
```

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `count` | `int` | Number of stored DNA profiles |

#### Methods

##### `get`

```python
def get(self, tenant_id: str) -> TenantDNA
```

Retrieve the DNA for a tenant. Returns a default `TenantDNA` if none exists.

##### `save`

```python
def save(self, tenant_id: str, dna: TenantDNA) -> None
```

Persist (in-memory) the DNA for a tenant.

##### `delete`

```python
def delete(self, tenant_id: str) -> None
```

Delete the DNA for a tenant (GDPR erasure).

##### `list_tenants`

```python
def list_tenants(self) -> List[str]
```

Return all tenant IDs that have stored DNA.

---

## EMA Formula

```
new_value = old_value * (1 - alpha) + observed_value * alpha
```

With `alpha = 0.2` (default), the profile adapts moderately:

| Sessions | Weight of Latest Session |
|----------|------------------------|
| 1 | 20% |
| 5 | ~67% cumulative influence |
| 10 | ~89% cumulative influence |

---

## Example

```python
from corteX.tenancy.dna import TenantDNA, TenantDNAStore

store = TenantDNAStore()

# Get or create DNA
dna = store.get("acme")  # Returns default if missing

# Update from a session
dna.update_from_session({
    "task_complexity": 0.8,
    "model_tier": "orchestrator",
    "tokens_used": 5000,
    "tools_used": {"code_interpreter": 5, "search": 2},
    "hour_utc": 14,
    "risk_score": 0.4,
    "intent": "code_generation",
})

# Save back
store.save("acme", dna)

# Use for model routing
print(f"Complexity: {dna.avg_task_complexity:.2f}")
print(f"Preferred tier: {dna.preferred_model_tier}")
print(f"Avg tokens: {dna.avg_tokens_per_task}")
print(f"Risk: {dna.risk_profile:.2f}")

# Serialize for persistence
data = dna.to_dict()
restored = TenantDNA.from_dict(data)

# GDPR deletion
store.delete("acme")
```

---

## Use Cases

| Use Case | DNA Field Used |
|----------|---------------|
| Model routing | `preferred_model_tier`, `avg_task_complexity` |
| Cost prediction | `avg_tokens_per_task` |
| Tool pre-warming | `tool_usage_frequency` |
| Anomaly detection | `risk_profile`, `peak_hours` |
| Capacity planning | `peak_hours`, `avg_tokens_per_task` |

---

## See Also

- [Tenant Manager](./manager.md) -- Tenant lifecycle management
- [Quota Tracker](./quota.md) -- Per-tenant resource limits
- [Cost Predictor](../observability/cost-predictor.md) -- Uses DNA for cost estimation
