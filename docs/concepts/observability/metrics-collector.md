# Metrics Collector

The Metrics Collector captures real-time runtime metrics for
monitoring dashboards, acting like the autonomic nervous system
reporting vital signs continuously. It tracks latency, token usage,
success rates, goal drift, and costs with memory-bounded sliding
window storage.

## How It Works

The `MetricsCollector` maintains five independent sliding-window
deques, each storing `MetricEntry` data points:

| Channel | Method | Unit | Purpose |
|---------|--------|------|---------|
| Latencies | `record_latency()` | ms | Request response times |
| Tokens | `record_tokens()` | count | Token consumption per model |
| Successes | `record_success()` | bool | Success/failure tracking |
| Drifts | `record_drift()` | score | Goal drift monitoring (0.0-1.0) |
| Costs | `record_cost()` | USD | Dollar cost per model call |

Each deque has a configurable maximum size (default 10,000 entries).
When the window fills, the oldest entries are automatically evicted.
All entries include a timestamp, tenant ID, and operation name for
filtering.

The `get_dashboard()` method aggregates all channels into a
`DashboardData` object with computed averages, error rates, totals,
and top-N rankings - ready for display in a monitoring UI.

## Key Features

- **Five metric channels** - latency, tokens, success rate, drift, and cost
- **Sliding window storage** - bounded memory via `deque(maxlen=N)`
- **Per-tenant filtering** - `get_dashboard(tenant_id)` isolates one tenant's metrics
- **Process-wide view** - `get_dashboard(None)` aggregates across all tenants
- **Prometheus export** - `export_prometheus()` outputs text exposition format with tenant labels
- **OpenTelemetry export** - `export_opentelemetry()` produces OTel-compatible resource spans
- **Dashboard aggregation** - top operations, top models, average latency, error rate, drift average
- **Period tracking** - `DashboardData.period_seconds` shows the time span of collected data

## Integration

The Metrics Collector is called throughout the corteX runtime.
The LLM providers record latency, tokens, and cost after each model
call. The orchestrator records success/failure outcomes. The Goal
Tracker feeds drift scores. All metrics flow into a single collector
instance per process.

The Prometheus export endpoint can be scraped by standard monitoring
infrastructure (Prometheus, Grafana). The OpenTelemetry export
integrates with distributed tracing systems (Jaeger, Zipkin). The
dashboard data powers the optional FastAPI server's monitoring UI.

## Usage Example

```python
from corteX.observability.metrics import MetricsCollector

collector = MetricsCollector(window_size=5000)

# Record metrics during agent execution
collector.record_latency("acme", "llm_call", 245.5)
collector.record_tokens("acme", "gemini-2.5-pro", 1850)
collector.record_success("acme", "llm_call", True)
collector.record_drift("acme", 0.12)
collector.record_cost("acme", "gemini-2.5-pro", 0.0023)

# Get aggregated dashboard for one tenant
dashboard = collector.get_dashboard(tenant_id="acme")
print(f"Requests: {dashboard.total_requests}")
print(f"Avg latency: {dashboard.avg_latency_ms}ms")
print(f"Error rate: {dashboard.error_rate}")
print(f"Total tokens: {dashboard.tokens_total}")
print(f"Total cost: ${dashboard.cost_total_usd}")
print(f"Drift avg: {dashboard.drift_avg}")

# Export for Prometheus scraping
prom_text = collector.export_prometheus()
# cortex_latency_ms{tenant_id="acme"} 245.50
# cortex_tokens_total{tenant_id="acme"} 1850
# cortex_error_rate{tenant_id="acme"} 0.0000
# cortex_cost_usd{tenant_id="acme"} 0.002300

# Export for OpenTelemetry
otel_data = collector.export_opentelemetry()
# {"resourceSpans": [{"resource": {...}, "scopeSpans": [...]}]}

# Collector stats
stats = collector.get_stats()
# {"window_size": 5000, "latency_entries": 1, ...}
```

## See Also

- [Decision Tracer](decision-tracer.md) - explains why decisions were made
- [Cost Predictor](cost-predictor.md) - predicts costs before execution
- [Tenant Audit Stream](audit-stream.md) - tamper-evident audit log
