# Decision Tracer

The Decision Tracer records WHY the agent made each decision, not just
what it did. Every model selection, tool choice, and plan step is
traced with full context including alternatives considered, confidence
scores, brain state, and reasoning. This provides complete
explainability for agent behavior.

## How It Works

The `DecisionTracer` maintains an in-memory ring buffer of
`DecisionTrace` objects, each capturing a single decision point.
Three specialized methods record different decision types:

- **`trace_model_selection()`** - records which model was chosen,
  what alternatives were considered, the routing reasoning, and the
  brain state snapshot at decision time
- **`trace_tool_selection()`** - records which tool was selected,
  with scored alternatives (e.g., `"search(0.85)"`, `"browse(0.42)"`)
- **`trace_plan_step()`** - records individual plan steps with risk
  scores and reasoning (confidence is computed as `1.0 - risk`)

Each trace includes a `parent_id` for hierarchical tracing. The
tracer maintains a parent stack so nested decisions automatically
link to their parent, enabling tree-structured trace visualization.

## Key Features

- **Three decision types** - model selection, tool selection, and plan steps
- **Full context capture** - alternatives, confidence, latency, tokens, brain state
- **Hierarchical tracing** - parent stack enables nested decision trees
- **Ring buffer storage** - configurable max size (default 1,000) with FIFO eviction
- **Per-tenant isolation** - each tracer is scoped to a tenant ID
- **JSON export** - `export_json()` dumps all traces as a formatted JSON array
- **Statistics** - `get_stats()` returns trace counts by type, average confidence, and average latency
- **Query by type** - `get_by_type()` filters traces to a specific decision category
- **Query by parent** - `get_by_parent()` retrieves all children of a decision

## Integration

The Decision Tracer is called by the orchestrator and brain engine at
every decision point during agent execution. The model router calls
`trace_model_selection()` after choosing a provider. The tool
framework calls `trace_tool_selection()` when picking a tool. The
planner calls `trace_plan_step()` for each step in a plan.

The Metrics Collector operates alongside the tracer - while the tracer
captures the "why" behind decisions, the metrics collector captures
the "how much" (latency, tokens, cost). Together they provide
complete observability.

## Usage Example

```python
from corteX.observability.tracer import DecisionTracer

tracer = DecisionTracer(tenant_id="acme-corp", max_traces=500)

# Trace a model selection decision
trace = tracer.trace_model_selection(
    routing_decision={
        "selected_model": "gemini-2.5-pro",
        "alternatives": ["gpt-4", "local-llama"],
        "reasoning": "Task requires strong reasoning; Gemini scored highest",
        "confidence": 0.92,
    },
    brain_snapshot={"attention": 0.85, "calibration": 0.78},
    latency_ms=1250.0,
    tokens_consumed=3400,
)

# Use parent stack for nested tracing
tracer.push_parent(trace.trace_id)

# Trace a tool selection within this model call
tracer.trace_tool_selection(
    tool="search_db",
    confidence=0.88,
    alternatives=[("search_db", 0.88), ("browse_web", 0.42)],
    reasoning="Database search matches the structured query pattern",
)

tracer.pop_parent()

# Query traces
recent = tracer.get_traces(last_n=10)
model_traces = tracer.get_by_type("model_selection")
stats = tracer.get_stats()
# stats = {"total_traces": 2, "avg_confidence": 0.9, ...}
```

## See Also

- [Metrics Collector](metrics-collector.md) - quantitative runtime metrics
- [Cost Predictor](cost-predictor.md) - pre-execution cost estimation
- [Tenant Audit Stream](audit-stream.md) - tamper-evident compliance audit log
