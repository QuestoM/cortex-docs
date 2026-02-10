# Decision Theory, Game Theory & Bayesian Mathematics for corteX

## Research Document: Enhancing the Brain-Inspired Weight Engine

**Author**: corteX Research Agent
**Date**: 2026-02-09
**Status**: Research Complete -- Ready for Implementation Planning

---

## Table of Contents

1. [Part I: Kahneman's Decision Theory for AI Agent Weights](#part-i-kahnemans-decision-theory)
2. [Part II: Game Theory for Multi-Agent Decision Making](#part-ii-game-theory)
3. [Part III: Bayesian Mathematics for Probabilistic Weight Updates](#part-iii-bayesian-mathematics)
4. [Integration Map: Where Each Pattern Fits in corteX](#integration-map)
5. [Priority Ranking](#priority-ranking)
6. [References](#references)

---

## Part I: Kahneman's Decision Theory

### 1.1 System 1 vs. System 2: Dual-Process Theory for Agent Decisions

**Theoretical Background**

Daniel Kahneman's dual-process theory (Thinking, Fast and Slow) describes two
modes of cognition:

- **System 1**: Fast, automatic, intuitive, low-effort. Operates on pattern
  recognition and heuristics. Always active.
- **System 2**: Slow, deliberate, analytical, high-effort. Engages when System 1
  encounters novelty, complexity, or high stakes.

The brain does not always engage System 2 -- doing so would be metabolically
prohibitive. Instead, System 1 handles ~98% of decisions, and System 2 activates
only when surprise, conflict, or high stakes are detected.

**Mapping to corteX Agent Architecture**

| Brain System | corteX Equivalent | Implementation |
|---|---|---|
| System 1 (Fast) | Cached patterns, weight lookups, heuristic routing | `PopulationDecoder`, `ToolPreferenceWeights.get_best_tool()` |
| System 2 (Slow) | Full LLM reasoning, multi-step planning, deep analysis | `LLMRouter.generate()` with `thinking=True`, `GoalTracker.verify_step()` |
| Conflict monitor (ACC) | Surprise signal magnitude | `PredictionEngine.compare()` -> `SurpriseSignal` |
| Engagement threshold | When to escalate from S1 to S2 | New: `DualProcessRouter` class |

**Concrete SDK Pattern: `DualProcessRouter`**

The agent should use System 1 (fast path) by default, and escalate to System 2
(slow path) when specific triggers are met:

```python
class DualProcessRouter:
    """
    Routes decisions through fast (System 1) or slow (System 2) paths.
    Escalation triggers:
    1. Surprise magnitude > threshold (prediction error)
    2. Task novelty > threshold (no cached pattern)
    3. Stakes level > threshold (enterprise safety)
    4. Conflict detected (population disagreement)
    5. Explicit user request for careful analysis
    """

    def __init__(self, weight_engine: WeightEngine, prediction_engine: PredictionEngine):
        self.weights = weight_engine
        self.predictor = prediction_engine
        self.escalation_threshold = 0.6  # Surprise level to trigger S2
        self.novelty_threshold = 0.7     # How unfamiliar the pattern is
        self.stakes_threshold = 0.8      # Enterprise safety concern

    def should_escalate(
        self,
        surprise: Optional[SurpriseSignal],
        population_agreement: float,
        task_novelty: float,
        enterprise_safety: float,
    ) -> bool:
        """Determine if System 2 engagement is needed."""
        # High surprise = prediction was wrong = need deeper analysis
        if surprise and surprise.surprise_magnitude > self.escalation_threshold:
            return True
        # Low population agreement = evaluators disagree = uncertain
        if population_agreement < 0.4:
            return True
        # Novel task with no cached patterns
        if task_novelty > self.novelty_threshold:
            return True
        # High-stakes enterprise context
        if enterprise_safety > self.stakes_threshold:
            return True
        return False

    def route(self, context: dict) -> str:
        """Returns 'system1' or 'system2'."""
        if self.should_escalate(**context):
            return "system2"
        return "system1"
```

**Integration Point**: `corteX/engine/prediction.py` -> New file
`corteX/engine/dual_process.py`. Called from `corteX/runtime/orchestrator.py`
before each LLM call. When System 1, use `prefer_speed=True` on the router.
When System 2, use `thinking=True` with full reasoning budget.

### 1.2 Prospect Theory: Loss Aversion in Weight Updates

**Theoretical Background**

Kahneman and Tversky's Prospect Theory (1979) identifies three key departures
from rational utility theory:

1. **Reference Dependence**: Value is perceived relative to a reference point,
   not in absolute terms.
2. **Loss Aversion**: Losses loom larger than gains. The pain of losing $100 is
   approximately 2-2.5x the pleasure of gaining $100.
3. **Diminishing Sensitivity**: The marginal impact of gains/losses decreases
   as their magnitude increases.

**The Value Function**

The prospect theory value function is defined as:

```
v(x) = x^alpha           if x >= 0  (gains)
v(x) = -lambda * |x|^beta    if x < 0   (losses)
```

Where:
- `alpha ~= 0.88` (diminishing sensitivity for gains)
- `beta ~= 0.88` (diminishing sensitivity for losses)
- `lambda ~= 2.25` (loss aversion coefficient)
- `x` = deviation from reference point

**The Probability Weighting Function**

Prospect theory also transforms probabilities through a weighting function:

```
w(p) = p^gamma / (p^gamma + (1-p)^gamma)^(1/gamma)
```

Where `gamma ~= 0.61` (Tversky & Kahneman, 1992). This overweights small
probabilities and underweights large ones -- explaining why agents should pay
outsized attention to rare failures.

**Application to corteX Weight Updates**

Currently, `ToolPreferenceWeights.record_use()` uses symmetric EMA:
```python
t["success_rate"] = t["success_rate"] * (1 - alpha) + (1.0 if success else 0.0) * alpha
```

This treats successes and failures symmetrically. Prospect theory demands
asymmetry.

**Concrete SDK Pattern: `ProspectTheoreticUpdater`**

```python
class ProspectTheoreticUpdater:
    """
    Applies Kahneman-Tversky prospect theory to weight updates.
    Failures (losses) are weighted 2.25x more than successes (gains).
    """

    def __init__(
        self,
        loss_aversion: float = 2.25,   # lambda
        gain_sensitivity: float = 0.88, # alpha
        loss_sensitivity: float = 0.88, # beta
    ):
        self.loss_aversion = loss_aversion
        self.gain_sensitivity = gain_sensitivity
        self.loss_sensitivity = loss_sensitivity

    def compute_value(self, outcome_delta: float, reference_point: float = 0.5) -> float:
        """
        Compute the subjective value of an outcome relative to reference.
        outcome_delta > 0 = gain (success), < 0 = loss (failure)
        """
        x = outcome_delta - reference_point
        if x >= 0:
            return x ** self.gain_sensitivity
        else:
            return -self.loss_aversion * (abs(x) ** self.loss_sensitivity)

    def asymmetric_update(
        self,
        current_weight: float,
        success: bool,
        learning_rate: float = 0.08,
    ) -> float:
        """
        Update weight with loss-aversion asymmetry.
        A single failure has 2.25x the impact of a single success.
        """
        if success:
            delta = learning_rate * (1.0 ** self.gain_sensitivity)
        else:
            delta = -learning_rate * self.loss_aversion * (1.0 ** self.loss_sensitivity)
        return max(0.0, min(1.0, current_weight + delta))
```

**Integration Point**: Replace the symmetric EMA in
`corteX/engine/weights.py` `ToolPreferenceWeights.record_use()` and
`ModelSelectionWeights.update()`. The reference point should be the running
average success rate (adapts over time).

### 1.3 Anchoring Bias: Initial Weights Shape Future Behavior

**Theoretical Background**

Anchoring (Tversky & Kahneman, 1974) occurs when an initial piece of
information disproportionately influences subsequent judgments. Even when the
anchor is arbitrary, subsequent estimates are biased toward it.

**Application to corteX**

In the current system, `ToolPreferenceWeights` initializes all tools with
`success_rate: 0.5`, `preference_score: 0.5`. This 0.5 anchor means:

- **Harmful anchoring**: A genuinely bad tool starts at 0.5 and takes multiple
  failures to drop, causing wasted execution in the meantime.
- **Beneficial anchoring**: A tool with known historical performance should be
  anchored to its historical rate, not 0.5.

**Concrete SDK Pattern: Informed Anchor Initialization**

```python
class AnchorManager:
    """
    Manages initial weight anchors using historical priors.
    Prevents arbitrary 0.5 initialization from dominating early behavior.
    """

    def __init__(self):
        self._historical_anchors: Dict[str, float] = {}
        self._anchor_confidence: Dict[str, float] = {}

    def set_anchor(self, key: str, value: float, confidence: float) -> None:
        """Set an informed anchor based on historical data or global weights."""
        self._historical_anchors[key] = value
        self._anchor_confidence[key] = confidence

    def get_initial_weight(self, key: str) -> Tuple[float, float]:
        """
        Get initial weight and confidence for a new entity.
        Returns (weight, confidence). Higher confidence = stronger anchor.
        """
        if key in self._historical_anchors:
            return self._historical_anchors[key], self._anchor_confidence[key]
        # Uninformed prior: 0.5 with low confidence (easy to update)
        return 0.5, 0.1

    def debiasing_rate(self, confidence: float) -> float:
        """
        Higher confidence anchors require more evidence to move.
        Low confidence anchors are easy to update (good for exploration).
        """
        # Learning rate inversely proportional to anchor confidence
        return max(0.02, 0.15 * (1 - confidence))
```

**Integration Point**: `corteX/engine/weights.py`
`ToolPreferenceWeights.register_tool()` -- replace hardcoded 0.5 initialization.
Feed from `GlobalWeights` or episodic memory for informed anchors.

### 1.4 Availability Heuristic: Recency Bias in Memory

**Theoretical Background**

The availability heuristic (Tversky & Kahneman, 1973) causes people to
overweight recent, vivid, or emotionally salient events when making judgments.
A tool that failed spectacularly yesterday looms larger than one that succeeded
quietly a hundred times.

**Application to corteX**

The current EMA in `ToolPreferenceWeights` already has recency bias built in
(recent events have higher weight). However, this is uncontrolled -- we should
be deliberate about when recency bias helps and when it hurts.

**When Recency Bias Helps**:
- Detecting real changes in tool reliability (a tool's API went down recently)
- Adapting to shifting user preferences within a session
- Reacting to environmental changes (new model version deployed)

**When Recency Bias Hurts**:
- Overreacting to a single anomalous failure
- Ignoring long-term reliable performance due to one bad result
- Catastrophizing a single slow response

**Concrete SDK Pattern: Controlled Availability**

```python
class AvailabilityFilter:
    """
    Controls recency bias in weight updates.
    Uses a two-window approach: short-term (recent) and long-term (stable).
    Detects when recent events genuinely differ from baseline vs. noise.
    """

    def __init__(self, short_window: int = 5, long_window: int = 50):
        self.short_window = short_window
        self.long_window = long_window
        self._events: Dict[str, List[Tuple[float, bool]]] = {}

    def record(self, key: str, timestamp: float, success: bool) -> None:
        if key not in self._events:
            self._events[key] = []
        self._events[key].append((timestamp, success))
        # Trim to long window
        if len(self._events[key]) > self.long_window * 2:
            self._events[key] = self._events[key][-self.long_window * 2:]

    def get_adjusted_rate(self, key: str) -> Tuple[float, bool]:
        """
        Returns (adjusted_success_rate, is_anomalous).
        If short-term rate significantly differs from long-term,
        flags it as anomalous (might be real change or noise).
        """
        events = self._events.get(key, [])
        if len(events) < self.short_window:
            return 0.5, False

        recent = events[-self.short_window:]
        historical = events[-self.long_window:] if len(events) >= self.long_window else events

        recent_rate = sum(1 for _, s in recent if s) / len(recent)
        historical_rate = sum(1 for _, s in historical if s) / len(historical)

        deviation = abs(recent_rate - historical_rate)
        is_anomalous = deviation > 0.3  # Significant deviation

        if is_anomalous:
            # Blend: trust recent more when deviation is real
            adjusted = 0.7 * recent_rate + 0.3 * historical_rate
        else:
            # Stable: trust long-term more
            adjusted = 0.3 * recent_rate + 0.7 * historical_rate

        return adjusted, is_anomalous
```

**Integration Point**: `corteX/engine/weights.py`
`ToolPreferenceWeights.record_use()` -- wrap the EMA update with availability
filtering. Also integrates with `corteX/engine/adaptation.py`
`SustainedAdaptation` for detecting genuine behavioral shifts.

### 1.5 Framing Effects: How Context Shapes Decisions

**Theoretical Background**

Framing effects (Tversky & Kahneman, 1981) demonstrate that the same
information presented differently leads to different decisions. "90% success
rate" feels different from "10% failure rate" even though they are identical.

**Application to corteX**

When the agent presents options to a user or makes internal decisions, framing
matters:

- **Internal framing**: How tool scores are compared. A tool with score 0.7 out
  of 1.0 looks different when framed as "30% below maximum" vs. "40% above
  minimum."
- **Relative framing**: Tool A (0.75) vs. Tool B (0.70) looks like a close
  race. But framed as "Tool A is 7% better than Tool B," it becomes decisive.
- **Loss framing**: "This tool has failed 3 of last 10 times" is more
  impactful than "this tool succeeded 7 of last 10 times."

**Concrete SDK Pattern: `FrameNormalizer`**

```python
class FrameNormalizer:
    """
    Normalizes decision framing to prevent framing-induced biases.
    Presents all comparisons in a consistent frame.
    """

    @staticmethod
    def normalize_scores(scores: Dict[str, float]) -> Dict[str, float]:
        """
        Normalize scores to relative frame.
        Prevents the anchoring effect of absolute scores.
        """
        if not scores:
            return {}
        min_s = min(scores.values())
        max_s = max(scores.values())
        range_s = max_s - min_s
        if range_s < 0.01:
            return {k: 0.5 for k in scores}
        return {k: (v - min_s) / range_s for k, v in scores.items()}

    @staticmethod
    def loss_frame_penalty(success_rate: float) -> float:
        """
        Apply loss-framing: present failure rate perspective.
        A 0.9 success rate = 0.1 failure rate.
        With loss aversion (2.25x), the perceived quality drops.
        Perceived quality = success_rate - 2.25 * failure_rate (normalized)
        """
        failure_rate = 1.0 - success_rate
        perceived = success_rate - 2.25 * failure_rate
        # Normalize back to 0-1 range
        # At success_rate=1.0: perceived = 1.0
        # At success_rate=0.5: perceived = 0.5 - 2.25*0.5 = -0.625
        # Normalize: (perceived + 2.25) / (1 + 2.25) = (perceived + 2.25) / 3.25
        return max(0.0, min(1.0, (perceived + 2.25) / 3.25))
```

**Integration Point**: `corteX/engine/population.py`
`PopulationToolSelector.select()` -- normalize scores before comparison.
Also useful in `corteX/core/llm/router.py` `LLMRouter._select_model()`.

---

## Part II: Game Theory for Multi-Agent Decision Making

### 2.1 Nash Equilibrium: Stable Tool/Model Selection Strategies

**Theoretical Background**

A Nash Equilibrium is a state where no player can improve their outcome by
unilaterally changing their strategy. In a multi-agent system, this represents a
stable configuration where each component's strategy is optimal given what the
others are doing.

For corteX, the "players" are:
- Available LLM models (competing for task assignment)
- Available tools (competing for execution)
- Task types (competing for compute budget)

**Application to corteX**

When multiple models are available for a task type, the system should converge
to a stable routing strategy (Nash Equilibrium) where:
- Each model handles the task types it is best at
- No model would benefit from being assigned to a different task type
- The overall system performance is locally optimal

**Mathematical Formulation**

For models M = {m1, m2, ..., mn} and task types T = {t1, t2, ..., tk}:

```
Strategy profile: sigma = (sigma_1, ..., sigma_n) where sigma_i: T -> [0,1]
                  sigma_i(t) = probability model i is assigned task type t

Nash Equilibrium: For all i, for all sigma_i':
  U_i(sigma_i, sigma_{-i}) >= U_i(sigma_i', sigma_{-i})

Where U_i = sum_t sigma_i(t) * quality(i,t) * (1/latency(i,t)) - cost(i,t)
```

**Concrete SDK Pattern: `NashRoutingOptimizer`**

```python
class NashRoutingOptimizer:
    """
    Finds stable routing strategies using iterative best-response dynamics.
    Converges toward Nash Equilibrium for model-task assignment.
    """

    def __init__(self, models: List[str], task_types: List[str]):
        self.models = models
        self.task_types = task_types
        # Strategy: model -> task_type -> assignment probability
        n = len(models)
        k = len(task_types)
        # Initialize uniform
        self.strategy: Dict[str, Dict[str, float]] = {
            m: {t: 1.0 / k for t in task_types} for m in models
        }
        self.utilities: Dict[str, Dict[str, float]] = {
            m: {t: 0.5 for t in task_types} for m in models
        }

    def update_utility(self, model: str, task_type: str, quality: float,
                       latency_ms: float, cost: float = 0.0) -> None:
        """Record observed utility for a model-task combination."""
        # Utility = quality-weighted speed minus cost
        speed_factor = 1.0 / max(latency_ms / 1000.0, 0.1)
        utility = quality * speed_factor - cost
        # EMA update
        alpha = 0.1
        old = self.utilities[model][task_type]
        self.utilities[model][task_type] = old * (1 - alpha) + utility * alpha

    def best_response(self, model: str) -> Dict[str, float]:
        """
        Compute the best response strategy for a model given others' strategies.
        Assigns higher probability to task types where this model has
        comparative advantage.
        """
        utils = self.utilities[model]
        total = sum(max(0, u) for u in utils.values())
        if total == 0:
            return {t: 1.0 / len(self.task_types) for t in self.task_types}
        return {t: max(0, u) / total for t, u in utils.items()}

    def iterate(self, steps: int = 10) -> None:
        """Run iterated best-response to approach Nash Equilibrium."""
        for _ in range(steps):
            for model in self.models:
                self.strategy[model] = self.best_response(model)

    def get_assignment(self, task_type: str) -> List[Tuple[str, float]]:
        """Get ranked models for a task type with assignment probabilities."""
        scored = []
        for model in self.models:
            prob = self.strategy[model].get(task_type, 0)
            utility = self.utilities[model].get(task_type, 0)
            scored.append((model, prob * utility))
        scored.sort(key=lambda x: x[1], reverse=True)
        return scored
```

**Integration Point**: `corteX/core/llm/router.py` `LLMRouter._select_model()`.
Run `iterate()` periodically (e.g., at session end during consolidation) to
update routing strategies.

### 2.2 Minimax: Risk Minimization for High-Stakes Decisions

**Theoretical Background**

The minimax principle (von Neumann, 1928) seeks to minimize the maximum
possible loss. In decision theory under uncertainty, the minimax strategy
assumes an adversarial environment and prepares for the worst case.

```
Minimax strategy: argmin_a max_s Loss(a, s)

Where:
  a = action (tool/model choice)
  s = possible state of the world (failure modes)
  Loss(a, s) = cost if action a is taken and state s occurs
```

**Application to corteX Enterprise Decisions**

For high-stakes enterprise tasks (data_sensitivity > 0.8, compliance_rules
present), the agent should use minimax thinking:

- What is the worst thing that could happen if I use this tool?
- What is the maximum damage of a wrong model output?
- Which choice minimizes the worst-case scenario?

**Concrete SDK Pattern: `MinimaxSafetyGuard`**

```python
class MinimaxSafetyGuard:
    """
    Applies minimax reasoning for high-stakes enterprise decisions.
    Selects actions that minimize worst-case loss.
    """

    def __init__(self, risk_threshold: float = 0.7):
        self.risk_threshold = risk_threshold
        # Worst-case loss estimates per tool/action
        self._worst_case_losses: Dict[str, float] = {}

    def register_worst_case(self, action: str, max_loss: float) -> None:
        """Register the worst-case loss for an action."""
        self._worst_case_losses[action] = max_loss

    def minimax_select(
        self,
        candidates: List[str],
        expected_gains: Dict[str, float],
        enterprise_safety: float,
    ) -> str:
        """
        Select action using minimax when stakes are high.
        Falls back to expected-value maximization when stakes are low.
        """
        if enterprise_safety < self.risk_threshold:
            # Low stakes: maximize expected gain (normal behavior)
            return max(candidates, key=lambda c: expected_gains.get(c, 0))

        # High stakes: minimize worst-case loss
        def minimax_score(action: str) -> float:
            worst_loss = self._worst_case_losses.get(action, 0.5)
            expected_gain = expected_gains.get(action, 0.0)
            # Blend: mostly minimize loss, some weight on gain
            return -worst_loss * 0.8 + expected_gain * 0.2

        return max(candidates, key=minimax_score)
```

**Integration Point**: `corteX/runtime/orchestrator.py` -- wrap tool selection
with minimax guard when `enterprise.get("data_sensitivity") > threshold`. Also
integrates with `corteX/plugins/agents/supervisor.py` for policy enforcement.

### 2.3 Mechanism Design: Truthful Capability Reporting

**Theoretical Background**

Mechanism design (Myerson, Maskin -- Nobel Prize 2007) is "reverse game theory":
designing the rules of the game so that strategic agents truthfully reveal
information. The Vickrey-Clarke-Groves (VCG) mechanism ensures truthful
reporting is the dominant strategy.

**Application to corteX**

Tools and models in corteX self-report capabilities (speed tier, supported
tasks). But what if a tool "lies" (due to misconfiguration or optimistic
metadata)? The scoring system should incentivize truthful self-reporting.

**Concrete SDK Pattern: Incentive-Compatible Scoring**

```python
class TruthfulScoringMechanism:
    """
    Designs scoring incentives so that tools benefit from honest
    capability reporting.

    VCG-inspired: A tool's score is based on the marginal contribution
    it makes to overall system performance, not self-reported capabilities.
    """

    def __init__(self):
        self._declared_capabilities: Dict[str, Dict[str, float]] = {}
        self._observed_performance: Dict[str, Dict[str, float]] = {}

    def declare(self, tool: str, capabilities: Dict[str, float]) -> None:
        """Tool declares its capabilities."""
        self._declared_capabilities[tool] = capabilities

    def observe(self, tool: str, actual_performance: Dict[str, float]) -> None:
        """Record actual observed performance."""
        if tool not in self._observed_performance:
            self._observed_performance[tool] = {}
        for k, v in actual_performance.items():
            alpha = 0.15
            old = self._observed_performance[tool].get(k, v)
            self._observed_performance[tool][k] = old * (1 - alpha) + v * alpha

    def credibility_score(self, tool: str) -> float:
        """
        How well do declared capabilities match observed performance?
        High credibility = tool reports honestly.
        """
        declared = self._declared_capabilities.get(tool, {})
        observed = self._observed_performance.get(tool, {})
        if not declared or not observed:
            return 0.5

        errors = []
        for key in declared:
            if key in observed:
                errors.append(abs(declared[key] - observed[key]))
        if not errors:
            return 0.5

        avg_error = sum(errors) / len(errors)
        return max(0.0, 1.0 - avg_error * 2)

    def adjusted_score(self, tool: str, raw_score: float) -> float:
        """
        Adjust a tool's score by its credibility.
        Honest tools get their full score. Dishonest tools are penalized.
        """
        credibility = self.credibility_score(tool)
        return raw_score * credibility
```

**Integration Point**: `corteX/engine/weights.py`
`ToolPreferenceWeights.get_preference()` -- multiply by credibility score.
Feed declarations from tool registration in `corteX/tools/executor.py`.

### 2.4 Cooperative Game Theory: Shapley Values for Attribution

**Theoretical Background**

Shapley values (Shapley, 1953) provide a mathematically fair way to distribute
credit among cooperating players. For a coalitional game with N players:

```
phi_i(v) = sum_{S subset N, i not in S}
    [|S|! * (|N|-|S|-1)! / |N|!] * [v(S union {i}) - v(S)]
```

Where:
- `phi_i(v)` = player i's Shapley value (fair contribution)
- `v(S)` = value generated by coalition S
- The sum is over all possible coalitions not containing i

Properties (uniquely characterized):
- **Efficiency**: Contributions sum to total value
- **Symmetry**: Equal players get equal credit
- **Additivity**: Contributions across games add up
- **Dummy player**: Non-contributors get zero

**Application to corteX**

When multiple tools and models collaborate on a task, Shapley values determine
how much credit each component deserves for the final outcome.

**Concrete SDK Pattern: `ShapleyAttributor`**

```python
import itertools
import math

class ShapleyAttributor:
    """
    Computes Shapley values for multi-tool/model task attribution.
    Answers: "How much did each component contribute to the outcome?"
    """

    def __init__(self):
        self._coalition_values: Dict[frozenset, float] = {}

    def record_coalition_value(self, players: Set[str], value: float) -> None:
        """Record the outcome value for a coalition of tools/models."""
        self._coalition_values[frozenset(players)] = value

    def compute_shapley(self, all_players: Set[str]) -> Dict[str, float]:
        """
        Compute Shapley values for each player.
        For N players, this requires evaluating 2^N coalitions.
        For small N (typical: 2-6 tools + 1-2 models), this is tractable.
        """
        n = len(all_players)
        players_list = list(all_players)
        shapley_values: Dict[str, float] = {p: 0.0 for p in all_players}

        for i, player in enumerate(players_list):
            others = [p for p in players_list if p != player]
            for size in range(len(others) + 1):
                for coalition in itertools.combinations(others, size):
                    coalition_set = frozenset(coalition)
                    coalition_with_player = frozenset(coalition) | {player}
                    v_with = self._coalition_values.get(coalition_with_player, 0.0)
                    v_without = self._coalition_values.get(coalition_set, 0.0)
                    marginal = v_with - v_without
                    # Shapley weight
                    weight = (math.factorial(size) * math.factorial(n - size - 1)
                              / math.factorial(n))
                    shapley_values[player] += weight * marginal

        return shapley_values

    def approximate_shapley(self, all_players: Set[str],
                            num_permutations: int = 100) -> Dict[str, float]:
        """
        Monte Carlo approximation for larger player sets.
        Sample random permutations and average marginal contributions.
        """
        import random
        players_list = list(all_players)
        shapley_values: Dict[str, float] = {p: 0.0 for p in all_players}

        for _ in range(num_permutations):
            perm = players_list[:]
            random.shuffle(perm)
            coalition = set()
            for player in perm:
                v_with = self._coalition_values.get(
                    frozenset(coalition | {player}), 0.0)
                v_without = self._coalition_values.get(
                    frozenset(coalition), 0.0)
                shapley_values[player] += (v_with - v_without)
                coalition.add(player)

        # Average
        for p in shapley_values:
            shapley_values[p] /= num_permutations

        return shapley_values
```

**Integration Point**: `corteX/engine/plasticity.py` `HebbianRule.apply()` --
instead of equal credit, distribute using Shapley values. Also feeds into
`corteX/engine/weights.py` for multi-tool weight updates.

### 2.5 Repeated Games: Learning Tool Reliability Over Time

**Theoretical Background**

In the iterated Prisoner's Dilemma (Axelrod, 1984), cooperation strategies like
Tit-for-Tat emerge when games are repeated. Key insights:

- **Reputation matters**: Past behavior predicts future behavior.
- **Forgiveness helps**: Occasional failures should not permanently destroy trust.
- **Reciprocity**: Tools that consistently deliver get more opportunities.
- **Grim trigger**: A threshold below which trust is permanently lost (but can
  be recovered through sustained good behavior).

**Application to corteX**

The relationship between corteX and each tool is an iterated game. Each
invocation is one "round." The trust dynamics should follow:

```
Trust evolution:
  trust(t+1) = trust(t) + alpha * (outcome - trust(t))     # Basic EMA

Tit-for-Tat variant:
  trust(t+1) = trust(t) + alpha * (outcome - trust(t))
               + beta * (consistency - 0.5)                  # Consistency bonus
               - gamma * (recent_failures > threshold)        # Grim trigger
```

Where `consistency` measures how predictable the tool's performance is (low
variance = high consistency), and the grim trigger activates quarantine for
unreliable tools.

**Concrete SDK Pattern: `ReputationSystem`**

```python
class ReputationSystem:
    """
    Tracks tool/model reputation over iterated interactions.
    Implements modified Tit-for-Tat with forgiveness and grim trigger.
    """

    def __init__(self):
        self._trust: Dict[str, float] = {}
        self._consistency: Dict[str, float] = {}
        self._history: Dict[str, List[bool]] = {}
        self._quarantined: Dict[str, float] = {}  # tool -> quarantine_until

    def record(self, tool: str, success: bool) -> float:
        """Record interaction outcome and return updated trust."""
        if tool not in self._trust:
            self._trust[tool] = 0.5
            self._history[tool] = []
            self._consistency[tool] = 0.5

        self._history[tool].append(success)

        # Trust update (EMA)
        alpha = 0.1
        outcome = 1.0 if success else 0.0
        self._trust[tool] += alpha * (outcome - self._trust[tool])

        # Consistency update (inverse variance of recent outcomes)
        recent = self._history[tool][-20:]
        if len(recent) > 2:
            mean = sum(1 for s in recent if s) / len(recent)
            variance = sum((1.0 if s else 0.0 - mean) ** 2
                          for s in recent) / len(recent)
            self._consistency[tool] = max(0.0, 1.0 - variance * 4)

        # Grim trigger: quarantine after N consecutive failures
        consecutive_failures = 0
        for s in reversed(self._history[tool]):
            if not s:
                consecutive_failures += 1
            else:
                break
        if consecutive_failures >= 3:
            import time
            quarantine_duration = 60 * (2 ** consecutive_failures)
            self._quarantined[tool] = time.time() + quarantine_duration

        return self._trust[tool]

    def get_trust(self, tool: str) -> float:
        """Get current trust level (0.0 to 1.0)."""
        import time
        if tool in self._quarantined:
            if time.time() < self._quarantined[tool]:
                return 0.0  # Quarantined
            else:
                del self._quarantined[tool]
        return self._trust.get(tool, 0.5)

    def is_quarantined(self, tool: str) -> bool:
        import time
        if tool in self._quarantined:
            if time.time() < self._quarantined[tool]:
                return True
            del self._quarantined[tool]
        return False
```

**Integration Point**: `corteX/engine/weights.py`
`ToolPreferenceWeights.get_preference()` -- multiply by reputation trust score.
Also integrates with `corteX/engine/plasticity.py` `LTDRule` for the grim
trigger mechanism.

### 2.6 Bayesian Games: Decisions Under Incomplete Information

**Theoretical Background**

In a Bayesian game (Harsanyi, 1967-68), players have incomplete information
about other players' types. Each player has a prior belief over others' types
and updates beliefs through observed actions.

**Application to corteX**

When a new tool is registered, corteX has incomplete information about its true
capabilities. The agent holds a prior belief (from declared capabilities or
global weights) and updates through observation:

```
Type space: theta_i in Theta_i (tool i's true quality type)
Prior: p(theta_i) = initial belief about tool quality
Update: p(theta_i | observations) = posterior after seeing results
Strategy: sigma_i(theta_i) = how to use tool i given believed type
```

**Concrete SDK Pattern**: This maps directly to the Bayesian updating framework
covered in Part III. The game-theoretic insight is that the agent should reason
about the *information structure*: which observations are most informative about
a tool's true type, and how to design experiments (exploration) that maximally
reduce uncertainty.

**Integration Point**: Connects the Bayesian inference system (Part III) with
the game-theoretic framework. The exploration strategy in Thompson Sampling
(Section 3.2) is the optimal Bayesian game strategy.

---

## Part III: Bayesian Mathematics for Probabilistic Weight Updates

### 3.1 Bayesian Updating: From EMA to Proper Posterior Inference

**Theoretical Background**

Bayesian inference follows Bayes' rule:

```
P(theta | data) = P(data | theta) * P(theta) / P(data)
  posterior     =   likelihood    *   prior   / evidence
```

The current EMA approach in corteX is a special case of Bayesian updating
with a specific implicit prior. But proper Bayesian updating offers:

1. **Uncertainty quantification**: Not just a point estimate, but a full
   distribution over possible values.
2. **Prior incorporation**: Historical knowledge encoded as priors.
3. **Coherent updating**: Each observation updates beliefs in a
   mathematically consistent way.
4. **Natural exploration**: Uncertainty drives exploration (Thompson Sampling).

**Current EMA vs. Bayesian Updating**

| Feature | EMA (current) | Bayesian (proposed) |
|---|---|---|
| Output | Point estimate (single number) | Full distribution (mean + uncertainty) |
| Memory | Implicit (exponential decay) | Explicit (prior parameters) |
| Uncertainty | Not tracked | Natural byproduct |
| Exploration | Not addressed | Thompson Sampling |
| Prior knowledge | Hardcoded 0.5 | Informed priors |
| Learning rate | Fixed alpha | Adaptive (shrinks with evidence) |

### 3.2 Thompson Sampling: Exploration vs. Exploitation

**Theoretical Background**

Thompson Sampling (Thompson, 1933) is a Bayesian approach to the multi-armed
bandit problem that elegantly balances exploration and exploitation:

```
Algorithm:
1. For each arm (tool/model), maintain a posterior distribution
   over its quality parameter.
2. Sample one value from each posterior.
3. Select the arm with the highest sampled value.
4. Observe the reward and update the posterior.
```

This naturally explores uncertain options (wide posteriors produce occasionally
high samples) while exploiting known good options (peaked posteriors produce
consistently high samples).

**Why Thompson Sampling for corteX**

The current `ToolPreferenceWeights.get_best_tool()` always selects the tool
with the highest point estimate. This is purely exploitative -- it never explores
alternatives that might be better but are uncertain. Thompson Sampling fixes this.

**Concrete SDK Pattern: `BayesianToolSelector`**

```python
import random
import math

class BetaDistribution:
    """
    Beta distribution for binary outcomes (success/failure).
    Conjugate prior for Bernoulli likelihood.
    Beta(alpha, beta) where:
      alpha = 1 + number of successes
      beta = 1 + number of failures
      mean = alpha / (alpha + beta)
      variance = alpha * beta / ((alpha + beta)^2 * (alpha + beta + 1))
    """

    def __init__(self, alpha: float = 1.0, beta: float = 1.0):
        self.alpha = alpha  # pseudo-count of successes + 1
        self.beta = beta    # pseudo-count of failures + 1

    def update(self, success: bool) -> None:
        """Update posterior with new observation."""
        if success:
            self.alpha += 1.0
        else:
            self.beta += 1.0

    @property
    def mean(self) -> float:
        return self.alpha / (self.alpha + self.beta)

    @property
    def variance(self) -> float:
        ab = self.alpha + self.beta
        return (self.alpha * self.beta) / (ab * ab * (ab + 1))

    @property
    def std(self) -> float:
        return math.sqrt(self.variance)

    @property
    def confidence_interval_95(self) -> Tuple[float, float]:
        """Approximate 95% CI using normal approximation."""
        return (max(0, self.mean - 2 * self.std),
                min(1, self.mean + 2 * self.std))

    def sample(self) -> float:
        """Draw a single sample from the Beta distribution."""
        # Use the gamma function trick for Beta sampling
        x = random.gammavariate(self.alpha, 1.0)
        y = random.gammavariate(self.beta, 1.0)
        return x / (x + y) if (x + y) > 0 else 0.5

    @property
    def total_observations(self) -> float:
        return self.alpha + self.beta - 2  # Subtract initial pseudo-counts

    def to_dict(self) -> Dict[str, float]:
        return {"alpha": self.alpha, "beta": self.beta}


class BayesianToolSelector:
    """
    Thompson Sampling for tool selection.
    Maintains a Beta posterior for each tool's success probability.
    Naturally balances exploration (uncertain tools) and exploitation (known good tools).
    """

    def __init__(self):
        self._posteriors: Dict[str, BetaDistribution] = {}

    def register_tool(self, name: str, prior_alpha: float = 1.0,
                      prior_beta: float = 1.0) -> None:
        """Register a tool with optional informative prior."""
        self._posteriors[name] = BetaDistribution(prior_alpha, prior_beta)

    def record(self, name: str, success: bool) -> None:
        """Update posterior after observing outcome."""
        if name not in self._posteriors:
            self.register_tool(name)
        self._posteriors[name].update(success)

    def select(self, candidates: List[str]) -> str:
        """
        Thompson Sampling selection.
        Sample from each candidate's posterior, pick the highest.
        """
        if not candidates:
            return ""

        best_tool = candidates[0]
        best_sample = -1.0

        for tool in candidates:
            if tool not in self._posteriors:
                self.register_tool(tool)
            sample = self._posteriors[tool].sample()
            if sample > best_sample:
                best_sample = sample
                best_tool = tool

        return best_tool

    def get_posterior_summary(self, name: str) -> Dict[str, Any]:
        """Get posterior summary for a tool."""
        if name not in self._posteriors:
            return {"mean": 0.5, "std": 0.29, "observations": 0}
        post = self._posteriors[name]
        return {
            "mean": post.mean,
            "std": post.std,
            "ci_95": post.confidence_interval_95,
            "observations": post.total_observations,
            "alpha": post.alpha,
            "beta": post.beta,
        }

    def get_all_summaries(self) -> Dict[str, Dict[str, Any]]:
        return {name: self.get_posterior_summary(name)
                for name in self._posteriors}
```

**Integration Point**: Replace
`corteX/engine/weights.py` `ToolPreferenceWeights.get_best_tool()` with
Thompson Sampling selection. The `BetaDistribution` objects persist across
sessions alongside the weight file.

### 3.3 Bayesian Optimization: Hyperparameter Tuning

**Theoretical Background**

Bayesian optimization uses a Gaussian Process (GP) surrogate model to
efficiently optimize expensive black-box functions. Key concepts:

1. **Surrogate Model**: GP that models the objective function.
   - Mean function: best estimate of objective value at any point
   - Variance function: uncertainty about the objective at that point

2. **Acquisition Function**: Determines the next point to evaluate.
   - **Expected Improvement (EI)**:
     `EI(x) = E[max(f(x) - f(x_best), 0)]`
   - **Upper Confidence Bound (UCB)**:
     `UCB(x) = mu(x) + kappa * sigma(x)`
   - **Probability of Improvement (PI)**:
     `PI(x) = P(f(x) > f(x_best) + xi)`

**Application to corteX**

Hyperparameters that need tuning:
- Learning rates per weight category (currently hardcoded in `LearningRates`)
- Critical period length (currently `10` in `CriticalPeriodModulator`)
- Homeostatic regulation strength (currently `0.02`)
- EMA alpha values throughout the engine
- Outlier threshold in `PopulationDecoder` (currently `2.0`)
- Escalation thresholds for dual-process routing

**Concrete SDK Pattern: `BayesianHyperparameterTuner`**

```python
class SimpleBayesianOptimizer:
    """
    Lightweight Bayesian optimization for corteX hyperparameters.
    Uses a simplified surrogate model (no full GP dependency).
    Maintains observations and suggests next parameters to try.
    """

    def __init__(self, param_bounds: Dict[str, Tuple[float, float]]):
        self.param_bounds = param_bounds
        self._observations: List[Tuple[Dict[str, float], float]] = []
        self._best_params: Optional[Dict[str, float]] = None
        self._best_value: float = float('-inf')

    def suggest(self, n_random: int = 5) -> Dict[str, float]:
        """
        Suggest next parameters to try.
        Uses random search initially, then UCB-inspired suggestions.
        """
        import random

        if len(self._observations) < n_random:
            # Random exploration phase
            return {
                param: random.uniform(low, high)
                for param, (low, high) in self.param_bounds.items()
            }

        # UCB-inspired: perturb best known params with decreasing noise
        noise_scale = 1.0 / math.sqrt(len(self._observations))
        suggestion = {}
        for param, (low, high) in self.param_bounds.items():
            best_val = self._best_params[param]
            noise = random.gauss(0, (high - low) * noise_scale)
            suggestion[param] = max(low, min(high, best_val + noise))

        return suggestion

    def record(self, params: Dict[str, float], objective_value: float) -> None:
        """Record an observation (params -> objective value)."""
        self._observations.append((params, objective_value))
        if objective_value > self._best_value:
            self._best_value = objective_value
            self._best_params = params.copy()

    def get_best(self) -> Tuple[Optional[Dict[str, float]], float]:
        """Get best parameters found so far."""
        return self._best_params, self._best_value
```

**Integration Point**: New file `corteX/engine/tuning.py`. Runs during
`WeightEngine.consolidate()` to evaluate current hyperparameter performance
and suggest adjustments. Objective function = overall session success rate
or average quality score.

### 3.4 Conjugate Priors: The Right Distribution for Each Data Type

**Theoretical Background**

A conjugate prior is a prior distribution that, when combined with a particular
likelihood function, produces a posterior in the same distribution family.
This enables closed-form Bayesian updates without numerical integration.

| Data Type | Likelihood | Conjugate Prior | Posterior |
|---|---|---|---|
| Binary success/failure | Bernoulli | Beta(a, b) | Beta(a + s, b + f) |
| Count data | Poisson | Gamma(a, b) | Gamma(a + sum, b + n) |
| Latency (positive reals) | Exponential | Gamma(a, b) | Gamma(a + n, b + sum) |
| Quality scores (real) | Normal(mu, sigma_known) | Normal(m0, s0) | Normal(m_n, s_n) |
| Categorical choices | Multinomial | Dirichlet(a1...ak) | Dirichlet(a1+c1...ak+ck) |

**Concrete SDK Patterns for corteX**

```python
class GammaDistribution:
    """
    Gamma distribution for latency modeling.
    Conjugate prior for Exponential likelihood.
    Gamma(shape, rate) where:
      mean = shape / rate
      variance = shape / rate^2
    """

    def __init__(self, shape: float = 2.0, rate: float = 0.001):
        self.shape = shape  # alpha
        self.rate = rate    # beta

    def update(self, observed_latency_ms: float) -> None:
        """Update with an observed latency value."""
        self.shape += 1
        self.rate += observed_latency_ms

    @property
    def mean(self) -> float:
        """Expected latency."""
        return self.shape / self.rate if self.rate > 0 else float('inf')

    @property
    def variance(self) -> float:
        return self.shape / (self.rate ** 2) if self.rate > 0 else float('inf')

    def sample(self) -> float:
        """Sample a latency value."""
        return random.gammavariate(self.shape, 1.0 / self.rate)

    def predictive_probability(self, latency_ms: float) -> float:
        """P(next latency = x | data) -- how surprising is this latency?"""
        # Posterior predictive is a Lomax (Pareto Type II) distribution
        # Using simplified approximation
        expected = self.mean
        if expected <= 0:
            return 0.5
        ratio = latency_ms / expected
        return math.exp(-0.5 * (ratio - 1) ** 2)  # Gaussian approx


class NormalNormalUpdater:
    """
    Normal-Normal conjugate pair for quality score tracking.
    Prior: Normal(mu_0, sigma_0^2)
    Likelihood: Normal(mu, sigma_known^2)
    Posterior: Normal(mu_n, sigma_n^2) with closed-form updates.
    """

    def __init__(self, prior_mean: float = 0.5, prior_precision: float = 1.0,
                 known_noise_precision: float = 4.0):
        # Precision = 1/variance (working in precision space is cleaner)
        self.mu = prior_mean
        self.precision = prior_precision          # tau_0 = 1/sigma_0^2
        self.noise_precision = known_noise_precision  # tau = 1/sigma^2

    def update(self, observed_quality: float) -> None:
        """Update posterior with a quality observation."""
        new_precision = self.precision + self.noise_precision
        new_mu = ((self.precision * self.mu + self.noise_precision * observed_quality)
                  / new_precision)
        self.precision = new_precision
        self.mu = new_mu

    @property
    def mean(self) -> float:
        return self.mu

    @property
    def variance(self) -> float:
        return 1.0 / self.precision if self.precision > 0 else float('inf')

    @property
    def std(self) -> float:
        return math.sqrt(self.variance)

    def sample(self) -> float:
        """Sample from posterior."""
        return random.gauss(self.mu, self.std)

    @property
    def confidence_interval_95(self) -> Tuple[float, float]:
        return (self.mu - 1.96 * self.std, self.mu + 1.96 * self.std)


class DirichletMultinomialUpdater:
    """
    Dirichlet-Multinomial conjugate pair for categorical choice modeling.
    Useful for: model selection across task types, outcome type distributions.
    """

    def __init__(self, categories: List[str], prior_counts: Optional[List[float]] = None):
        self.categories = categories
        n = len(categories)
        self.alphas: Dict[str, float] = {}
        for i, cat in enumerate(categories):
            self.alphas[cat] = prior_counts[i] if prior_counts else 1.0

    def update(self, observed_category: str) -> None:
        """Update posterior with an observed category."""
        if observed_category in self.alphas:
            self.alphas[observed_category] += 1.0

    def get_probabilities(self) -> Dict[str, float]:
        """Get expected probability for each category."""
        total = sum(self.alphas.values())
        return {cat: a / total for cat, a in self.alphas.items()}

    def sample(self) -> Dict[str, float]:
        """Sample a probability vector from the Dirichlet posterior."""
        samples = {cat: random.gammavariate(a, 1.0)
                   for cat, a in self.alphas.items()}
        total = sum(samples.values())
        return {cat: s / total for cat, s in samples.items()} if total > 0 else self.get_probabilities()
```

**Integration Point**:
- `BetaDistribution`: `corteX/engine/weights.py`
  `ToolPreferenceWeights.success_rate` -> replace with Beta posterior
- `GammaDistribution`: `corteX/engine/weights.py`
  `ToolPreferenceWeights.avg_latency_ms` -> replace with Gamma posterior
- `NormalNormalUpdater`: `corteX/engine/prediction.py`
  `PredictionEngine._tool_stats["avg_quality"]` -> replace with Normal posterior
- `DirichletMultinomialUpdater`: `corteX/core/llm/router.py`
  `LLMRouter._model_weights` -> replace with Dirichlet posterior over task
  type affinities

### 3.5 Multi-Armed Bandits: UCB1 Alternative to Thompson Sampling

**Theoretical Background**

While Thompson Sampling is Bayesian, the Upper Confidence Bound (UCB1) algorithm
provides a frequentist alternative with strong regret guarantees:

```
UCB1 selection:
  Select arm j = argmax_j [ X_bar_j + sqrt(2 * ln(t) / n_j) ]

Where:
  X_bar_j = average reward of arm j
  t = total number of rounds so far
  n_j = number of times arm j has been played
```

The second term is the exploration bonus: it grows logarithmically with total
time (all arms become worth trying eventually) and decreases with the number
of times an arm has been played (well-explored arms need less exploration).

**Regret bound**: O(sqrt(K * T * ln(T))) where K = number of arms, T = time.

**Concrete SDK Pattern: `UCB1Selector`**

```python
class UCB1Selector:
    """
    UCB1 algorithm for model/tool selection.
    Deterministic alternative to Thompson Sampling.
    Better when you need reproducible selection decisions.
    """

    def __init__(self, exploration_weight: float = 2.0):
        self.c = exploration_weight  # Controls exploration-exploitation tradeoff
        self._avg_rewards: Dict[str, float] = {}
        self._counts: Dict[str, int] = {}
        self._total_rounds: int = 0

    def register(self, arm: str) -> None:
        if arm not in self._avg_rewards:
            self._avg_rewards[arm] = 0.0
            self._counts[arm] = 0

    def record(self, arm: str, reward: float) -> None:
        """Record reward for an arm."""
        self.register(arm)
        self._counts[arm] += 1
        self._total_rounds += 1
        # Incremental mean update
        n = self._counts[arm]
        old_avg = self._avg_rewards[arm]
        self._avg_rewards[arm] = old_avg + (reward - old_avg) / n

    def select(self, candidates: List[str]) -> str:
        """Select arm using UCB1."""
        if not candidates:
            return ""

        # First, play each arm at least once
        for arm in candidates:
            self.register(arm)
            if self._counts[arm] == 0:
                return arm

        # UCB1 formula
        best_arm = candidates[0]
        best_ucb = float('-inf')

        for arm in candidates:
            avg = self._avg_rewards[arm]
            n = self._counts[arm]
            t = self._total_rounds
            exploration_bonus = math.sqrt(self.c * math.log(t) / n)
            ucb_value = avg + exploration_bonus

            if ucb_value > best_ucb:
                best_ucb = ucb_value
                best_arm = arm

        return best_arm

    def get_stats(self) -> Dict[str, Dict[str, float]]:
        return {
            arm: {
                "avg_reward": self._avg_rewards[arm],
                "count": self._counts[arm],
                "ucb_bonus": (math.sqrt(self.c * math.log(max(1, self._total_rounds))
                              / max(1, self._counts[arm]))),
            }
            for arm in self._avg_rewards
        }
```

**Integration Point**: `corteX/core/llm/router.py` `LLMRouter._select_model()`
-- can be used as alternative to Thompson Sampling when deterministic behavior
is preferred (e.g., in testing or when `enterprise.audit_logging = True`).

### 3.6 Bayesian Surprise: Formalizing the Prediction Error Signal

**Theoretical Background**

Bayesian surprise (Itti & Baldi, 2009) quantifies how much an observation
changes beliefs, measured as the KL divergence between prior and posterior:

```
Surprise(data) = D_KL(posterior || prior)
               = integral [ posterior(theta) * log(posterior(theta) / prior(theta)) ] d_theta
```

For conjugate distributions, KL divergence has closed-form solutions:

**Beta-Beta KL divergence** (for success rate surprise):
```
D_KL(Beta(a1,b1) || Beta(a2,b2)) =
    log(B(a2,b2)/B(a1,b1))
    + (a1-a2)*psi(a1) + (b1-b2)*psi(b1)
    + (a2-a1+b2-b1)*psi(a1+b1)
```
Where `psi` is the digamma function and `B` is the beta function.

**Normal-Normal KL divergence** (for quality score surprise):
```
D_KL(N(mu1,s1^2) || N(mu2,s2^2)) =
    log(s2/s1) + (s1^2 + (mu1-mu2)^2)/(2*s2^2) - 0.5
```

**Application to corteX**

The current `PredictionEngine.compare()` uses ad-hoc heuristics to compute
surprise. Bayesian surprise provides a principled replacement.

**Concrete SDK Pattern: `BayesianSurpriseCalculator`**

```python
class BayesianSurpriseCalculator:
    """
    Computes Bayesian surprise as KL divergence between
    prior and posterior distributions.

    This replaces the heuristic surprise computation in PredictionEngine
    with a principled information-theoretic measure.
    """

    @staticmethod
    def beta_kl_divergence(post_alpha: float, post_beta: float,
                           prior_alpha: float, prior_beta: float) -> float:
        """
        KL divergence between two Beta distributions.
        D_KL(Beta(post) || Beta(prior))
        Measures how much the posterior differs from the prior.
        """
        from math import lgamma
        # Using log-gamma for numerical stability
        def log_beta(a, b):
            return lgamma(a) + lgamma(b) - lgamma(a + b)

        def digamma_approx(x):
            """Stirling's approximation for digamma function."""
            # psi(x) approx ln(x) - 1/(2x) for large x
            if x > 6:
                return math.log(x) - 1.0 / (2 * x) - 1.0 / (12 * x * x)
            # Recursion for small x: psi(x) = psi(x+1) - 1/x
            if x <= 0:
                return 0.0
            return digamma_approx(x + 1) - 1.0 / x

        kl = (log_beta(prior_alpha, prior_beta)
              - log_beta(post_alpha, post_beta))
        kl += ((post_alpha - prior_alpha) * digamma_approx(post_alpha)
               + (post_beta - prior_beta) * digamma_approx(post_beta))
        kl += ((prior_alpha - post_alpha + prior_beta - post_beta)
               * digamma_approx(post_alpha + post_beta))

        return max(0.0, kl)  # KL is non-negative

    @staticmethod
    def normal_kl_divergence(post_mu: float, post_var: float,
                             prior_mu: float, prior_var: float) -> float:
        """
        KL divergence between two Normal distributions.
        D_KL(N(post_mu, post_var) || N(prior_mu, prior_var))
        """
        if prior_var <= 0 or post_var <= 0:
            return 0.0
        return (math.log(prior_var / post_var) / 2
                + (post_var + (post_mu - prior_mu) ** 2) / (2 * prior_var)
                - 0.5)

    def compute_surprise(
        self,
        prior_params: Dict[str, float],
        posterior_params: Dict[str, float],
        distribution_type: str = "beta",
    ) -> float:
        """
        Compute Bayesian surprise for a single observation.
        Returns KL divergence in nats (natural log units).
        """
        if distribution_type == "beta":
            return self.beta_kl_divergence(
                posterior_params["alpha"], posterior_params["beta"],
                prior_params["alpha"], prior_params["beta"],
            )
        elif distribution_type == "normal":
            return self.normal_kl_divergence(
                posterior_params["mu"], posterior_params["var"],
                prior_params["mu"], prior_params["var"],
            )
        return 0.0

    def surprise_to_learning_signal(self, surprise: float,
                                     max_surprise: float = 5.0) -> float:
        """
        Convert raw surprise (KL divergence) to a learning signal [0, 1].
        Uses sigmoid-like scaling so extreme surprises don't cause instability.
        """
        normalized = surprise / max_surprise
        return math.tanh(normalized)  # Bounded [0, 1) for positive KL
```

**Integration Point**: `corteX/engine/prediction.py`
`PredictionEngine.compare()` -- replace the heuristic surprise computation
with `BayesianSurpriseCalculator`. Before each observation, snapshot the
prior parameters. After updating, compute KL divergence between prior
and posterior. This feeds directly into `corteX/engine/plasticity.py`
`PlasticityManager.on_step_complete()` through the `SurpriseSignal`.

---

## Integration Map

### Where Each Pattern Fits in the corteX Codebase

```
corteX/engine/
  |
  +-- weights.py          <-- ProspectTheoreticUpdater (1.2)
  |                            AnchorManager (1.3)
  |                            AvailabilityFilter (1.4)
  |                            BayesianToolSelector (3.2)
  |                            BetaDistribution (3.4)
  |                            GammaDistribution (3.4)
  |
  +-- prediction.py       <-- BayesianSurpriseCalculator (3.6)
  |                            NormalNormalUpdater (3.4)
  |
  +-- plasticity.py       <-- ShapleyAttributor (2.4) for credit assignment
  |                            ReputationSystem (2.5) for trust-based LTP/LTD
  |
  +-- population.py       <-- FrameNormalizer (1.5) for score normalization
  |
  +-- dual_process.py     <-- DualProcessRouter (1.1)       [NEW FILE]
  |
  +-- bayesian.py         <-- BetaDistribution              [NEW FILE]
  |                            GammaDistribution
  |                            NormalNormalUpdater
  |                            DirichletMultinomialUpdater
  |                            BayesianSurpriseCalculator
  |
  +-- game_theory.py      <-- NashRoutingOptimizer (2.1)    [NEW FILE]
  |                            MinimaxSafetyGuard (2.2)
  |                            TruthfulScoringMechanism (2.3)
  |                            ShapleyAttributor (2.4)
  |                            ReputationSystem (2.5)
  |
  +-- tuning.py           <-- SimpleBayesianOptimizer (3.3) [NEW FILE]
  |
  +-- bandits.py          <-- UCB1Selector (3.5)            [NEW FILE]
  |                            BayesianToolSelector (3.2)

corteX/core/llm/
  +-- router.py           <-- NashRoutingOptimizer integration
  |                            UCB1Selector / ThompsonSampling for model selection
  |                            DirichletMultinomialUpdater for task-model routing

corteX/runtime/
  +-- orchestrator.py     <-- DualProcessRouter integration
  |                            MinimaxSafetyGuard for enterprise decisions
```

### Data Flow

```
User Request
    |
    v
DualProcessRouter (1.1)  -- Decide fast/slow path
    |
    +--[System 1]--> PopulationDecoder + FrameNormalizer (1.5)
    |                    |
    |                    v
    |                BayesianToolSelector (3.2) or UCB1Selector (3.5)
    |                    |
    |                    v
    |                Tool Execution
    |
    +--[System 2]--> LLMRouter with NashRoutingOptimizer (2.1)
                         |
                         v
                     Full LLM Reasoning (thinking=True)
                         |
                         v
                     Tool Execution
    |
    v
Outcome Observation
    |
    +-- BetaDistribution.update() (3.4) -- success/failure posterior
    +-- GammaDistribution.update() (3.4) -- latency posterior
    +-- NormalNormalUpdater.update() (3.4) -- quality posterior
    +-- ProspectTheoreticUpdater (1.2) -- loss-aversive weight update
    +-- ReputationSystem.record() (2.5) -- trust update
    +-- BayesianSurpriseCalculator (3.6) -- compute surprise signal
    |
    v
PlasticityManager
    |
    +-- HebbianRule with ShapleyAttributor (2.4)
    +-- LTP/LTD with ReputationSystem (2.5)
    +-- HomeostaticRegulation
    |
    v
Consolidation (session end)
    |
    +-- NashRoutingOptimizer.iterate() (2.1)
    +-- SimpleBayesianOptimizer.suggest() (3.3) -- tuning
    +-- TruthfulScoringMechanism.credibility_score() (2.3)
    +-- WeightEngine.consolidate()
```

---

## Priority Ranking

### Impact vs. Complexity Matrix

| # | Pattern | Impact | Complexity | Priority | Rationale |
|---|---------|--------|------------|----------|-----------|
| 1 | **BetaDistribution + Thompson Sampling** (3.2, 3.4) | Very High | Low | P0 | Direct replacement for EMA. Adds exploration, uncertainty. Minimal code change. |
| 2 | **ProspectTheoreticUpdater** (1.2) | High | Low | P0 | Simple formula change in weight updates. Immediately improves failure handling. |
| 3 | **BayesianSurpriseCalculator** (3.6) | High | Medium | P1 | Replaces heuristic surprise with principled KL divergence. Requires posterior tracking. |
| 4 | **DualProcessRouter** (1.1) | High | Medium | P1 | Reduces unnecessary LLM calls. Requires integration with orchestrator. |
| 5 | **ReputationSystem** (2.5) | High | Low | P1 | Adds trust dynamics and quarantine. Simple data structure addition. |
| 6 | **UCB1Selector** (3.5) | Medium | Low | P1 | Deterministic alternative to Thompson. Good for enterprise/audit. |
| 7 | **GammaDistribution** (3.4) | Medium | Low | P2 | Better latency modeling. Drop-in replacement for EMA. |
| 8 | **AvailabilityFilter** (1.4) | Medium | Low | P2 | Controlled recency bias. Integrates with existing adaptation. |
| 9 | **MinimaxSafetyGuard** (2.2) | Medium | Low | P2 | Enterprise safety feature. Simple threshold logic. |
| 10 | **AnchorManager** (1.3) | Medium | Low | P2 | Better initialization. Needs global weight data to be useful. |
| 11 | **NashRoutingOptimizer** (2.1) | Medium | Medium | P3 | Sophisticated but requires multi-model setup to be useful. |
| 12 | **ShapleyAttributor** (2.4) | Medium | High | P3 | Fair attribution. O(2^N) complexity, needs Monte Carlo for scale. |
| 13 | **TruthfulScoringMechanism** (2.3) | Low-Med | Medium | P3 | Mechanism design. Value increases with more tools in ecosystem. |
| 14 | **FrameNormalizer** (1.5) | Low | Low | P3 | Nice-to-have normalization. Easy to add when needed. |
| 15 | **SimpleBayesianOptimizer** (3.3) | High | High | P4 | Hyperparameter tuning. Needs significant data/sessions. Long-term value. |
| 16 | **NormalNormalUpdater** (3.4) | Medium | Low | P2 | Quality score modeling. Drop-in replacement. |
| 17 | **DirichletMultinomialUpdater** (3.4) | Medium | Medium | P3 | Task-type distribution modeling. Needs multi-model routing. |

### Recommended Implementation Order

**Phase 1 (P0) -- Foundation**: Beta posteriors + Thompson Sampling + Prospect Theory
- Replace EMA with `BetaDistribution` for success rates
- Add `ProspectTheoreticUpdater` for asymmetric updates
- Add `BayesianToolSelector` using Thompson Sampling
- Estimated effort: 2-3 days, ~200 lines of new code

**Phase 2 (P1) -- Intelligence**: Bayesian Surprise + Dual Process + Reputation
- Add `BayesianSurpriseCalculator` replacing heuristic surprise
- Add `DualProcessRouter` for System 1/2 routing
- Add `ReputationSystem` for trust dynamics
- Add `UCB1Selector` as deterministic alternative
- Estimated effort: 3-5 days, ~400 lines of new code

**Phase 3 (P2) -- Refinement**: Latency modeling + Safety + Availability
- Add `GammaDistribution` for latency posteriors
- Add `NormalNormalUpdater` for quality posteriors
- Add `MinimaxSafetyGuard` for enterprise decisions
- Add `AvailabilityFilter` for controlled recency bias
- Add `AnchorManager` for informed initialization
- Estimated effort: 3-4 days, ~350 lines of new code

**Phase 4 (P3) -- Ecosystem**: Game Theory + Attribution
- Add `NashRoutingOptimizer` for multi-model routing
- Add `ShapleyAttributor` for credit assignment
- Add `TruthfulScoringMechanism` for honest capability reporting
- Add `DirichletMultinomialUpdater` for task-type modeling
- Add `FrameNormalizer` for score normalization
- Estimated effort: 5-7 days, ~500 lines of new code

**Phase 5 (P4) -- Meta-optimization**: Bayesian Hyperparameter Tuning
- Add `SimpleBayesianOptimizer` for hyperparameter tuning
- Requires accumulated session data for meaningful optimization
- Estimated effort: 3-4 days, ~200 lines of new code

---

## Mathematical Reference: Key Formulas

### Prospect Theory Value Function
```
v(x) = x^0.88                    if x >= 0 (gain)
v(x) = -2.25 * |x|^0.88         if x < 0  (loss)
```

### Beta-Bernoulli Posterior Update
```
Prior: Beta(alpha, beta)
Observation: success (s=1) or failure (s=0)
Posterior: Beta(alpha + s, beta + 1 - s)
Mean: alpha / (alpha + beta)
```

### Thompson Sampling Selection
```
For each tool i:
  theta_i ~ Beta(alpha_i, beta_i)    # Sample from posterior
Select tool j = argmax_i theta_i     # Pick highest sample
```

### UCB1 Selection
```
Select arm j = argmax_j [ X_bar_j + sqrt(2 * ln(t) / n_j) ]
```

### Bayesian Surprise (KL Divergence)
```
Surprise = D_KL(posterior || prior)
For Beta: D_KL(Beta(a1,b1) || Beta(a2,b2))
For Normal: D_KL(N(mu1,s1^2) || N(mu2,s2^2)) = log(s2/s1) + (s1^2 + (mu1-mu2)^2)/(2*s2^2) - 0.5
```

### Shapley Value
```
phi_i = sum_{S subset N\{i}} [|S|!(|N|-|S|-1)! / |N|!] * [v(S u {i}) - v(S)]
```

### Nash Equilibrium Best Response
```
BR_i(sigma_{-i}) = argmax_{sigma_i} U_i(sigma_i, sigma_{-i})
NE: sigma_i = BR_i(sigma_{-i}) for all i
```

### Expected Improvement (Bayesian Optimization)
```
EI(x) = (mu(x) - f_best) * Phi(Z) + sigma(x) * phi(Z)
Z = (mu(x) - f_best) / sigma(x)
```

### Gamma-Exponential Posterior Update
```
Prior: Gamma(shape, rate)
Observation: latency x_i (Exponential distributed)
Posterior: Gamma(shape + 1, rate + x_i)
Expected latency: shape / rate
```

---

## References

### Kahneman's Decision Theory
- Kahneman, D. (2011). *Thinking, Fast and Slow*. Farrar, Straus and Giroux.
- Kahneman, D. & Tversky, A. (1979). "Prospect Theory: An Analysis of Decision under Risk." *Econometrica*, 47(2), 263-291.
- Tversky, A. & Kahneman, D. (1992). "Advances in prospect theory: Cumulative representation of uncertainty." *Journal of Risk and Uncertainty*, 5(4), 297-323.
- Tversky, A. & Kahneman, D. (1974). "Judgment under Uncertainty: Heuristics and Biases." *Science*, 185(4157), 1124-1131.

### Game Theory
- Nash, J. (1950). "Equilibrium Points in N-person Games." *Proceedings of the National Academy of Sciences*, 36(1), 48-49.
- Von Neumann, J. & Morgenstern, O. (1944). *Theory of Games and Economic Behavior*. Princeton University Press.
- Shapley, L. S. (1953). "A Value for N-person Games." *Contributions to the Theory of Games*, 2, 307-317.
- Axelrod, R. (1984). *The Evolution of Cooperation*. Basic Books.
- Myerson, R. B. (1981). "Optimal Auction Design." *Mathematics of Operations Research*, 6(1), 58-73.
- Harsanyi, J. C. (1967-68). "Games with Incomplete Information Played by Bayesian Players." *Management Science*, Parts I-III.

### Bayesian Mathematics
- Thompson, W. R. (1933). "On the Likelihood that One Unknown Probability Exceeds Another." *Biometrika*, 25(3-4), 285-294.
- Russo, D. et al. (2018). "A Tutorial on Thompson Sampling." *Foundations and Trends in Machine Learning*, 11(1), 1-96.
- Auer, P., Cesa-Bianchi, N. & Fischer, P. (2002). "Finite-time Analysis of the Multiarmed Bandit Problem." *Machine Learning*, 47(2), 235-256.
- Itti, L. & Baldi, P. (2009). "Bayesian Surprise Attracts Human Attention." *Vision Research*, 49(10), 1295-1306.

### Computational Neuroscience
- Friston, K. (2010). "The Free-Energy Principle: A Unified Brain Theory?" *Nature Reviews Neuroscience*, 11(2), 127-138.
- Doya, K. (2007). *Bayesian Brain: Probabilistic Approaches to Neural Coding*. MIT Press.

### Applied Research (2025-2026)
- De La Fuente, N. et al. (2024). "Game Theory and Multi-Agent Reinforcement Learning: From Nash Equilibria to Evolutionary Dynamics." arXiv:2412.20523.
- "Game-Theoretic Lens on LLM-based Multi-Agent Systems." arXiv:2601.15047 (2026).
- "Learning, Reasoning, Refinement: A Framework for Kahneman's Dual-System Intelligence in GUI Agents." arXiv:2506.17913 (2025).

---

*This research document was prepared for the corteX project. Complexity creates intelligence -- these mathematical foundations provide the rigorous backbone for the brain-inspired weight engine to evolve from heuristic-driven to principled probabilistic reasoning.*
