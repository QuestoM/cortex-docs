# Decision Tracer API Reference

## Module: `corteX.observability.tracer`

Records WHY the agent made each decision, not just what. Brain analogy: introspective cortex providing meta-cognitive transparency. Every model selection, tool choice, and plan step is traced with full context: alternatives considered, confidence scores, brain state, and reasoning.

## Classes

### `DecisionTrace`

**Type**: `@dataclass`

A single traced decision with full context.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `trace_id` | `str` | Auto-generated 16-char hex ID |
| `parent_id` | `Optional[str]` | Parent trace ID for nested tracing |
| `tenant_id` | `str` | Tenant identifier |
| `step_type` | `str` | One of `"model_selection"`, `"tool_selection"`, `"plan_step"` |
| `decision` | `str` | The decision made (e.g., model name, tool name, step description) |
| `alternatives` | `List[str]` | Alternatives that were considered |
| `reasoning` | `str` | Why this decision was made |
| `confidence` | `float` | Confidence score (0.0-1.0) |
| `latency_ms` | `float` | Time taken for this decision |
| `tokens_consumed` | `int` | Tokens used |
| `brain_state` | `Dict[str, float]` | Brain state snapshot at decision time |
| `timestamp` | `float` | Unix timestamp |
| `metadata` | `Dict[str, Any]` | Additional metadata |

#### Methods

- `to_dict() -> Dict[str, Any]` -- Serialize to flat dictionary
- `to_json() -> str` -- Serialize to JSON string

---

### `DecisionTracer`

Records agent decisions with full trace context in an in-memory ring buffer.

#### Constructor

```python
DecisionTracer(tenant_id: str = "", max_traces: int = 1000)
```

**Parameters**:

- `tenant_id` (`str`): Tenant identifier for this tracer.
- `max_traces` (`int`, default=1000): Ring buffer capacity (minimum 50).

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `tenant_id` | `str` | Current tenant ID |

#### Methods

##### `trace_model_selection`

```python
def trace_model_selection(
    self, routing_decision: Dict[str, Any],
    brain_snapshot: Dict[str, float],
    latency_ms: float = 0.0,
    tokens_consumed: int = 0
) -> DecisionTrace
```

Trace a model selection decision.

**Parameters**:

- `routing_decision` (`Dict[str, Any]`): Dict with `selected_model`, `alternatives`, `reasoning`, `confidence` keys.
- `brain_snapshot` (`Dict[str, float]`): Current brain state.
- `latency_ms` (`float`): Time taken for the model call.
- `tokens_consumed` (`int`): Tokens used.

**Returns**: `DecisionTrace` -- The recorded trace.

##### `trace_tool_selection`

```python
def trace_tool_selection(
    self, tool: str, confidence: float,
    alternatives: List[Tuple[str, float]],
    latency_ms: float = 0.0,
    reasoning: str = ""
) -> DecisionTrace
```

Trace a tool selection decision.

**Parameters**:

- `tool` (`str`): Selected tool name.
- `confidence` (`float`): Confidence score.
- `alternatives` (`List[Tuple[str, float]]`): `(tool_name, score)` tuples considered.
- `latency_ms` (`float`): Time for tool selection.
- `reasoning` (`str`): Why this tool was chosen.

##### `trace_plan_step`

```python
def trace_plan_step(
    self, step: str, risk: float, reasoning: str,
    latency_ms: float = 0.0, tokens_consumed: int = 0,
    brain_state: Optional[Dict[str, float]] = None
) -> DecisionTrace
```

Trace a plan step decision. Confidence is computed as `1.0 - risk`.

##### `push_parent` / `pop_parent`

```python
def push_parent(self, trace_id: str) -> None
def pop_parent(self) -> Optional[str]
```

Manage the parent stack for nested tracing. Child traces automatically link to the current parent.

##### `get_traces`

```python
def get_traces(self, last_n: int = 20) -> List[DecisionTrace]
```

Get the most recent traces.

##### `get_by_type`

```python
def get_by_type(self, step_type: str) -> List[DecisionTrace]
```

Get all traces of a specific type.

##### `get_by_parent`

```python
def get_by_parent(self, parent_id: str) -> List[DecisionTrace]
```

Get all child traces of a parent.

##### `export_json`

```python
def export_json(self) -> str
```

Export all traces as a JSON array.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: Dict with `total_traces`, `max_traces`, `traces_by_type`, `avg_confidence`, `avg_latency_ms`, `tenant_id`.

##### `clear`

```python
def clear(self) -> int
```

Clear all traces. Returns the count removed.

---

## Example

```python
from corteX.observability.tracer import DecisionTracer

tracer = DecisionTracer(tenant_id="acme", max_traces=1000)

# Trace model selection
trace = tracer.trace_model_selection(
    routing_decision={
        "selected_model": "gemini-3-pro-preview",
        "alternatives": ["gemini-3-flash-preview", "gpt-4"],
        "reasoning": "Complex task requires orchestrator-tier model",
        "confidence": 0.85,
    },
    brain_snapshot={"attention": 0.9, "fatigue": 0.1},
    latency_ms=150.5,
    tokens_consumed=2500,
)

# Nested tracing
tracer.push_parent(trace.trace_id)

tracer.trace_tool_selection(
    tool="code_interpreter",
    confidence=0.9,
    alternatives=[("search", 0.3), ("file_read", 0.5)],
    reasoning="Code generation task",
)

tracer.trace_plan_step(
    step="Generate User model",
    risk=0.2,
    reasoning="Standard CRUD model creation",
    tokens_consumed=500,
)

tracer.pop_parent()

# Query traces
model_traces = tracer.get_by_type("model_selection")
children = tracer.get_by_parent(trace.trace_id)

# Export
json_output = tracer.export_json()
stats = tracer.get_stats()
print(f"Total traces: {stats['total_traces']}")
print(f"Avg confidence: {stats['avg_confidence']:.2f}")
```

---

## See Also

- [Metrics Collector](./metrics.md) -- Quantitative metrics for dashboards
- [Cost Predictor](./cost-predictor.md) -- Cost estimation per decision
- [Audit Stream](./audit-stream.md) -- Tamper-evident regulatory audit log
