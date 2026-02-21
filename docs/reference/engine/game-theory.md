# Game Theory

`corteX.engine.game_theory`

Strategic decision-making primitives for multi-agent systems. Provides Kahneman System 1/2 dual-process routing, reputation-based trust dynamics with quarantine, minimax safety for high-stakes decisions, Nash equilibrium routing optimization, Shapley value attribution, and truthful scoring mechanisms.

---

## ProcessType

Enum for cognitive process selection.

```python
class ProcessType(str, Enum):
    SYSTEM1 = "system1"  # Fast, heuristic, cached patterns
    SYSTEM2 = "system2"  # Slow, deliberate, full LLM reasoning
```

---

## EscalationContext

Context for deciding whether to escalate from System 1 to System 2.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `surprise_magnitude` | `float` | `0.0` | From PredictionEngine `[0, 1]` |
| `population_agreement` | `float` | `1.0` | From population decoder `[0, 1]` |
| `task_novelty` | `float` | `0.0` | How unfamiliar the pattern is `[0, 1]` |
| `enterprise_safety` | `float` | `0.0` | Enterprise risk level `[0, 1]` |
| `user_explicit_request` | `bool` | `False` | User asked for careful analysis |
| `error_in_last_step` | `bool` | `False` | Previous step produced an error |
| `goal_drift` | `float` | `0.0` | From GoalTracker `[0, 1]` |

---

## DualProcessRouter

Routes decisions through fast (System 1) or slow (System 2) paths based on escalation triggers. Any single trigger activates System 2.

### Constructor

```python
DualProcessRouter(
    surprise_threshold: float = 0.6,
    agreement_threshold: float = 0.4,
    novelty_threshold: float = 0.7,
    safety_threshold: float = 0.8,
    drift_threshold: float = 0.4,
)
```

### Methods

#### `route(context: EscalationContext) -> ProcessType`

Determines whether to use System 1 or System 2.

**Escalation triggers (any one activates System 2):**

| Trigger | Condition |
|---------|-----------|
| High surprise | `surprise_magnitude > 0.6` |
| Low agreement | `population_agreement < 0.4` |
| Novel task | `task_novelty > 0.7` |
| High safety concern | `enterprise_safety > 0.8` |
| User request | `user_explicit_request == True` |
| Previous error | `error_in_last_step == True` |
| Goal drift | `goal_drift > 0.4` |

#### `system2_ratio -> float` (property)

Fraction of all decisions that used System 2.

#### `last_escalation_reasons -> List[str]` (property)

Reasons for the most recent escalation (empty if System 1 was chosen).

#### `get_stats() -> Dict[str, Any]`

Returns: `system1_count`, `system2_count`, `system2_ratio`, `total_decisions`.

#### `to_dict() -> Dict[str, Any]`

Returns serialized state including thresholds and stats.

---

## ReputationSystem

Tracks tool/model reputation over iterated interactions using modified Tit-for-Tat with forgiveness and exponential quarantine.

### Constructor

```python
ReputationSystem(
    trust_alpha: float = 0.1,
    consistency_beta: float = 0.05,
    quarantine_threshold: int = 3,
    quarantine_base_seconds: float = 60.0,
)
```

### Methods

#### `record(tool: str, success: bool) -> float`

Records an interaction outcome and returns updated trust. Trust evolves via EMA with a consistency bonus. After N consecutive failures, the tool is quarantined with exponentially increasing duration (`base * 2^(failures - threshold)`).

#### `get_trust(tool: str) -> float`

Returns current trust (`0.0` to `1.0`). Returns `0.0` if the tool is quarantined. After quarantine expires, trust rebuilds from a reduced base.

#### `is_quarantined(tool: str) -> bool`

Returns whether a tool is currently quarantined.

#### `get_available_tools(candidates: List[str]) -> List[str]`

Filters out quarantined tools from candidates.

#### `get_ranked_tools(candidates: List[str]) -> List[Tuple[str, float]]`

Ranks available tools by trust, highest first.

#### `forgive(tool: str) -> None`

Manually ends quarantine and resets trust to `0.3` (human override).

#### `get_stats() -> Dict[str, Any]`

Returns `trust_scores`, `consistency_scores`, `quarantined` tools list, and `total_interactions` per tool.

#### `to_dict() -> Dict[str, Any]`

Returns serialized state including trust, consistency, history, and total interactions.

#### `from_dict(data: Dict[str, Any]) -> ReputationSystem` (classmethod)

Restores from serialized state.

---

## MinimaxSafetyGuard

Applies minimax reasoning for high-stakes enterprise decisions. Minimizes worst-case loss when stakes are high, maximizes expected gain when stakes are low.

### Constructor

```python
MinimaxSafetyGuard(risk_threshold: float = 0.7)
```

### Methods

#### `register_worst_case(action: str, max_loss: float) -> None`

Registers the worst-case loss for an action (`0.0` to `1.0`).

#### `select(candidates: List[str], expected_gains: Dict[str, float], enterprise_safety: float) -> str`

Selects an action. Below the risk threshold: pure expected-value maximization. Above: blended score = `(1 - safety_weight) * gain - safety_weight * worst_loss`.

#### `is_high_stakes(enterprise_safety: float) -> bool`

Returns whether the current safety level triggers minimax reasoning.

---

## NashRoutingOptimizer

Finds stable routing strategies using iterated best-response dynamics converging toward Nash Equilibrium for model-task assignment.

### Constructor

```python
NashRoutingOptimizer(
    models: Optional[List[str]] = None,
    task_types: Optional[List[str]] = None,
)
```

Default task types: `conversation`, `coding`, `planning`, `reasoning`, `summarization`, `validation`, `tool_use`.

### Methods

#### `add_model(model: str) -> None`

Registers a new model with uniform initial strategy.

#### `record_utility(model: str, task_type: str, quality: float, latency_ms: float, cost: float = 0.0) -> None`

Records observed utility for a model-task combination. Utility = `quality * speed_factor - cost`.

#### `best_response(model: str) -> Dict[str, float]`

Computes the best-response strategy: higher probability assigned to task types where this model has comparative advantage.

#### `iterate(steps: int = 10) -> None`

Runs iterated best-response dynamics to approach Nash Equilibrium.

#### `get_best_model(task_type: str) -> Optional[str]`

Returns the best model for a specific task type based on current strategies.

#### `get_assignment(task_type: str) -> List[Tuple[str, float]]`

Returns all models ranked by composite score for a task type.

---

## ShapleyAttributor

Computes Shapley values for fair multi-tool/model credit assignment. Answers: "How much did each component contribute to the outcome?"

Uses exact computation for N <= 8 players, Monte Carlo approximation for larger sets.

### Methods

#### `record_coalition_value(players: Set[str], value: float) -> None`

Records the outcome value for a coalition of tools/models.

#### `compute(all_players: Set[str]) -> Dict[str, float]`

Computes Shapley values (auto-selects exact vs. approximate).

#### `compute_exact(all_players: Set[str]) -> Dict[str, float]`

Exact O(2^N * N) computation for small player sets.

#### `compute_approximate(all_players: Set[str], num_permutations: int = 100) -> Dict[str, float]`

Monte Carlo approximation for larger player sets.

#### `get_credit_allocation(all_players: Set[str], total_reward: float) -> Dict[str, float]`

Allocates total reward among players based on Shapley values. Ensures allocations sum to `total_reward`.

#### `update_running(tool: str, marginal_contribution: float, alpha: float = 0.1) -> None`

Updates running (incremental) Shapley estimate via EMA.

#### `get_running_shapley() -> Dict[str, float]`

Returns the running (incremental) Shapley estimates for all tools. These are EMA-smoothed marginal contribution values updated via `update_running()`.

---

## TruthfulScoringMechanism

VCG-inspired mechanism that incentivizes tools to honestly report capabilities. A tool's score is based on its marginal contribution, not self-reported capabilities.

### Methods

#### `declare(tool: str, capabilities: Dict[str, float]) -> None`

Tool declares its capabilities (e.g., `{"speed": 0.8, "accuracy": 0.9}`).

#### `observe(tool: str, actual_performance: Dict[str, float]) -> None`

Records actual observed performance (EMA with alpha `0.15`).

#### `credibility_score(tool: str) -> float`

Returns how well declared capabilities match observed performance. `1.0` = perfectly honest, `0.0` = complete mismatch.

#### `adjusted_score(tool: str, raw_score: float) -> float`

Adjusts a raw score by credibility: `raw_score * credibility`.

#### `get_all_credibilities() -> Dict[str, float]`

Returns credibility scores for all known tools (both declared and observed). Useful for dashboard views and auditing which tools are reporting honestly.

### Example

```python
from corteX.engine.game_theory import (
    DualProcessRouter, EscalationContext,
    ReputationSystem, ShapleyAttributor,
)

# Dual-process routing
router = DualProcessRouter()
process = router.route(EscalationContext(
    surprise_magnitude=0.8,  # high surprise -> System 2
    task_novelty=0.3,
))
# process == ProcessType.SYSTEM2
print(router.to_dict())  # serialized state with thresholds and stats

# Reputation tracking
reputation = ReputationSystem()
for _ in range(5):
    reputation.record("flaky_tool", success=True)
reputation.record("flaky_tool", success=False)
reputation.record("flaky_tool", success=False)
reputation.record("flaky_tool", success=False)
# flaky_tool is now quarantined
print(reputation.get_stats())  # trust scores, consistency, quarantined list

# Shapley attribution
shapley = ShapleyAttributor()
shapley.record_coalition_value({"tool_a"}, 0.6)
shapley.record_coalition_value({"tool_b"}, 0.4)
shapley.record_coalition_value({"tool_a", "tool_b"}, 0.9)
credits = shapley.compute({"tool_a", "tool_b"})
# credits ~ {"tool_a": 0.55, "tool_b": 0.35}

# Running Shapley estimates
shapley.update_running("tool_a", 0.6)
shapley.update_running("tool_b", 0.3)
print(shapley.get_running_shapley())  # {"tool_a": 0.06, "tool_b": 0.03}
```
