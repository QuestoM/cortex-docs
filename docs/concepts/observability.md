# Observability

corteX provides built-in observability that answers not just "what happened" but "why it happened." The observability stack traces agent decisions with full context, predicts costs before execution, collects real-time metrics, and maintains tamper-evident audit logs for enterprise compliance.

## What It Does

The observability system provides four capabilities:

1. **DecisionTracer**: Records the reasoning behind every agent decision with full context
2. **CostPredictor**: Estimates total plan cost before execution with confidence intervals
3. **MetricsCollector**: Real-time metrics collection with Prometheus and OpenTelemetry export
4. **TenantAuditStream**: Tamper-evident, hash-chained audit logs for compliance

Together, these components give operators full visibility into agent behavior -- from individual decision rationale to aggregate cost trends to legally defensible audit trails.

---

## Decision Tracer

**Records WHY the agent made each decision, not just what it did.**

Traditional logging captures actions: "Called tool X." The DecisionTracer captures reasoning: "Called tool X because confidence was 0.92, the alternative tool Y had a 0.3 reputation score, and the brain state indicated System 1 routing."

### Why It Matters

When an agent makes a mistake, you need to understand the decision chain that led to it. The DecisionTracer provides full context for every branch point:

- Which model was selected and why
- What alternatives were considered and rejected
- What the brain state (confidence, drift, risk) was at decision time
- What the outcome was and how it affected future decisions

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="support",
    system_prompt="Help users with their accounts.",
    tracing=True  # Enable decision tracing
)

session = agent.start_session(user_id="user_1")
response = await session.run("Help me reset my password")

# Access the decision trace
traces = session.get_decision_traces()

for trace in traces:
    print(f"Decision: {trace.decision_type}")
    print(f"  Choice: {trace.chosen}")
    print(f"  Alternatives: {trace.alternatives}")
    print(f"  Confidence: {trace.confidence:.2%}")
    print(f"  Reasoning: {trace.reasoning}")

# Example output:
# Decision: model_selection
#   Choice: gemini-3-flash-preview
#   Alternatives: [gpt-4o-mini, claude-haiku-4-5]
#   Confidence: 94%
#   Reasoning: Low complexity task, System 1 routing, cost-optimized
#
# Decision: tool_selection
#   Choice: reset_password
#   Alternatives: [lookup_account, send_email]
#   Confidence: 87%
#   Reasoning: Direct match for user intent, tool reputation 0.95
```

### Trace Hierarchy

Decisions are linked in a parent-child hierarchy, enabling you to follow the complete decision chain:

```text
Session Start
  |-- model_selection (gemini-3-flash)
  |-- tool_selection (reset_password)
  |     |-- sub_decision: verify_identity
  |     |-- sub_decision: generate_reset_link
  |-- response_assembly
```

### Exporting Traces

Traces can be exported for analysis:

```python
# Export as JSON for custom analysis
trace_json = session.export_traces(format="json")

# Export as OpenTelemetry spans
otel_spans = session.export_traces(format="otel")
```

---

## Cost Predictor

**Estimate total plan cost BEFORE execution, not after.**

The CostPredictor analyzes a planned sequence of steps and estimates the total LLM cost with confidence intervals. If the predicted cost exceeds the session budget, the agent can request approval before proceeding.

### Why It Matters

LLM costs are unpredictable. A plan that looks simple might require dozens of tool calls, each consuming thousands of tokens. The CostPredictor prevents budget surprises by providing upfront estimates:

- Estimate cost per step based on task complexity
- Calculate total plan cost with upper and lower bounds
- Compare against budget before execution begins
- Track prediction accuracy to improve future estimates

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="developer",
    system_prompt="Help build software.",
    cost_prediction=True
)

session = agent.start_session(
    user_id="dev_1",
    budget=5.00  # $5 budget for this session
)

response = await session.run(
    "Build a REST API with authentication and database integration"
)

# Before executing, the agent predicts costs
prediction = session.get_cost_prediction()

print(f"Estimated cost: ${prediction.estimated_total:.2f}")
print(f"Confidence interval: ${prediction.lower_bound:.2f} -- ${prediction.upper_bound:.2f}")
print(f"Steps planned: {prediction.step_count}")
print(f"Budget remaining: ${prediction.budget_remaining:.2f}")
print(f"Within budget: {prediction.within_budget}")

# Example output:
# Estimated cost: $1.85
# Confidence interval: $1.20 -- $2.50
# Steps planned: 12
# Budget remaining: $3.15
# Within budget: True
```

### Prediction Accuracy

The predictor learns from actual costs to improve future estimates:

```python
# After execution completes
accuracy = session.get_prediction_accuracy()

print(f"Predicted: ${accuracy.predicted:.2f}")
print(f"Actual: ${accuracy.actual:.2f}")
print(f"Error: {accuracy.error_pct:.1%}")

# Example output:
# Predicted: $1.85
# Actual: $2.10
# Error: 13.5%
```

Over time, the predictor calibrates its estimates based on your specific usage patterns, model choices, and task types.

---

## Metrics Collector

**Real-time metrics collection with Prometheus and OpenTelemetry export.**

The MetricsCollector captures latency, token usage, success rates, drift scores, and costs using sliding-window storage. Metrics are available for dashboards, alerts, and capacity planning.

### Collected Metrics

| Metric | Type | Description |
|---|---|---|
| `cortex_request_latency_ms` | Histogram | End-to-end request latency |
| `cortex_tokens_used` | Counter | Input and output token consumption |
| `cortex_request_success` | Counter | Successful vs. failed requests |
| `cortex_drift_score` | Gauge | Current goal drift score |
| `cortex_cost_usd` | Counter | Accumulated LLM costs |
| `cortex_tool_calls` | Counter | Tool invocations by tool name |
| `cortex_model_selection` | Counter | Model selections by model ID |
| `cortex_brain_confidence` | Gauge | Current brain confidence level |

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="support",
    system_prompt="Help users.",
    metrics=True
)

session = agent.start_session(user_id="user_1")
response = await session.run("Help me")

# Access current metrics
metrics = session.get_metrics()
print(f"Avg latency: {metrics.avg_latency_ms:.0f}ms")
print(f"Total tokens: {metrics.total_tokens}")
print(f"Total cost: ${metrics.total_cost:.4f}")
print(f"Success rate: {metrics.success_rate:.1%}")
```

### Prometheus Export

Export metrics in Prometheus text format for integration with Grafana, Datadog, or any Prometheus-compatible monitoring system:

```python
# Export Prometheus-formatted metrics
prom_text = engine.metrics.export_prometheus()

# Example output:
# cortex_request_latency_ms_bucket{le="100"} 42
# cortex_request_latency_ms_bucket{le="500"} 87
# cortex_request_latency_ms_bucket{le="1000"} 95
# cortex_tokens_used_total{model="gemini-3-flash"} 125000
# cortex_cost_usd_total{tenant="acme_corp"} 0.0315
```

### OpenTelemetry Integration

Export spans in OpenTelemetry format for distributed tracing:

```python
# Export OTel spans
otel_data = engine.metrics.export_otel()

# Each agent turn produces a span with:
# - Trace ID linking all decisions in a session
# - Span ID for individual operations
# - Attributes: model, tokens, cost, drift_score
```

---

## Tenant Audit Stream

**Tamper-evident, hash-chained audit logs for enterprise compliance.**

The TenantAuditStream produces audit entries where each entry is cryptographically chained to its predecessor using SHA-256. This creates an immutable, tamper-evident log suitable for regulatory audits, legal proceedings, and security investigations.

### Why It Matters

Enterprise customers require proof that their AI agents operated within policy. Traditional logs can be edited or deleted. Hash-chained audit streams provide:

- **Tamper evidence**: Any modification to a past entry breaks the chain
- **Completeness proof**: Gaps in the chain are immediately detectable
- **Tenant isolation**: Each tenant has an independent audit chain
- **Compliance readiness**: Meets SOC2 CC7.2, GDPR Article 30, and HIPAA audit requirements

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="hr-assistant",
    system_prompt="Help HR team.",
    audit=True
)

session = agent.start_session(
    user_id="hr_1",
    tenant_id="acme_corp"
)

response = await session.run("Show employee directory")

# Access audit entries
audit_entries = session.get_audit_trail()

for entry in audit_entries:
    print(f"Event: {entry.event_type}")
    print(f"  Timestamp: {entry.timestamp}")
    print(f"  Tenant: {entry.tenant_id}")
    print(f"  User: {entry.user_id}")
    print(f"  Chain hash: {entry.chain_hash[:16]}...")
    print(f"  Previous hash: {entry.prev_hash[:16]}...")
```

### Audit Event Types

| Event Type | When It Fires | What It Records |
|---|---|---|
| **session_start** | Session created | User ID, tenant ID, agent config |
| **model_selection** | LLM model chosen | Model ID, alternatives, reasoning |
| **tool_execution** | Tool called | Tool name, arguments (sanitized), result summary |
| **data_classification** | PII detected | Classification level, data type (not the data itself) |
| **compliance_check** | Policy evaluated | Framework, policy, decision (proceed/block) |
| **budget_event** | Budget threshold crossed | Current spend, limit, action taken |
| **session_end** | Session completed | Final metrics, total cost, success status |

### Verifying Chain Integrity

Verify that the audit chain has not been tampered with:

```python
# Verify the complete audit chain
verification = engine.audit.verify_chain(tenant_id="acme_corp")

print(f"Chain length: {verification.entry_count}")
print(f"Integrity: {'VALID' if verification.valid else 'TAMPERED'}")
print(f"Time span: {verification.first_entry} to {verification.last_entry}")

if not verification.valid:
    print(f"Break detected at entry: {verification.break_position}")
```

---

## Integration

All observability modules work together automatically:

```python
import cortex

engine = cortex.Engine(
    observability={
        "tracing": True,
        "cost_prediction": True,
        "metrics": True,
        "audit": True,
        "prometheus_port": 9090,  # Optional: expose metrics endpoint
    }
)

# Every session now includes:
# 1. Full decision traces with reasoning
# 2. Pre-execution cost predictions
# 3. Real-time metrics collection
# 4. Hash-chained audit entries
```

---

## See Also

- [Monitor Your Agent](../guides/advanced/observability.md) -- Practical guide to setting up monitoring
- [Audit Logging](../enterprise/audit.md) -- Enterprise audit configuration
- [Security Framework](security.md) -- Security modules that generate audit events
- [Compliance](../enterprise/compliance.md) -- Compliance framework configuration
