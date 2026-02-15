# Weight Engine

`corteX.engine.weights`

The synaptic weight system that drives adaptive behavior across all corteX components. Inspired by Hebbian learning, Long-Term Potentiation/Depression, homeostatic regulation, prediction error signals, Bayesian conjugate priors, Kahneman-Tversky prospect theory, and Thompson Sampling.

Seven weight categories operate at different timescales and scopes, from per-turn behavioral adjustments to cross-deployment global learning.

---

## WeightCategory

Enum defining the seven weight tiers.

```python
class WeightCategory(str, Enum):
    BEHAVIORAL = "behavioral"
    TOOL_PREFERENCE = "tool_preference"
    MODEL_SELECTION = "model_selection"
    GOAL_ALIGNMENT = "goal_alignment"
    USER_INSIGHTS = "user_insights"
    ENTERPRISE = "enterprise"
    GLOBAL = "global"
```

---

## WeightUpdate

A single weight change event, analogous to a synaptic signal.

| Attribute | Type | Description |
|-----------|------|-------------|
| `category` | `WeightCategory` | Which weight tier to update |
| `key` | `str` | The specific weight key within the category |
| `delta` | `float` | Magnitude and direction of the change |
| `reason` | `str` | Human-readable explanation for the update |
| `timestamp` | `float` | Unix timestamp (auto-populated) |
| `source` | `str` | Which component triggered this update |

---

## LearningRates

Controls how fast each weight category adapts. Like synaptic plasticity thresholds.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `behavioral` | `float` | `0.12` | Fast adaptation for per-turn signals |
| `tool_preference` | `float` | `0.08` | Moderate adaptation for tool outcomes |
| `model_selection` | `float` | `0.04` | Slow adaptation for model routing |
| `goal_alignment` | `float` | `0.15` | Fast tracking for goal drift |
| `user_insights` | `float` | `0.03` | Slow cross-session learning |
| `enterprise` | `float` | `0.0` | Enterprise weights are set, not learned |
| `global_` | `float` | `0.01` | Very slow cross-deployment learning |

### Methods

#### `get(category: WeightCategory) -> float`

Returns the learning rate for a given weight category.

---

## BehavioralWeights

Tier 1 weights updated every turn based on immediate signals. Analogy: fast synaptic transmission.

Default weight keys: `verbosity`, `formality`, `initiative`, `detail_level`, `creativity`, `speed_vs_quality`, `autonomy`, `explanation_depth`, `code_density`, `risk_tolerance`. All range from `-1.0` to `1.0`.

### Methods

#### `get(key: str, default: float = 0.0) -> float`

Returns the current value for a behavioral weight.

#### `update(key: str, delta: float, lr: float = 0.12) -> float`

Applies an update with momentum and homeostatic clamping. Returns the actual delta applied. The momentum factor (`0.7`) smooths rapid changes, and a homeostatic pull (`-0.01 * value`) gently resists extreme values.

#### `to_dict() -> Dict[str, float]`

Returns a snapshot of all behavioral weights.

---

## ToolPreferenceWeights

Tier 2 weights tracking which tools work best for which tasks. Updated after each tool execution. Enhanced with Bayesian posteriors, prospect-theoretic updates, and Thompson Sampling.

### Constructor

```python
ToolPreferenceWeights(anchor_manager: Optional[AnchorManager] = None)
```

### Methods

#### `register_tool(name: str) -> None`

Registers a new tool with default metrics and an informed Beta prior via the AnchorManager.

#### `record_use(name: str, success: bool, latency_ms: float, lr: float = 0.08) -> None`

Records a tool execution result. Performs both EMA (fast heuristic) and Bayesian posterior updates. Applies LTP bonuses for consecutive successes and LTD penalties for repeated failures with prospect-theoretic loss aversion.

#### `get_preference(name: str) -> float`

Returns the EMA-based preference score (`0.0` to `1.0`) for a tool.

#### `get_bayesian_preference(name: str) -> float`

Returns the Bayesian posterior mean for a tool's success rate.

#### `get_best_tool(candidates: List[str]) -> Optional[str]`

Deterministic selection: returns the candidate with the highest preference score.

#### `get_best_tool_thompson(candidates: List[str]) -> Optional[str]`

Thompson Sampling selection: draws from each candidate's Beta posterior, picks the highest sample. Automatically balances exploration and exploitation.

#### `get_best_tool_with_latency(candidates: List[str], speed_weight: float = 0.3) -> Optional[str]`

Thompson Sampling that also considers latency via a speed-vs-quality tradeoff parameter.

#### `get_tool_surprise(name: str) -> float`

Returns `1.0` if the tool's recent performance is anomalous (differs significantly from baseline), `0.0` otherwise.

#### `get_posterior_summary(name: str) -> Dict[str, Any]`

Returns full Bayesian posterior summary including mean, std, 95% CI, and observation count.

#### `decay_posteriors(factor: float = 0.99) -> None`

Applies temporal decay to all posteriors for non-stationary environments.

#### `to_dict() -> Dict[str, Dict[str, float]]`

Returns EMA-based tool metrics.

#### `to_full_dict() -> Dict[str, Any]`

Returns full serialization including Bayesian state and anchors.

---

## ModelSelectionWeights

Tier 3 weights mapping task types to optimal LLM models. Default task types: `planning`, `coding`, `summarization`, `validation`, `conversation`, `tool_use`, `reasoning`.

### Methods

#### `set_initial(task_type: str, model_id: str, score: float) -> None`

Sets the initial score for a model on a task type.

#### `update(task_type: str, model_id: str, delta: float, lr: float = 0.04) -> None`

Updates the score for a model-task combination.

#### `get_scores(task_type: str) -> Dict[str, float]`

Returns all model scores for a given task type.

---

## UserInsightWeights

Tier 2 feedback weights learned across sessions. Tracks preferences like response length, domain expertise, frustration, engagement, and topic affinities.

### Methods

#### `update(key: str, value: Any) -> None`

Sets a user insight value directly.

#### `get(key: str, default: Any = None) -> Any`

Retrieves a user insight value.

#### `update_topic_preference(topic: str, delta: float) -> None`

Adjusts the affinity score for a specific topic.

---

## EnterpriseWeights

Tier 3 enterprise-level configuration set by admins. Not learned -- directly configured. Tracks safety, data sensitivity, autonomy caps, allowed tools, compliance rules, and brand voice.

### Methods

#### `set(key: str, value: Any) -> None`

Admin sets an enterprise weight.

#### `user_override(key: str, value: Any) -> bool`

Attempts a user override. Returns `True` if the key is in the overridable set (`safety_strictness`, `max_autonomy_level`, `feedback_collection`).

#### `get(key: str, default: Any = None) -> Any`

Retrieves an enterprise weight value.

---

## GlobalWeights

Tier 4 aggregated learning across all corteX deployments (opt-in only). Tracks common failure patterns, optimal model routing, tool reliability, and best-practice templates.

### Methods

#### `merge_from_cloud(cloud_data: Dict[str, Any]) -> None`

Merges weights received from the corteX cloud. No-op if `enabled` is `False`.

---

## WeightEngine

The central nervous system of adaptive behavior. Manages all 7 weight categories and coordinates updates. Enhanced with Bayesian posteriors, prospect-theoretic updates, Thompson Sampling, anchor management, availability filtering, and frame normalization.

### Constructor

```python
WeightEngine(
    learning_rates: Optional[LearningRates] = None,
    anchor_manager: Optional[AnchorManager] = None,
)
```

| Attribute | Type | Description |
|-----------|------|-------------|
| `behavioral` | `BehavioralWeights` | Tier 1 per-turn weights |
| `tools` | `ToolPreferenceWeights` | Tier 2 tool selection weights |
| `models` | `ModelSelectionWeights` | Tier 3 model routing weights |
| `user_insights` | `UserInsightWeights` | Cross-session user preferences |
| `enterprise` | `EnterpriseWeights` | Admin-configured enterprise rules |
| `global_` | `GlobalWeights` | Cross-deployment aggregated weights |
| `goal_alignment` | `Dict[str, float]` | Goal tracking weights |

### Methods

#### `apply_update(update: WeightUpdate) -> float`

Applies a single weight update and records it in history. Returns the actual delta applied.

#### `batch_update(updates: List[WeightUpdate]) -> Dict[str, float]`

Applies multiple updates atomically. Returns a dict of `"category.key" -> actual_delta`.

#### `snapshot() -> Dict[str, Any]`

Returns a full serializable snapshot of all weight categories.

#### `save(path: str) -> None`

Persists all weights to a JSON file.

#### `load(path: str) -> None`

Restores weights from a JSON file.

#### `get_recent_updates(n: int = 10) -> List[WeightUpdate]`

Returns the N most recent weight update events.

#### `get_effective_autonomy() -> float`

Calculates effective autonomy considering both behavioral preference and enterprise cap.

#### `consolidate() -> int`

Sleep-like consolidation: prunes noise in weights, decays failure counts, and reduces momentum. Returns the number of weights consolidated.

#### `get_normalized_tool_scores(candidates: List[str]) -> Dict[str, float]`

Returns frame-normalized tool scores in relative `[0, 1]` range, preventing anchoring bias.

#### `get_loss_framed_quality(tool_name: str) -> float`

Returns loss-framed quality perception for a tool using Bayesian posterior and prospect theory.

#### `compute_surprise_signal(prediction_quality: float, actual_quality: float) -> float`

Computes Bayesian surprise from prediction error. Returns a normalized signal in `[0, 1]`.

### Example

```python
from corteX.engine.weights import WeightEngine, WeightUpdate, WeightCategory

engine = WeightEngine()

# Behavioral update
engine.behavioral.update("verbosity", 0.3)

# Record tool usage with Bayesian tracking
engine.tools.record_use("code_interpreter", success=True, latency_ms=1500)

# Thompson Sampling tool selection
best = engine.tools.get_best_tool_thompson(["tool_a", "tool_b", "tool_c"])

# Batch update with audit trail
updates = [
    WeightUpdate(
        category=WeightCategory.BEHAVIORAL,
        key="autonomy",
        delta=0.1,
        reason="User confirmed good output",
        source="feedback_engine",
    ),
]
engine.batch_update(updates)

# Persist weights
engine.save("weights.json")
```
