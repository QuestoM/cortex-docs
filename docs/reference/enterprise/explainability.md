# Explainability Engine API Reference

## Module: `corteX.enterprise.explainability`

Decision explainability engine -- GDPR Arts 13-15 and 22 compliance. Explains WHY the agent made each decision by correlating `DecisionTracer` traces with weight engine state, goal tracking, and tool/model metadata. Produces both structured data and human-readable narratives.

Brain analogy: Introspective cortex for metacognitive self-monitoring.

## Enums

### `ExplanationLevel`

**Type**: `str, Enum`

Granularity of an explanation.

| Value | Description |
|-------|-------------|
| `summary` | One-liner for dashboards |
| `standard` | For non-technical stakeholders |
| `detailed` | For technical review / audit |
| `gdpr_full` | Full disclosure per GDPR Art. 15 |

---

### `DecisionCategory`

**Type**: `str, Enum`

What kind of decision was made.

| Value | Description |
|-------|-------------|
| `tool_selection` | A tool was chosen for execution |
| `model_routing` | An LLM model was selected |
| `plan_step` | A planning step was decided |
| `autonomy_routing` | Autonomy level was determined |
| `goal_verification` | Goal alignment was checked |
| `safety_check` | Safety policy was evaluated |

---

## Data Classes

### `DecisionExplanation`

**Type**: `@dataclass`

Full explanation of a single decision. GDPR Art. 15 compliant.

| Attribute | Type | Description |
|-----------|------|-------------|
| `session_id` | `str` | Session identifier |
| `step_index` | `int` | Position in the decision trace |
| `timestamp` | `float` | When the decision was made |
| `category` | `DecisionCategory` | What kind of decision |
| `decision` | `str` | What was decided |
| `confidence` | `float` | Confidence score (0.0 to 1.0) |
| `reasoning` | `str` | Machine-generated reasoning |
| `alternatives` | `List[AlternativeConsidered]` | Options that were evaluated but not chosen |
| `weight_influences` | `List[WeightInfluence]` | How weights influenced the outcome |
| `goal_alignment` | `Optional[GoalAlignmentInfo]` | Goal tracking context |
| `human_readable` | `str` | Human-readable narrative |
| `gdpr_disclosure` | `str` | GDPR Art. 15 disclosure text |

### `ToolSelectionExplanation`

**Type**: `@dataclass`

Detailed explanation of why a specific tool was selected.

| Attribute | Type | Description |
|-----------|------|-------------|
| `selected_tool` | `str` | Name of the selected tool |
| `confidence` | `float` | Selection confidence |
| `selection_method` | `str` | Algorithm used (e.g., `"thompson_sampling"`) |
| `preference_score` | `float` | Preference weight value |
| `bayesian_posterior_mean` | `float` | Bayesian posterior success rate |
| `success_rate` | `float` | Historical success rate |
| `avg_latency_ms` | `float` | Average execution time |
| `total_uses` | `int` | Total times this tool has been used |

### `RoutingExplanation`

**Type**: `@dataclass`

Detailed explanation of why a specific model was chosen.

| Attribute | Type | Description |
|-----------|------|-------------|
| `selected_model` | `str` | Name of the selected model |
| `task_type` | `str` | Task classification that drove selection |
| `confidence` | `float` | Routing confidence |
| `model_scores` | `Dict[str, float]` | Scores for all candidate models |
| `brain_state` | `Dict[str, float]` | Brain engine state at decision time |

### `SessionExplanationSummary`

**Type**: `@dataclass`

High-level summary of all decisions in a session.

| Attribute | Type | Description |
|-----------|------|-------------|
| `session_id` | `str` | Session identifier |
| `total_steps` | `int` | Total decisions made |
| `avg_confidence` | `float` | Average confidence across decisions |
| `categories_breakdown` | `Dict[str, int]` | Count per decision category |
| `goal_progress` | `float` | Goal completion progress (0.0-1.0) |
| `drift_score` | `float` | How far the session drifted from goal |
| `tools_used` | `List[str]` | Unique tools used |
| `models_used` | `List[str]` | Unique models used |
| `total_tokens` | `int` | Total tokens consumed |
| `total_latency_ms` | `float` | Total execution time |

---

## Classes

### `ExplainabilityEngine`

Generates GDPR-compliant explanations for agent decisions. Reads from a `DecisionTracer` and enriches traces with weight engine context, goal alignment, and human-readable narratives.

#### Constructor

```python
ExplainabilityEngine(
    tracer: Optional[DecisionTracer] = None,
    weight_snapshot: Optional[Dict[str, Any]] = None,
    goal_tracker_state: Optional[Dict[str, Any]] = None,
)
```

**Parameters**:

- `tracer` (`Optional[DecisionTracer]`): Decision tracer instance. Creates a default one if not provided.
- `weight_snapshot` (`Optional[Dict[str, Any]]`): Current weight engine state for correlating decisions with weights.
- `goal_tracker_state` (`Optional[Dict[str, Any]]`): Goal tracker state for alignment analysis.

#### Methods

##### `explain_decision`

```python
def explain_decision(
    self,
    session_id: str,
    step_index: int,
    level: ExplanationLevel = ExplanationLevel.STANDARD,
) -> DecisionExplanation
```

Explain a specific decision by step index. Returns a full `DecisionExplanation` with alternatives, weight influences, goal alignment, and human-readable narrative.

**Raises**: `IndexError` if `step_index` is out of range.

##### `get_decision_trace`

```python
def get_decision_trace(
    self, session_id: str
) -> List[DecisionStep]
```

Get the full ordered decision trace for a session. Each step includes category, confidence, reasoning, alternatives, and weight influences.

##### `explain_tool_selection`

```python
def explain_tool_selection(
    self, session_id: str, step_index: int,
) -> ToolSelectionExplanation
```

Explain why a specific tool was selected. Includes preference scores, Bayesian posterior, success rate, and latency.

**Raises**: `IndexError` if out of range, `ValueError` if step is not a tool selection.

##### `explain_model_routing`

```python
def explain_model_routing(
    self, session_id: str, step_index: int,
) -> RoutingExplanation
```

Explain why a specific model was chosen. Includes model scores, task type, and brain state.

**Raises**: `IndexError` if out of range, `ValueError` if step is not a model selection.

##### `get_session_summary`

```python
def get_session_summary(
    self, session_id: str
) -> SessionExplanationSummary
```

Build a high-level summary of all decisions in a session. Includes confidence averages, category breakdown, tools/models used, and goal progress.

##### `set_weight_snapshot`

```python
def set_weight_snapshot(self, snapshot: Dict[str, Any]) -> None
```

Update the weight snapshot used for explanations.

##### `set_goal_state`

```python
def set_goal_state(self, state: Dict[str, Any]) -> None
```

Update the goal tracker state used for explanations.

##### `generate_human_readable`

```python
def generate_human_readable(
    self,
    explanation: DecisionExplanation,
    level: ExplanationLevel = ExplanationLevel.STANDARD,
) -> str
```

Generate a human-readable narrative for a decision at the specified detail level.

---

## Example

```python
from corteX.enterprise.explainability import ExplainabilityEngine
from corteX.enterprise.explainability_types import ExplanationLevel
from corteX.observability.tracer import DecisionTracer

tracer = DecisionTracer()
engine = ExplainabilityEngine(
    tracer=tracer,
    weight_snapshot=weights.snapshot(),
    goal_tracker_state=goal_tracker.state(),
)

# Explain a specific decision (GDPR Art. 15)
explanation = engine.explain_decision("sess_abc", step_index=3)
print(explanation.human_readable)
# "Step 3: Selected tool 'web_search' with 87% confidence.
#  Primary factor: high preference score (0.82) based on
#  previous success rate of 94%. Goal alignment: 0.91."

print(explanation.gdpr_disclosure)
# Full GDPR Art. 15 disclosure text

# Full decision trace for a session
trace = engine.get_decision_trace("sess_abc")
for step in trace:
    print(f"  Step {step.step_index}: {step.category.value} -> {step.decision}")

# Explain tool selection in detail
tool_exp = engine.explain_tool_selection("sess_abc", step_index=2)
print(f"Selected: {tool_exp.selected_tool}")
print(f"Method: {tool_exp.selection_method}")
print(f"Success rate: {tool_exp.success_rate:.1%}")

# Explain model routing
routing = engine.explain_model_routing("sess_abc", step_index=0)
print(f"Model: {routing.selected_model} for task type: {routing.task_type}")

# Session summary
summary = engine.get_session_summary("sess_abc")
print(summary.human_readable)
print(f"Tools: {summary.tools_used}, Models: {summary.models_used}")
print(f"Goal progress: {summary.goal_progress:.0%}")
```

---

## See Also

- [Decision Tracer](../observability/tracer.md) -- Raw decision trace recording
- [Profiling Manager](./profiling.md) -- Art. 22 profiling opt-out
- [GDPR Manager](./gdpr.md) -- Full DSAR lifecycle
- [Compliance Engine](../security/compliance.md) -- Policy enforcement
