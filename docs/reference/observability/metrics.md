# Metrics Collector API Reference

## Module: `corteX.observability.metrics`

Real-time metrics for monitoring dashboards. Brain analogy: autonomic nervous system reporting vital signs continuously. Collects latency, token usage, success rates, drift scores, and costs with sliding window storage. Exports Prometheus text format and OpenTelemetry spans.

## Classes

### `MetricEntry`

**Type**: `@dataclass`

A single metric data point.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `timestamp` | `float` | Unix timestamp |
| `tenant_id` | `str` | Tenant identifier |
| `operation` | `str` | Operation or model name |
| `value` | `float` | Metric value |
| `unit` | `str` | Unit of measurement (`"ms"`, `"tokens"`, `"bool"`, `"score"`, `"usd"`) |
| `metadata` | `Dict[str, Any]` | Additional metadata |

---

### `DashboardData`

**Type**: `@dataclass`

Aggregated metrics for a monitoring dashboard.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `total_requests` | `int` | Total request count |
| `avg_latency_ms` | `float` | Average latency in milliseconds |
| `error_rate` | `float` | Error rate (0.0-1.0) |
| `tokens_total` | `int` | Total tokens consumed |
| `cost_total_usd` | `float` | Total cost in USD |
| `top_operations` | `List[Tuple[str, int]]` | Top 10 operations by count |
| `top_models` | `List[Tuple[str, int]]` | Top 10 models by token usage |
| `drift_avg` | `float` | Average goal drift score |
| `period_seconds` | `float` | Time span of the data |

#### Methods

- `to_dict() -> Dict[str, Any]`
- `to_json() -> str`

---

### `MetricsCollector`

Collects and exposes real-time metrics for monitoring systems. Uses sliding window deques for memory-bounded storage.

#### Constructor

```python
MetricsCollector(window_size: int = 10_000)
```

**Parameters**:

- `window_size` (`int`, default=10,000): Maximum entries per metric type (minimum 100).

#### Methods

##### Recording Methods

```python
def record_latency(self, tenant_id: str, operation: str, ms: float) -> None
def record_tokens(self, tenant_id: str, model: str, count: int) -> None
def record_success(self, tenant_id: str, operation: str, success: bool) -> None
def record_drift(self, tenant_id: str, score: float) -> None
def record_cost(self, tenant_id: str, model: str, cost_usd: float) -> None
```

Record individual metric data points. All methods append to bounded deques.

##### `get_dashboard`

```python
def get_dashboard(self, tenant_id: Optional[str] = None) -> DashboardData
```

Build aggregated dashboard data. Pass `None` for process-wide metrics, or a tenant ID for per-tenant filtering.

**Returns**: `DashboardData` with aggregated metrics.

##### `export_prometheus`

```python
def export_prometheus(self) -> str
```

Export metrics in Prometheus text exposition format.

**Exported metrics**:

| Metric | Type | Description |
|--------|------|-------------|
| `cortex_latency_ms` | gauge | Average request latency per tenant |
| `cortex_tokens_total` | counter | Total tokens consumed per tenant |
| `cortex_error_rate` | gauge | Error rate per tenant (0-1) |
| `cortex_cost_usd` | counter | Total cost in USD per tenant |

**Example output**:

```
# HELP cortex_latency_ms Request latency in ms
# TYPE cortex_latency_ms gauge
cortex_latency_ms{tenant_id="acme"} 125.50
# HELP cortex_tokens_total Total tokens consumed
# TYPE cortex_tokens_total counter
cortex_tokens_total{tenant_id="acme"} 45000
# HELP cortex_error_rate Error rate (0-1)
# TYPE cortex_error_rate gauge
cortex_error_rate{tenant_id="acme"} 0.0500
# HELP cortex_cost_usd Total cost in USD
# TYPE cortex_cost_usd counter
cortex_cost_usd{tenant_id="acme"} 0.125000
```

##### `export_opentelemetry`

```python
def export_opentelemetry(self) -> Dict[str, Any]
```

Export metrics as OpenTelemetry-compatible resource spans.

**Returns**: `Dict[str, Any]` with `resourceSpans` structure containing:

- `service.name`: `"cortex-agent"`
- `service.version`: `"1.0.0"`
- Spans with `operationName`, `startTime`, `duration`, `tags`, `attributes`

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: Dict with `window_size`, `latency_entries`, `token_entries`, `success_entries`, `drift_entries`, `cost_entries`.

##### `clear`

```python
def clear(self) -> None
```

Clear all collected metrics.

---

## Example

```python
from corteX.observability.metrics import MetricsCollector

collector = MetricsCollector(window_size=10_000)

# Record metrics
collector.record_latency("acme", "model_call", 125.5)
collector.record_tokens("acme", "gemini-3-pro-preview", 2500)
collector.record_success("acme", "model_call", True)
collector.record_drift("acme", 0.15)
collector.record_cost("acme", "gemini-3-pro-preview", 0.005)

# Get dashboard
dashboard = collector.get_dashboard(tenant_id="acme")
print(f"Avg latency: {dashboard.avg_latency_ms}ms")
print(f"Error rate: {dashboard.error_rate:.1%}")
print(f"Total tokens: {dashboard.tokens_total:,}")
print(f"Total cost: ${dashboard.cost_total_usd:.4f}")

# Process-wide dashboard
global_dashboard = collector.get_dashboard()

# Export for Prometheus
prom_text = collector.export_prometheus()

# Export for OpenTelemetry
otel_data = collector.export_opentelemetry()

# Expose via FastAPI
# @app.get("/metrics")
# async def metrics():
#     return Response(collector.export_prometheus(), media_type="text/plain")
```

---

## Memory Management

Each metric type uses a bounded `deque` with `maxlen=window_size`. When the deque is full, the oldest entries are automatically evicted. Default capacity is 10,000 entries per metric type (5 types = 50,000 entries maximum).

---

## See Also

- [Decision Tracer](./tracer.md) -- Qualitative decision tracing
- [Cost Predictor](./cost-predictor.md) -- Predicted vs actual cost comparison
- [Audit Stream](./audit-stream.md) -- Regulatory-grade audit logging
- [Quota Tracker](../tenancy/quota.md) -- Real-time quota enforcement
