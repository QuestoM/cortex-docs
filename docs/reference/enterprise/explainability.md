# Explainability Engine API Reference

## Module: `corteX.enterprise.explainability`

Decision explainability engine for GDPR Arts 13-15, 22 compliance. Explains WHY the agent made each decision by correlating `DecisionTracer` traces with weight engine state, goal tracking, and tool/model metadata. Brain analogy: introspective cortex for metacognitive self-monitoring.

## Supporting Types

### `ExplanationLevel`

**Type**: `str, Enum`

Granularity of an explanation.

| Value | Description |
|-------|-------------|
| `SUMMARY` | One-liner for dashboards |
| `STANDARD` | For non-technical stakeholders |
| `DETAILED` | For technical review / audit |
| `GDPR_FULL` | Full disclosure per GDPR Art. 15 |

---

### `DecisionCategory`

**Type**: `str, Enum`

What kind of decision was made.

| Value | Description |
|-------|-------------|
| `TOOL_SELECTION` | A tool was chosen for execution |
| `MODEL_ROUTING` | An LLM model was chosen for a task |
| `PLAN_STEP` | A planning step was decided |
| `AUTONOMY_ROUTING` | Autonomy level routing decision |
| `GOAL_VERIFICATION` | Goal alignment verification |
| `SAFETY_CHECK` | Safety policy check |

---

### `AlternativeConsidered`

**Type**: `@dataclass`

A single alternative that was evaluated but not chosen.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | -- | Name of the alternative |
| `score` | `float` | -- | Score assigned to this alternative |
| `reason_not_chosen` | `str` | -- | Why this alternative was not selected |
| `metadata` | `Dict[str, Any]` | `{}` | Additional metadata |

---

### `WeightInfluence`

**Type**: `@dataclass`

How a specific weight influenced a decision.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `category` | `str` | -- | Weight category (e.g., `"tool_preference"`, `"behavioral"`) |
| `key` | `str` | -- | Specific weight key (e.g., `"code_interpreter"`, `"autonomy"`) |
| `value` | `float` | -- | Current weight value |
| `influence` | `float` | -- | How much this weight pushed toward the decision |
| `description` | `str` | `""` | Human-readable description of the influence |

---

### `GoalAlignmentInfo`

**Type**: `@dataclass`

Goal tracking context at decision time.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `original_goal` | `str` | -- | The original goal text |
| `progress` | `float` | -- | Goal progress `[0.0, 1.0]` |
| `drift_score` | `float` | -- | How far off-track (`0.0` = on track, `1.0` = fully drifted) |
| `alignment_score` | `float` | -- | How aligned this step is with the goal |
| `loop_detected` | `bool` | `False` | Whether a loop was detected |
| `stall_turns` | `int` | `0` | Number of turns the agent has stalled |

---

### `DecisionStep`

**Type**: `@dataclass`

A single decision within a session trace.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `step_index` | `int` | -- | Index of this step in the trace |
| `timestamp` | `float` | -- | Unix timestamp of the decision |
| `category` | `DecisionCategory` | -- | Type of decision |
| `decision` | `str` | -- | What was decided |
| `confidence` | `float` | -- | Confidence score `[0.0, 1.0]` |
| `reasoning` | `str` | -- | Machine-generated reasoning |
| `alternatives` | `List[AlternativeConsidered]` | `[]` | Alternatives that were considered |
| `weight_influences` | `List[WeightInfluence]` | `[]` | Weight factors that influenced the decision |
| `goal_alignment` | `Optional[GoalAlignmentInfo]` | `None` | Goal tracking context at decision time |
| `latency_ms` | `float` | `0.0` | Decision latency in milliseconds |
| `tokens_consumed` | `int` | `0` | Tokens consumed by this decision |
| `metadata` | `Dict[str, Any]` | `{}` | Additional metadata |

---

## Data Classes

### `DecisionExplanation`

**Type**: `@dataclass`

Full explanation of a single decision. GDPR Art. 15 compliant.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `session_id` | `str` | -- | Session identifier |
| `step_index` | `int` | -- | Step index in the trace |
| `timestamp` | `float` | -- | Unix timestamp of the decision |
| `category` | `DecisionCategory` | -- | Type of decision |
| `decision` | `str` | -- | What was decided |
| `confidence` | `float` | -- | Confidence score `[0.0, 1.0]` |
| `reasoning` | `str` | -- | Machine-generated reasoning chain |
| `alternatives` | `List[AlternativeConsidered]` | `[]` | Alternatives that were considered |
| `weight_influences` | `List[WeightInfluence]` | `[]` | Weight factors that influenced the decision |
| `goal_alignment` | `Optional[GoalAlignmentInfo]` | `None` | Goal tracking context at decision time |
| `latency_ms` | `float` | `0.0` | Decision latency in milliseconds |
| `tokens_consumed` | `int` | `0` | Tokens consumed by this decision |
| `human_readable` | `str` | `""` | Human-readable narrative (set by engine) |
| `gdpr_disclosure` | `str` | `""` | GDPR-formatted disclosure text |
| `metadata` | `Dict[str, Any]` | `{}` | Additional metadata from the trace |

#### Methods

##### `to_dict`

```python
def to_dict(self) -> Dict[str, Any]
```

Serialize to dictionary for JSON export and audit logging. Includes all fields, with nested serialization of `alternatives`, `weight_influences`, and `goal_alignment`.

---

### `ToolSelectionExplanation`

**Type**: `@dataclass`

Why a specific tool was selected.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `selected_tool` | `str` | -- | Name of the tool that was selected |
| `confidence` | `float` | -- | Confidence in the selection `[0.0, 1.0]` |
| `selection_method` | `str` | -- | Method used (e.g., `"thompson_sampling"`, `"preference_score"`) |
| `preference_score` | `float` | -- | Preference score from the weight engine |
| `bayesian_posterior_mean` | `float` | -- | Bayesian posterior mean (success rate estimate) |
| `alternatives` | `List[AlternativeConsidered]` | `[]` | Other tools that were considered |
| `weight_influences` | `List[WeightInfluence]` | `[]` | Weight factors that influenced tool selection |
| `success_rate` | `float` | `0.0` | Historical success rate of this tool |
| `avg_latency_ms` | `float` | `0.0` | Average execution latency in ms |
| `total_uses` | `int` | `0` | Total number of times this tool has been used |
| `reasoning` | `str` | `""` | Machine-generated reasoning for the selection |
| `human_readable` | `str` | `""` | Human-readable narrative (set by engine) |

---

### `RoutingExplanation`

**Type**: `@dataclass`

Why a specific model was chosen for a task.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `selected_model` | `str` | -- | Model ID that was selected |
| `task_type` | `str` | -- | Type of task (e.g., `"general"`, `"coding"`, `"analysis"`) |
| `confidence` | `float` | -- | Confidence in the routing `[0.0, 1.0]` |
| `model_scores` | `Dict[str, float]` | `{}` | Scores for all candidate models |
| `alternatives` | `List[AlternativeConsidered]` | `[]` | Other models that were considered |
| `weight_influences` | `List[WeightInfluence]` | `[]` | Weight factors that influenced model routing |
| `brain_state` | `Dict[str, float]` | `{}` | Brain state snapshot at decision time |
| `reasoning` | `str` | `""` | Machine-generated reasoning for routing |
| `human_readable` | `str` | `""` | Human-readable narrative (set by engine) |

---

### `SessionExplanationSummary`

**Type**: `@dataclass`

High-level summary of all decisions in a session.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `session_id` | `str` | -- | Session identifier |
| `total_steps` | `int` | -- | Total number of decision steps |
| `avg_confidence` | `float` | -- | Average confidence across all steps |
| `categories_breakdown` | `Dict[str, int]` | `{}` | Count of decisions per category |
| `goal_progress` | `float` | `0.0` | Overall goal progress `[0.0, 1.0]` |
| `drift_score` | `float` | `0.0` | Overall drift score |
| `tools_used` | `List[str]` | `[]` | Unique tools used during the session |
| `models_used` | `List[str]` | `[]` | Unique models used during the session |
| `total_tokens` | `int` | `0` | Total tokens consumed |
| `total_latency_ms` | `float` | `0.0` | Total latency in milliseconds |
| `human_readable` | `str` | `""` | Human-readable session summary (set by engine) |

---

## ExplainabilityEngine

Generates GDPR-compliant explanations for agent decisions. Reads from a `DecisionTracer` and enriches traces with weight engine context, goal alignment, and human-readable narratives.

### Constructor

```python
ExplainabilityEngine(
    tracer: Optional[DecisionTracer] = None,
    weight_snapshot: Optional[Dict[str, Any]] = None,
    goal_tracker_state: Optional[Dict[str, Any]] = None,
)
```

**Parameters**:

- `tracer` (`Optional[DecisionTracer]`): Decision tracer instance. Creates a new one if not provided.
- `weight_snapshot` (`Optional[Dict[str, Any]]`): Current weight engine state for enriching explanations.
- `goal_tracker_state` (`Optional[Dict[str, Any]]`): Current goal tracker state for alignment info.

### Methods

#### `set_weight_snapshot`

```python
def set_weight_snapshot(self, snapshot: Dict[str, Any]) -> None
```

Update the weight snapshot used for explanations. Call this when weights change.

#### `set_goal_state`

```python
def set_goal_state(self, state: Dict[str, Any]) -> None
```

Update the goal tracker state used for explanations.

#### `explain_decision`

```python
def explain_decision(
    self,
    session_id: str,
    step_index: int,
    level: ExplanationLevel = ExplanationLevel.STANDARD,
) -> DecisionExplanation
```

Explain a specific decision by step index. Returns a full `DecisionExplanation` with alternatives, weight influences, goal alignment, reasoning, human-readable narrative, and GDPR disclosure.

**Raises**: `IndexError` if `step_index` is out of range.

#### `get_decision_trace`

```python
def get_decision_trace(self, session_id: str) -> List[DecisionStep]
```

Get the full ordered decision trace for a session. Returns a list of `DecisionStep` objects with all enrichment (alternatives, weights, goal alignment, latency, tokens, metadata).

#### `explain_tool_selection`

```python
def explain_tool_selection(
    self, session_id: str, step_index: int,
) -> ToolSelectionExplanation
```

Explain why a specific tool was selected. Enriches the explanation with tool preference weights, Bayesian posterior, success rate, latency, and usage history.

**Raises**:

- `IndexError` if `step_index` is out of range.
- `ValueError` if the step at `step_index` is not a `tool_selection` step.

#### `explain_model_routing`

```python
def explain_model_routing(
    self, session_id: str, step_index: int,
) -> RoutingExplanation
```

Explain why a specific model was chosen for a task. Enriches the explanation with model scores, brain state, weight influences, and routing reasoning.

**Raises**:

- `IndexError` if `step_index` is out of range.
- `ValueError` if the step at `step_index` is not a `model_selection` step.

#### `get_session_summary`

```python
def get_session_summary(self, session_id: str) -> SessionExplanationSummary
```

Build a high-level summary of all decisions in a session. Aggregates confidence, categories, tools used, models used, total tokens, and total latency. Returns a summary with an empty message if no decisions have been recorded.

#### `generate_human_readable`

```python
def generate_human_readable(
    self,
    explanation: DecisionExplanation,
    level: ExplanationLevel = ExplanationLevel.STANDARD,
) -> str
```

Generate a human-readable narrative for a decision at the specified granularity level.

---

## Example

```python
from corteX.enterprise.explainability import ExplainabilityEngine
from corteX.enterprise.explainability_types import ExplanationLevel
from corteX.observability.tracer import DecisionTracer

tracer = DecisionTracer()
engine = ExplainabilityEngine(
    tracer=tracer,
    weight_snapshot={"tool_preference": {"code_interpreter": {"preference_score": 0.85}}},
    goal_tracker_state={"original_goal": "Build a REST API", "progress": 0.6},
)

# Explain a specific decision
explanation = engine.explain_decision("sess_abc", step_index=3)
print(explanation.human_readable)
print(explanation.reasoning)
print(f"Confidence: {explanation.confidence}")
print(f"Latency: {explanation.latency_ms}ms")
print(f"Tokens: {explanation.tokens_consumed}")
print(f"Alternatives: {len(explanation.alternatives)}")
for alt in explanation.alternatives:
    print(f"  - {alt.name}: {alt.score} ({alt.reason_not_chosen})")
for wi in explanation.weight_influences:
    print(f"  - {wi.category}/{wi.key}: {wi.value} (influence: {wi.influence})")

# Explain tool selection
tool_expl = engine.explain_tool_selection("sess_abc", step_index=5)
print(tool_expl.human_readable)
print(f"Selected: {tool_expl.selected_tool} via {tool_expl.selection_method}")
print(f"Success rate: {tool_expl.success_rate}")

# Explain model routing
route_expl = engine.explain_model_routing("sess_abc", step_index=2)
print(route_expl.human_readable)
print(f"Selected: {route_expl.selected_model} for {route_expl.task_type}")

# Session summary
summary = engine.get_session_summary("sess_abc")
print(summary.human_readable)
print(f"Steps: {summary.total_steps}, Avg confidence: {summary.avg_confidence:.2f}")
print(f"Tools: {summary.tools_used}, Models: {summary.models_used}")
print(f"Total tokens: {summary.total_tokens}")

# GDPR disclosure
explanation = engine.explain_decision("sess_abc", step_index=3, level=ExplanationLevel.GDPR_FULL)
print(explanation.gdpr_disclosure)
```

---

## See Also

- [Decision Tracer](../observability/tracer.md) -- Trace recording for decisions
- [Weight Engine](../engine/weights.md) -- Synaptic weight system
- [Goal Tracker](../engine/goal-tracker.md) -- Goal tracking and drift detection
- [Compliance Engine](../security/compliance.md) -- Compliance policy enforcement
- [Enterprise Config](./config.md) -- Tenant-level configuration
