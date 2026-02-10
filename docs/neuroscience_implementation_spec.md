# Neuroscience Pattern Implementation Specification

## corteX Brain-Inspired AI Agent SDK - Remaining Patterns

**Status**: Implementation Specification
**Author**: Research Agent
**Date**: 2026-02-09
**Source Material**: Prof. Idan Segev, Hebrew University - "Mashav Moach: From Synapses to Free Will"
**Codebase Version**: Current engine/ modules (weights, prediction, plasticity, feedback, adaptation, population, goal_tracker, memory)

---

## Table of Contents

1. [P0: Proactive Prediction](#p0-proactive-prediction)
2. [P1: Cross-Modal Association](#p1-cross-modal-association)
3. [P1: Continuous Calibration](#p1-continuous-calibration)
4. [P2: Functional Columns (High-Level)](#p2-functional-columns)
5. [P2: Attentional Filter (High-Level)](#p2-attentional-filter)
6. [Cross-Pattern Interactions](#cross-pattern-interactions)
7. [New Brain Pattern Proposals](#new-brain-pattern-proposals)

---

## P0: Proactive Prediction

### Neuroscience Basis

> "The goalkeeper imagines what will happen in the next moment. He fantasizes about the future.
> He imagines what's going to happen. This machine constantly imagines. Sometimes it succeeds
> in imagining well, and if not, it updates and works like this constantly - it predicts, and
> updates when its prediction doesn't match reality." - Prof. Idan Segev, Lecture 4

The current `PredictionEngine` in `corteX/engine/prediction.py` is **reactive**: it predicts
the outcome of an action *about to be taken*, then compares after execution. This is analogous
to the cerebellum predicting the sensory consequences of a motor command.

But the cortex does something far more powerful: it predicts **what stimulus will arrive next**,
before any action is even planned. The goalkeeper does not wait for the ball to be kicked; he
predicts the trajectory *before* the kick occurs. This is proactive prediction -- forecasting
the next user message, the next task type, the next tool needed, *before the user sends anything*.

### Architecture Overview

```
ProactivePredictionEngine
  |
  +-- ConversationTrajectoryModel (n-gram + recency-weighted Markov chain)
  |
  +-- PredictionChainCache (sequence pattern store: A->B->? = C)
  |
  +-- PreWarmingScheduler (async background pre-loading of tools/context/models)
  |
  +-- IntegrationBridge (connects to existing PredictionEngine for unified API)
```

### File: `corteX/engine/prediction.py` (Enhancement)

All new code extends the existing `prediction.py` file. The existing `PredictionEngine` class
is preserved. A new `ProactivePredictionEngine` class wraps it and adds proactive capabilities.

### Data Structures

```python
@dataclass
class ConversationTurn:
    """A single turn in the conversation, featurized for prediction."""
    turn_id: str
    task_type: str              # coding, debugging, research, conversation, etc.
    tools_used: List[str]
    model_used: str
    topic_hash: str             # Coarse hash of topic for sequence matching
    complexity: float           # 0.0 to 1.0
    user_message_length: int
    timestamp: float = field(default_factory=time.time)


@dataclass
class NextTurnPrediction:
    """Prediction of what the user will ask next."""
    predicted_task_type: str
    confidence: float                   # 0.0 to 1.0
    predicted_tools: List[str]          # Tools likely needed
    predicted_model: str                # Best model for predicted task
    predicted_complexity: float         # Expected complexity
    prediction_basis: str               # Why this prediction (chain match, frequency, etc.)
    pre_warm_actions: List[str]         # Specific pre-warming actions to take
    timestamp: float = field(default_factory=time.time)
    chain_id: Optional[str] = None      # If matched a known chain


@dataclass
class PredictionChain:
    """A learned sequence pattern: A -> B -> C."""
    chain_id: str
    sequence: List[str]                 # Sequence of task_type hashes
    next_predicted: str                 # The predicted next element
    occurrences: int                    # How many times this chain was observed
    success_rate: float                 # How often the prediction was correct
    last_seen: float = field(default_factory=time.time)
    decay_factor: float = 1.0          # Recency weighting (decays over time)
```

### Class: ConversationTrajectoryModel

```python
class ConversationTrajectoryModel:
    """
    Models conversation as a Markov chain with variable-length memory.
    Uses n-gram analysis (n=1,2,3) to predict the next turn's task type.

    The model maintains transition probabilities:
      P(task_type_next | task_type_current) -- unigram
      P(task_type_next | task_type_prev, task_type_current) -- bigram
      P(task_type_next | t_prev2, t_prev1, t_current) -- trigram

    Higher-order n-grams are weighted more when they have sufficient data,
    falling back to lower-order when sparse.
    """

    def __init__(
        self,
        max_history: int = 200,
        unigram_weight: float = 0.2,
        bigram_weight: float = 0.5,
        trigram_weight: float = 0.3,
        recency_decay: float = 0.95,     # Weight of each older turn
    ):
        self._history: List[ConversationTurn] = []
        self._max_history = max_history
        self._weights = (unigram_weight, bigram_weight, trigram_weight)
        self._recency_decay = recency_decay

        # Transition counts: key = (prev_states_tuple,) -> {next_state: count}
        self._unigram_transitions: Dict[str, Dict[str, float]] = defaultdict(lambda: defaultdict(float))
        self._bigram_transitions: Dict[Tuple[str, str], Dict[str, float]] = defaultdict(lambda: defaultdict(float))
        self._trigram_transitions: Dict[Tuple[str, str, str], Dict[str, float]] = defaultdict(lambda: defaultdict(float))

    def record_turn(self, turn: ConversationTurn) -> None:
        """Record a completed turn and update transition probabilities."""
        self._history.append(turn)
        if len(self._history) > self._max_history:
            self._history = self._history[-self._max_history:]

        n = len(self._history)
        task = turn.task_type

        # Update unigram: P(next | current)
        if n >= 2:
            prev = self._history[-2].task_type
            self._unigram_transitions[prev][task] += 1.0

        # Update bigram: P(next | prev, current)
        if n >= 3:
            prev2 = self._history[-3].task_type
            prev1 = self._history[-2].task_type
            self._bigram_transitions[(prev2, prev1)][task] += 1.0

        # Update trigram: P(next | prev2, prev1, current)
        if n >= 4:
            prev3 = self._history[-4].task_type
            prev2 = self._history[-3].task_type
            prev1 = self._history[-2].task_type
            self._trigram_transitions[(prev3, prev2, prev1)][task] += 1.0

    def predict_next(self) -> List[Tuple[str, float]]:
        """
        Predict the next task type with probabilities.
        Returns sorted list of (task_type, probability) pairs.

        Mathematical model:
          P(next) = w1 * P_unigram(next) + w2 * P_bigram(next) + w3 * P_trigram(next)

        Where each P_n is normalized from raw counts with recency weighting.
        If an n-gram has no data, its weight is redistributed to lower-order models.
        """
        if not self._history:
            return [("conversation", 0.5)]

        current = self._history[-1].task_type
        w1, w2, w3 = self._weights

        # Unigram prediction
        uni_probs = self._normalize_counts(self._unigram_transitions.get(current, {}))

        # Bigram prediction
        bi_probs = {}
        if len(self._history) >= 2:
            prev = self._history[-2].task_type
            bi_probs = self._normalize_counts(
                self._bigram_transitions.get((prev, current), {})
            )

        # Trigram prediction
        tri_probs = {}
        if len(self._history) >= 3:
            prev2 = self._history[-3].task_type
            prev1 = self._history[-2].task_type
            tri_probs = self._normalize_counts(
                self._trigram_transitions.get((prev2, prev1, current), {})
            )

        # Redistribute weights if higher-order n-grams are empty
        effective_w1, effective_w2, effective_w3 = w1, w2, w3
        if not tri_probs:
            effective_w2 += effective_w3
            effective_w3 = 0.0
        if not bi_probs:
            effective_w1 += effective_w2
            effective_w2 = 0.0

        total_weight = effective_w1 + effective_w2 + effective_w3
        if total_weight == 0:
            return [("conversation", 0.5)]

        # Combine
        all_types = set(uni_probs.keys()) | set(bi_probs.keys()) | set(tri_probs.keys())
        combined: Dict[str, float] = {}
        for t in all_types:
            combined[t] = (
                effective_w1 * uni_probs.get(t, 0.0)
                + effective_w2 * bi_probs.get(t, 0.0)
                + effective_w3 * tri_probs.get(t, 0.0)
            ) / total_weight

        # Sort by probability descending
        ranked = sorted(combined.items(), key=lambda x: x[1], reverse=True)
        return ranked

    def _normalize_counts(self, counts: Dict[str, float]) -> Dict[str, float]:
        """Normalize raw counts to probabilities with Laplace smoothing."""
        if not counts:
            return {}
        total = sum(counts.values()) + len(counts) * 0.1  # Laplace smoothing
        return {k: (v + 0.1) / total for k, v in counts.items()}

    def get_trajectory_entropy(self) -> float:
        """
        Compute the entropy of the predicted distribution.
        Low entropy = high confidence (predictable user).
        High entropy = uncertain (unpredictable user).

        H = -sum(p * log2(p)) for all predicted task types
        """
        predictions = self.predict_next()
        if not predictions:
            return 1.0
        entropy = 0.0
        for _, prob in predictions:
            if prob > 0:
                entropy -= prob * math.log2(prob)
        return entropy
```

### Class: PredictionChainCache

```python
class PredictionChainCache:
    """
    Stores and matches learned conversation chains.
    A chain is a sequence like: [coding, debugging, testing] -> documentation.

    If the user's last 3 turns were coding -> debugging -> testing,
    and this chain has been seen 5 times before with documentation
    following 4/5 times, then predict "documentation" with 80% confidence.

    Chain matching uses variable-length prefix matching:
    - Try to match the longest chain first (most specific)
    - Fall back to shorter chains if no long match

    Mathematical model:
      confidence(chain) = occurrences / (occurrences + k) * success_rate * decay_factor
      where k = 3 (pseudocount for Bayesian smoothing)
    """

    def __init__(
        self,
        max_chain_length: int = 5,
        min_occurrences: int = 2,
        max_chains: int = 500,
        decay_halflife_hours: float = 24.0,
    ):
        self._chains: Dict[str, PredictionChain] = {}
        self._max_length = max_chain_length
        self._min_occurrences = min_occurrences
        self._max_chains = max_chains
        self._decay_halflife = decay_halflife_hours * 3600  # Convert to seconds

    def record_sequence(self, task_types: List[str]) -> None:
        """
        Record a completed sequence and extract all sub-chains.
        For [A, B, C, D], extracts:
          [A] -> B, [A,B] -> C, [A,B,C] -> D
          [B] -> C, [B,C] -> D
          [C] -> D
        """
        for length in range(1, min(len(task_types), self._max_length + 1)):
            for start in range(len(task_types) - length):
                prefix = tuple(task_types[start:start + length])
                next_item = task_types[start + length]
                chain_key = f"{':'.join(prefix)}->{next_item}"

                if chain_key in self._chains:
                    chain = self._chains[chain_key]
                    chain.occurrences += 1
                    chain.last_seen = time.time()
                else:
                    self._chains[chain_key] = PredictionChain(
                        chain_id=chain_key,
                        sequence=list(prefix),
                        next_predicted=next_item,
                        occurrences=1,
                        success_rate=0.5,
                        last_seen=time.time(),
                    )

        # Prune if over capacity
        if len(self._chains) > self._max_chains:
            self._prune()

    def match(self, recent_types: List[str]) -> Optional[Tuple[PredictionChain, float]]:
        """
        Find the best matching chain for the recent task sequence.
        Tries longest prefix first, then shorter.

        Returns (chain, confidence) or None if no match.
        """
        now = time.time()
        best_match: Optional[Tuple[PredictionChain, float]] = None
        best_score = 0.0

        # Try progressively shorter prefixes
        for length in range(min(len(recent_types), self._max_length), 0, -1):
            prefix = tuple(recent_types[-length:])

            # Find all chains matching this prefix
            for chain in self._chains.values():
                if tuple(chain.sequence) == prefix and chain.occurrences >= self._min_occurrences:
                    # Compute confidence with time decay
                    age_seconds = now - chain.last_seen
                    decay = math.exp(-0.693 * age_seconds / self._decay_halflife)
                    chain.decay_factor = decay

                    # Bayesian smoothed confidence
                    k = 3  # Pseudocount
                    confidence = (
                        chain.occurrences / (chain.occurrences + k)
                        * chain.success_rate
                        * decay
                    )

                    # Longer chains get a bonus (more specific = more trustworthy)
                    length_bonus = 1.0 + 0.1 * length
                    score = confidence * length_bonus

                    if score > best_score:
                        best_score = score
                        best_match = (chain, confidence)

        return best_match

    def update_accuracy(self, chain_id: str, was_correct: bool) -> None:
        """Update the accuracy of a chain prediction after verification."""
        chain = self._chains.get(chain_id)
        if chain:
            alpha = 0.2  # EMA factor
            actual = 1.0 if was_correct else 0.0
            chain.success_rate = chain.success_rate * (1 - alpha) + actual * alpha

    def _prune(self) -> None:
        """Remove least useful chains (low occurrence, old, low success)."""
        scored = []
        now = time.time()
        for key, chain in self._chains.items():
            age = now - chain.last_seen
            score = chain.occurrences * chain.success_rate * math.exp(-age / self._decay_halflife)
            scored.append((key, score))
        scored.sort(key=lambda x: x[1])
        # Remove bottom 20%
        to_remove = len(scored) // 5
        for key, _ in scored[:to_remove]:
            del self._chains[key]
```

### Class: PreWarmingScheduler

```python
class PreWarmingScheduler:
    """
    Executes pre-warming actions based on proactive predictions.

    Pre-warming actions include:
    - Pre-selecting the model (avoiding cold-start latency)
    - Pre-loading tool definitions for predicted tools
    - Pre-fetching relevant memory context
    - Pre-building system prompts for predicted task types

    All pre-warming is advisory: if the prediction is wrong,
    the pre-warmed resources are simply discarded.

    Brain analogy: Motor cortex preparing a movement before
    the conscious decision to move. Readiness potential (Bereitschaftspotential)
    precedes conscious awareness of the decision by ~500ms.
    """

    def __init__(self, memory: Optional[MemoryFabric] = None):
        self._memory = memory
        self._pre_warmed: Dict[str, Any] = {}
        self._warming_in_progress: bool = False

    async def pre_warm(self, prediction: NextTurnPrediction) -> Dict[str, Any]:
        """
        Execute pre-warming based on a prediction.
        Returns a dict of pre-warmed resources.
        """
        self._warming_in_progress = True
        warmed: Dict[str, Any] = {}

        try:
            # 1. Pre-select model
            warmed["predicted_model"] = prediction.predicted_model
            warmed["predicted_task_type"] = prediction.predicted_task_type

            # 2. Pre-fetch memory context for predicted topic
            if self._memory:
                context = self._memory.get_relevant_context(
                    prediction.predicted_task_type, max_items=5
                )
                warmed["pre_fetched_context"] = context

            # 3. Record predicted tools for fast loading
            warmed["predicted_tools"] = prediction.predicted_tools

            # 4. Record confidence for downstream use
            warmed["confidence"] = prediction.confidence

            self._pre_warmed = warmed

        finally:
            self._warming_in_progress = False

        return warmed

    def get_pre_warmed(self) -> Dict[str, Any]:
        """Retrieve pre-warmed resources. Returns empty if nothing pre-warmed."""
        return dict(self._pre_warmed)

    def invalidate(self) -> None:
        """Invalidate all pre-warmed resources (prediction was wrong)."""
        self._pre_warmed.clear()

    def was_prediction_useful(self, actual_task_type: str) -> bool:
        """Check if the pre-warmed prediction matched what actually happened."""
        predicted = self._pre_warmed.get("predicted_task_type", "")
        return predicted == actual_task_type
```

### Class: ProactivePredictionEngine

```python
class ProactivePredictionEngine:
    """
    Main proactive prediction engine. Wraps the existing PredictionEngine
    and adds conversation-level prediction capabilities.

    The existing PredictionEngine handles: predict action outcome -> compare -> surprise.
    This engine handles: predict NEXT USER TURN -> pre-warm -> verify after turn.

    Integration:
    - Called asynchronously between user turns (not blocking the response)
    - Results stored in working memory for the orchestrator to consume
    - Feeds into AdaptationFilter: novel predictions get high attention,
      habituated predictions (same pattern repeated) get less attention

    Usage:
        proactive = ProactivePredictionEngine(memory=fabric)

        # After each turn completes:
        proactive.record_turn(ConversationTurn(...))
        next_prediction = proactive.predict_next_turn()

        # Pre-warm in background:
        await proactive.schedule_pre_warming(next_prediction)

        # When next turn arrives, verify:
        was_correct = proactive.verify_prediction(actual_task_type)
    """

    def __init__(
        self,
        memory: Optional[MemoryFabric] = None,
        weight_engine: Optional[WeightEngine] = None,
    ):
        self._trajectory = ConversationTrajectoryModel()
        self._chain_cache = PredictionChainCache()
        self._pre_warmer = PreWarmingScheduler(memory=memory)
        self._weight_engine = weight_engine
        self._memory = memory

        # History of proactive predictions for accuracy tracking
        self._prediction_history: List[Tuple[NextTurnPrediction, Optional[bool]]] = []
        self._accuracy_window: int = 50

        # Current pending prediction
        self._current_prediction: Optional[NextTurnPrediction] = None

        # Task type -> best model mapping (learned)
        self._task_model_map: Dict[str, str] = {}

        # Task type -> common tools mapping (learned)
        self._task_tool_map: Dict[str, List[str]] = {}

    def record_turn(self, turn: ConversationTurn) -> None:
        """Record a completed conversation turn for trajectory modeling."""
        self._trajectory.record_turn(turn)

        # Update task -> model mapping
        if turn.model_used:
            self._task_model_map[turn.task_type] = turn.model_used

        # Update task -> tools mapping
        if turn.tools_used:
            existing = self._task_tool_map.get(turn.task_type, [])
            for tool in turn.tools_used:
                if tool not in existing:
                    existing.append(tool)
            self._task_tool_map[turn.task_type] = existing[-10:]  # Keep recent 10

        # Record sequence for chain learning
        recent_types = [t.task_type for t in self._trajectory._history[-6:]]
        if len(recent_types) >= 2:
            self._chain_cache.record_sequence(recent_types)

        # Verify previous prediction
        if self._current_prediction:
            was_correct = self._current_prediction.predicted_task_type == turn.task_type
            self._prediction_history.append((self._current_prediction, was_correct))
            if len(self._prediction_history) > self._accuracy_window * 2:
                self._prediction_history = self._prediction_history[-self._accuracy_window:]

            # Update chain accuracy
            if self._current_prediction.chain_id:
                self._chain_cache.update_accuracy(
                    self._current_prediction.chain_id, was_correct
                )

            self._current_prediction = None

    def predict_next_turn(self) -> NextTurnPrediction:
        """
        Generate a proactive prediction for the next user turn.

        Algorithm:
        1. Check chain cache for known sequence match (highest priority)
        2. Use trajectory model (Markov chain) as fallback
        3. Combine with weight engine data for tool/model selection
        4. Compute confidence from historical accuracy

        Mathematical model:
          final_confidence = chain_confidence * chain_weight
                           + trajectory_confidence * trajectory_weight
          where:
            chain_weight = 0.7 if chain match found, else 0.0
            trajectory_weight = 1.0 - chain_weight
        """
        recent_types = [t.task_type for t in self._trajectory._history[-5:]]

        # 1. Try chain matching
        chain_match = self._chain_cache.match(recent_types)
        chain_prediction = None
        chain_confidence = 0.0

        if chain_match:
            chain, conf = chain_match
            chain_prediction = chain.next_predicted
            chain_confidence = conf

        # 2. Trajectory model prediction
        trajectory_preds = self._trajectory.predict_next()
        trajectory_top = trajectory_preds[0] if trajectory_preds else ("conversation", 0.3)
        trajectory_prediction, trajectory_confidence = trajectory_top

        # 3. Combine
        if chain_prediction and chain_confidence > 0.3:
            # Chain match found and confident enough
            predicted_type = chain_prediction
            confidence = 0.7 * chain_confidence + 0.3 * trajectory_confidence
            basis = f"Chain match: {':'.join(recent_types[-3:])} -> {chain_prediction}"
            chain_id = chain_match[0].chain_id if chain_match else None
        else:
            # Use trajectory model
            predicted_type = trajectory_prediction
            confidence = trajectory_confidence
            basis = f"Trajectory model (top prediction from {len(trajectory_preds)} candidates)"
            chain_id = None

        # 4. Look up associated tools and model
        predicted_tools = self._task_tool_map.get(predicted_type, [])
        predicted_model = self._task_model_map.get(predicted_type, "")

        # 5. Estimate complexity from recent trend
        recent_complexities = [t.complexity for t in self._trajectory._history[-5:]]
        predicted_complexity = (
            sum(recent_complexities) / len(recent_complexities)
            if recent_complexities else 0.5
        )

        # 6. Determine pre-warm actions
        pre_warm_actions = []
        if predicted_tools:
            pre_warm_actions.append(f"load_tools:{','.join(predicted_tools[:3])}")
        if predicted_model:
            pre_warm_actions.append(f"select_model:{predicted_model}")
        pre_warm_actions.append(f"fetch_context:{predicted_type}")

        # 7. Modulate confidence by historical accuracy
        accuracy = self.get_prediction_accuracy()
        confidence *= (0.5 + accuracy * 0.5)  # Scale by track record

        prediction = NextTurnPrediction(
            predicted_task_type=predicted_type,
            confidence=min(0.95, confidence),
            predicted_tools=predicted_tools,
            predicted_model=predicted_model,
            predicted_complexity=predicted_complexity,
            prediction_basis=basis,
            pre_warm_actions=pre_warm_actions,
            chain_id=chain_id,
        )

        self._current_prediction = prediction
        return prediction

    async def schedule_pre_warming(self, prediction: NextTurnPrediction) -> None:
        """Schedule pre-warming in the background based on prediction."""
        if prediction.confidence > 0.3:  # Only pre-warm if reasonably confident
            await self._pre_warmer.pre_warm(prediction)

    def get_pre_warmed_resources(self) -> Dict[str, Any]:
        """Get any pre-warmed resources from the last prediction."""
        return self._pre_warmer.get_pre_warmed()

    def get_prediction_accuracy(self) -> float:
        """
        Compute the rolling accuracy of proactive predictions.
        Returns 0.0 to 1.0.
        """
        recent = [
            correct for _, correct in self._prediction_history[-self._accuracy_window:]
            if correct is not None
        ]
        if not recent:
            return 0.5  # Prior: assume 50% accuracy
        return sum(1 for r in recent if r) / len(recent)

    def get_stats(self) -> Dict[str, Any]:
        """Get proactive prediction statistics."""
        return {
            "total_predictions": len(self._prediction_history),
            "accuracy": self.get_prediction_accuracy(),
            "trajectory_entropy": self._trajectory.get_trajectory_entropy(),
            "chain_count": len(self._chain_cache._chains),
            "known_task_types": list(self._task_model_map.keys()),
            "current_prediction": (
                self._current_prediction.predicted_task_type
                if self._current_prediction else None
            ),
        }
```

### Integration Points

| Existing File | Integration Point | Change Description |
|---|---|---|
| `corteX/engine/prediction.py` | Add new classes | ProactivePredictionEngine and supporting classes added to existing file |
| `corteX/sdk.py` | `Session.__init__()` | Initialize `self.proactive = ProactivePredictionEngine(memory=self.memory, weight_engine=self.weights)` |
| `corteX/sdk.py` | `Session.run()` | After step 13 (memory store), add: `self.proactive.record_turn(turn)` and `next_pred = self.proactive.predict_next_turn()` and `await self.proactive.schedule_pre_warming(next_pred)` |
| `corteX/sdk.py` | `Session.run()` step 4 | Before prediction, check `pre_warmed = self.proactive.get_pre_warmed_resources()` to use pre-selected model |
| `corteX/engine/adaptation.py` | `AdaptationFilter.process()` | Proactive predictions feed as signals: novel predictions amplified, repeated predictions habituated |
| `corteX/engine/__init__.py` | Module docstring | Update to include proactive prediction |

### Test Scenarios (10+)

1. **Cold start**: No history. `predict_next_turn()` returns default ("conversation") with low confidence.
2. **Simple sequence**: User sends coding, coding, coding. Predict "coding" with increasing confidence.
3. **Chain detection**: User pattern A->B->C seen 5 times. On A->B, predict C with high confidence.
4. **Chain breaking**: Known chain A->B->C, but user sends D. Accuracy tracking updates, chain weakened.
5. **Pre-warming hit**: Predicted "coding" -> pre-warmed code_interpreter. Actual = coding. Verify `was_prediction_useful() == True`.
6. **Pre-warming miss**: Predicted "coding" but actual = "research". Pre-warmed resources invalidated.
7. **Low confidence skip**: Prediction confidence < 0.3. Pre-warming should NOT be triggered.
8. **Trajectory entropy**: Unpredictable user (random task types). Entropy should be high, confidence low.
9. **Recency decay**: Old chain (from 48 hours ago) should have lower confidence than recent chain.
10. **Accuracy feedback loop**: After 10 wrong predictions, overall confidence modulator should reduce all confidences.
11. **Multi-candidate**: Trajectory model returns 3 candidates. Verify the top candidate is selected.
12. **Chain length priority**: Both 2-gram and 3-gram chains match. 3-gram should win (more specific).
13. **Session boundary**: New session starts. Trajectory resets but chain cache persists.

### Edge Cases and Failure Modes

- **Degenerate chains**: User alternates A, B, A, B -- creates contradictory chains. Solution: chain minimum occurrence threshold.
- **Cold user**: Brand new user with zero history. All predictions default, pre-warming disabled.
- **Model not available**: Pre-warmed model is unavailable at execution time. Graceful fallback to default.
- **Concurrent modifications**: Pre-warming runs async while user sends next message. Pre-warm must be thread-safe or use asyncio locks.
- **Memory pressure**: Too many chains stored. Pruning removes lowest-scored chains.

---

## P1: Cross-Modal Association

### Neuroscience Basis

> "Everything is connected to everything through synapses. Almost any neuron you take,
> through a few relay stations, is connected to every other cell." - Prof. Segev, Lecture 4

> The monkey trained to identify shapes by TOUCH cannot transfer that knowledge to VISION.
> But humans can, because "in our brain there are such intensive connections between areas
> that I immediately know how to associate something I touch." - Lecture 4, lines 976-1000

### Architecture Overview

```
CrossModalAssociator
  |
  +-- DomainSimilarityMap (defines which task types are "nearby")
  |
  +-- TransferLearningEngine (applies echo updates across domains)
  |
  +-- AssociationStrengthTracker (tracks and decays cross-domain links)
```

### File: `corteX/engine/association.py` (New File)

### Data Structures

```python
@dataclass
class DomainAssociation:
    """A learned association between two task type domains."""
    domain_a: str
    domain_b: str
    strength: float         # 0.0 (unrelated) to 1.0 (strongly associated)
    co_occurrence_count: int
    transfer_success_count: int
    transfer_failure_count: int
    last_transfer: float = 0.0
    created_at: float = field(default_factory=time.time)


@dataclass
class TransferEvent:
    """Record of a cross-modal transfer attempt."""
    source_domain: str
    target_domain: str
    source_pattern_key: str     # e.g., "code_interpreter+coding"
    target_pattern_key: str     # e.g., "code_interpreter+debugging"
    echo_strength: float        # How strong the transfer was
    verified: Optional[bool] = None  # Was the transfer successful?
    timestamp: float = field(default_factory=time.time)
```

### Class: DomainSimilarityMap

```python
class DomainSimilarityMap:
    """
    Maintains a similarity graph between task type domains.
    Similarity is both hardcoded (initial priors) and learned (from co-occurrence).

    The similarity map is a weighted undirected graph:
      sim(A, B) = prior_similarity * alpha + learned_similarity * (1 - alpha)

    Prior similarities encode domain knowledge:
      coding <-> debugging: 0.8 (very related)
      coding <-> testing: 0.7
      research <-> summarization: 0.6
      conversation <-> coding: 0.2 (weakly related)

    Learned similarities update based on:
      1. Co-occurrence within sessions (user switches between domains)
      2. Transfer success rate (echoed weights that prove correct)

    Mathematical model for learned similarity:
      sim_learned(A, B) = co_occurrence(A,B) / (count(A) + count(B)) * 2
                        * (transfer_successes / (transfer_successes + transfer_failures + k))
      where k = 5 (pseudocount for Bayesian smoothing)
    """

    # Initial prior similarities (symmetric)
    DEFAULT_PRIORS: Dict[Tuple[str, str], float] = {
        ("coding", "debugging"): 0.8,
        ("coding", "testing"): 0.7,
        ("coding", "tool_use"): 0.6,
        ("debugging", "testing"): 0.7,
        ("debugging", "tool_use"): 0.5,
        ("research", "summarization"): 0.6,
        ("research", "reasoning"): 0.6,
        ("planning", "reasoning"): 0.7,
        ("planning", "coding"): 0.5,
        ("conversation", "summarization"): 0.4,
        ("validation", "testing"): 0.6,
        ("validation", "debugging"): 0.5,
    }

    def __init__(self, alpha: float = 0.4):
        """alpha controls the blend: 0.0 = all learned, 1.0 = all prior."""
        self.alpha = alpha
        self._associations: Dict[str, DomainAssociation] = {}
        self._domain_counts: Dict[str, int] = defaultdict(int)

    def _pair_key(self, a: str, b: str) -> str:
        """Canonical key for an unordered pair."""
        return f"{min(a,b)}:{max(a,b)}"

    def get_similarity(self, domain_a: str, domain_b: str) -> float:
        """
        Get the current similarity between two domains.
        Combines prior and learned similarity.
        """
        if domain_a == domain_b:
            return 1.0

        # Prior similarity
        pair = (min(domain_a, domain_b), max(domain_a, domain_b))
        prior = self.DEFAULT_PRIORS.get(pair, 0.1)  # Default: slight association

        # Learned similarity
        key = self._pair_key(domain_a, domain_b)
        assoc = self._associations.get(key)
        if assoc is None:
            return prior  # No learned data yet

        # Co-occurrence based similarity
        total_count = self._domain_counts.get(domain_a, 1) + self._domain_counts.get(domain_b, 1)
        co_occ_sim = min(1.0, assoc.co_occurrence_count / total_count * 2)

        # Transfer success rate
        total_transfers = assoc.transfer_success_count + assoc.transfer_failure_count
        k = 5  # Pseudocount
        transfer_sim = assoc.transfer_success_count / (total_transfers + k)

        learned = co_occ_sim * 0.5 + transfer_sim * 0.5

        # Blend prior and learned
        return self.alpha * prior + (1 - self.alpha) * learned

    def record_co_occurrence(self, domain_a: str, domain_b: str) -> None:
        """Record that two domains appeared in the same session context."""
        if domain_a == domain_b:
            return
        self._domain_counts[domain_a] += 1
        self._domain_counts[domain_b] += 1

        key = self._pair_key(domain_a, domain_b)
        if key not in self._associations:
            self._associations[key] = DomainAssociation(
                domain_a=min(domain_a, domain_b),
                domain_b=max(domain_a, domain_b),
                strength=0.1,
                co_occurrence_count=0,
                transfer_success_count=0,
                transfer_failure_count=0,
            )
        self._associations[key].co_occurrence_count += 1

    def record_transfer_result(self, domain_a: str, domain_b: str, success: bool) -> None:
        """Record whether a cross-modal transfer was successful."""
        key = self._pair_key(domain_a, domain_b)
        assoc = self._associations.get(key)
        if assoc:
            if success:
                assoc.transfer_success_count += 1
            else:
                assoc.transfer_failure_count += 1

    def get_related_domains(self, domain: str, min_similarity: float = 0.3) -> List[Tuple[str, float]]:
        """Get all domains related to the given domain above threshold."""
        all_domains = set()
        for key in self._associations:
            parts = key.split(":")
            all_domains.update(parts)
        # Also include domains from priors
        for (a, b) in self.DEFAULT_PRIORS:
            all_domains.add(a)
            all_domains.add(b)

        related = []
        for other in all_domains:
            if other != domain:
                sim = self.get_similarity(domain, other)
                if sim >= min_similarity:
                    related.append((other, sim))
        related.sort(key=lambda x: x[1], reverse=True)
        return related
```

### Class: CrossModalAssociator

```python
class CrossModalAssociator:
    """
    The main cross-modal association engine.
    After a successful Hebbian update in one domain, applies a weaker
    "echo" update to associated domains.

    Brain analogy: When you learn that a rough texture (touch) is
    associated with a visual pattern, future encounters with that
    visual pattern already activate touch-related predictions.

    Echo update formula:
      echo_delta = original_delta * similarity(source, target) * echo_decay
      where echo_decay = 0.3 (echoes are 30% of original strength at max similarity)

    This means:
    - If coding success strengthens code_interpreter by +0.1,
      and similarity(coding, debugging) = 0.8:
      debugging echo = 0.1 * 0.8 * 0.3 = 0.024

    Integration with PlasticityManager:
    - Called by PlasticityManager.on_step_complete() after Hebbian rule fires
    - Generates additional PlasticityEvents for the echo updates

    Usage:
        associator = CrossModalAssociator(weight_engine, similarity_map)

        # After a successful Hebbian update
        echo_events = associator.propagate_learning(
            source_domain="coding",
            pattern_key="code_interpreter+coding",
            delta=0.1,
            outcome_quality=0.85,
        )
    """

    def __init__(
        self,
        weight_engine: WeightEngine,
        similarity_map: Optional[DomainSimilarityMap] = None,
        echo_decay: float = 0.3,
        min_echo_strength: float = 0.005,
        max_echo_domains: int = 3,
    ):
        self._weights = weight_engine
        self._similarity = similarity_map or DomainSimilarityMap()
        self._echo_decay = echo_decay
        self._min_echo = min_echo_strength
        self._max_domains = max_echo_domains
        self._transfer_history: List[TransferEvent] = []

    def propagate_learning(
        self,
        source_domain: str,
        pattern_key: str,
        delta: float,
        outcome_quality: float,
    ) -> List[PlasticityEvent]:
        """
        Propagate learning from source domain to associated domains.

        Args:
            source_domain: The task type where the original learning occurred
            pattern_key: The weight pattern key (e.g., "code_interpreter+coding")
            delta: The original Hebbian delta applied
            outcome_quality: The quality of the original outcome (0.0 to 1.0)

        Returns:
            List of PlasticityEvents for the echo updates
        """
        events: List[PlasticityEvent] = []

        # Only propagate sufficiently strong learning signals
        if abs(delta) < self._min_echo:
            return events

        # Only propagate on quality outcomes (don't spread failure aggressively)
        if outcome_quality < 0.4:
            return events

        # Get related domains
        related = self._similarity.get_related_domains(source_domain)[:self._max_domains]

        # Parse the pattern key to extract tool/model
        parts = pattern_key.split("+")
        if len(parts) != 2:
            return events
        tool_or_model, _ = parts

        for target_domain, similarity in related:
            # Compute echo strength
            echo_delta = delta * similarity * self._echo_decay

            # Below threshold? Skip.
            if abs(echo_delta) < self._min_echo:
                continue

            # Apply echo update to weight engine
            target_key = f"{tool_or_model}+{target_domain}"
            self._weights.models.update(target_domain, tool_or_model, echo_delta)

            # Record the transfer
            transfer = TransferEvent(
                source_domain=source_domain,
                target_domain=target_domain,
                source_pattern_key=pattern_key,
                target_pattern_key=target_key,
                echo_strength=echo_delta,
            )
            self._transfer_history.append(transfer)

            # Record co-occurrence
            self._similarity.record_co_occurrence(source_domain, target_domain)

            events.append(PlasticityEvent(
                rule="cross_modal_echo",
                affected_weights=[target_key],
                magnitude=abs(echo_delta),
                details=(
                    f"Echo from {source_domain}->{target_domain}: "
                    f"delta={echo_delta:+.4f} (sim={similarity:.2f})"
                ),
            ))

        return events

    def verify_transfers(
        self,
        domain: str,
        tool_or_model: str,
        success: bool,
    ) -> None:
        """
        After a step in a target domain, verify whether previous
        echo transfers to this domain were helpful.

        Updates the similarity map based on transfer outcomes.
        """
        # Find recent unverified transfers targeting this domain
        for transfer in reversed(self._transfer_history[-50:]):
            if transfer.target_domain == domain and transfer.verified is None:
                transfer.verified = success
                self._similarity.record_transfer_result(
                    transfer.source_domain,
                    transfer.target_domain,
                    success,
                )

    def get_stats(self) -> Dict[str, Any]:
        """Get cross-modal association statistics."""
        verified = [t for t in self._transfer_history if t.verified is not None]
        successes = sum(1 for t in verified if t.verified)
        return {
            "total_transfers": len(self._transfer_history),
            "verified_transfers": len(verified),
            "transfer_success_rate": successes / max(len(verified), 1),
            "active_domains": len(self._similarity._domain_counts),
            "association_count": len(self._similarity._associations),
        }
```

### Integration Points

| Existing File | Integration Point | Change Description |
|---|---|---|
| `corteX/engine/association.py` | New file | Contains DomainSimilarityMap and CrossModalAssociator |
| `corteX/engine/plasticity.py` | `PlasticityManager.__init__()` | Add `self.associator = CrossModalAssociator(weight_engine)` |
| `corteX/engine/plasticity.py` | `PlasticityManager.on_step_complete()` | After Hebbian rule fires, call `self.associator.propagate_learning(task_type, pattern_key, hebbian_delta, quality)` |
| `corteX/engine/plasticity.py` | `PlasticityManager.on_step_complete()` | Add `self.associator.verify_transfers(task_type, tool, success)` |
| `corteX/engine/__init__.py` | Module docstring | Update to include association |

### Test Scenarios (10+)

1. **Basic echo**: Coding success delta +0.1, similarity(coding,debugging) = 0.8. Verify debugging receives echo of ~0.024.
2. **No echo below threshold**: Very small delta (0.001). No echo should propagate.
3. **Failure suppression**: Negative outcome (quality < 0.4). No echo should propagate.
4. **Multiple targets**: Source domain "coding" has 3 related domains. All 3 receive echoes.
5. **Asymmetric similarity**: After transfer success in coding->debugging but failure in coding->conversation, similarity should diverge.
6. **Transfer verification**: Propagate echo, then verify the target domain had success. Similarity should increase.
7. **Transfer failure**: Propagate echo, target domain fails. Similarity should decrease.
8. **Co-occurrence learning**: User alternates coding and debugging. Learned similarity should increase.
9. **Unknown domain**: Source domain "exotic_task" with no priors. Default similarity (0.1) applies.
10. **Max domains cap**: Source has 10 related domains but max_echo_domains=3. Only top 3 receive echoes.
11. **Bidirectional**: Echo from A->B, then later from B->A. Both should work independently.
12. **Decay interaction**: Echo propagation interacts with homeostatic regulation. Verify echoed weights don't exceed bounds.
13. **Population coding interaction**: Echo updates should be visible as weak votes in PopulationDecoder.

### Edge Cases and Failure Modes

- **Echo amplification loop**: A echoes to B, B echoes to A, creating runaway strengthening. Solution: echo_decay ensures each hop reduces by 70%. Single-hop only (no cascading echoes).
- **Stale associations**: Old co-occurrences from different user behavior. Solution: time-decay associations.
- **Domain explosion**: Too many domains. Solution: cap on tracked associations.
- **Contradictory transfers**: Tool X succeeds in coding but fails in debugging. Similarity should naturally decrease via `record_transfer_result`.

---

## P1: Continuous Calibration

### Neuroscience Basis

> "You wake up in the morning - your brain needs to re-learn itself. You're not exactly the
> same person as before... So the network must necessarily reorganize slightly...
> This is REAL-TIME learning - the network reorganizes all the time." - Prof. Segev, Lecture 3

The BCI (Brain-Computer Interface) adaptive algorithm continuously recalibrates the mapping
between neural signals and desired outputs. The brain's neural code is not fixed -- it drifts
daily, requiring the interface to constantly re-learn.

Similarly, the corteX agent-user relationship drifts: the user's preferences change, their
skill level evolves, their mood shifts between sessions. The agent must detect and adapt to
this drift continuously, not only at discrete feedback moments.

### Architecture Overview

```
ContinuousCalibrationEngine
  |
  +-- SessionStartCalibrator (micro-calibration at session start)
  |
  +-- DriftDetector (detects relationship drift during session)
  |
  +-- RecalibrationScheduler (periodic recalibration triggers)
  |
  +-- CalibrationMetrics (tracks calibration quality over time)
```

### File: `corteX/engine/calibration.py` (New File)

### Data Structures

```python
@dataclass
class CalibrationSnapshot:
    """A snapshot of the agent-user relationship state."""
    snapshot_id: str
    timestamp: float
    behavioral_weights: Dict[str, float]
    prediction_accuracy: float
    feedback_distribution: Dict[str, int]   # signal_type -> count
    avg_response_quality: float
    avg_response_latency_ms: float
    user_engagement_score: float
    turn_count: int
    session_id: str


@dataclass
class DriftReport:
    """Report on detected drift between two snapshots."""
    dimension: str              # Which weight/metric drifted
    old_value: float
    new_value: float
    drift_magnitude: float      # Absolute change
    drift_rate: float           # Change per unit time
    significance: float         # 0.0 to 1.0 (statistical significance)
    recommended_action: str     # "no_action", "recalibrate", "alert"


@dataclass
class CalibrationResult:
    """Result of a calibration cycle."""
    adjustments: List[Tuple[str, float, float]]  # (weight_key, old_value, new_value)
    drift_reports: List[DriftReport]
    calibration_quality: float                     # 0.0 to 1.0
    timestamp: float = field(default_factory=time.time)
```

### Class: SessionStartCalibrator

```python
class SessionStartCalibrator:
    """
    Runs micro-calibration at session start.
    "What has changed since last session?"

    Compares the current state to the last session's end state
    and adjusts weights based on detected drift.

    The calibration process:
    1. Load the last session's ending snapshot
    2. Compare key metrics to detect drift
    3. Apply "wake-up" adjustments:
       - If prediction accuracy was declining: increase learning rate
       - If user frustration was rising: pre-adjust behavioral weights
       - If engagement was dropping: increase initiative
    4. Apply a mild "regression to mean" on extreme weights
       (analogous to the brain's overnight reorganization)

    Mathematical model:
      For each behavioral weight w:
        drift_d = w_last_session_end - w_session_before_that_end
        if |drift_d| > threshold:
          w_session_start = w_last_session_end + momentum * drift_d
        else:
          w_session_start = w_last_session_end * (1 - regression_rate) + w_prior * regression_rate

    Where:
      momentum = 0.3 (continue observed trends mildly)
      regression_rate = 0.05 (gentle pull toward defaults)
      threshold = 0.1 (minimum drift to count as trend)
    """

    def __init__(
        self,
        regression_rate: float = 0.05,
        momentum: float = 0.3,
        drift_threshold: float = 0.1,
    ):
        self._regression_rate = regression_rate
        self._momentum = momentum
        self._drift_threshold = drift_threshold
        self._session_snapshots: List[CalibrationSnapshot] = []

    def calibrate(
        self,
        weight_engine: WeightEngine,
        previous_snapshot: Optional[CalibrationSnapshot] = None,
        older_snapshot: Optional[CalibrationSnapshot] = None,
    ) -> CalibrationResult:
        """
        Run session-start micro-calibration.
        Adjusts weights based on inter-session drift analysis.
        """
        adjustments: List[Tuple[str, float, float]] = []
        drift_reports: List[DriftReport] = []

        if previous_snapshot is None:
            return CalibrationResult(
                adjustments=[],
                drift_reports=[],
                calibration_quality=0.5,
            )

        # Analyze behavioral weight drift
        for key, current_val in weight_engine.behavioral.weights.items():
            prev_val = previous_snapshot.behavioral_weights.get(key, current_val)

            # Compute drift from two sessions ago (if available)
            trend = 0.0
            if older_snapshot:
                older_val = older_snapshot.behavioral_weights.get(key, prev_val)
                trend = prev_val - older_val

            # Apply momentum if significant trend detected
            if abs(trend) > self._drift_threshold:
                # Continue the trend mildly
                adjustment = self._momentum * trend
                new_val = max(-1.0, min(1.0, current_val + adjustment))
                if abs(new_val - current_val) > 0.001:
                    weight_engine.behavioral.weights[key] = new_val
                    adjustments.append((key, current_val, new_val))
                    drift_reports.append(DriftReport(
                        dimension=f"behavioral.{key}",
                        old_value=current_val,
                        new_value=new_val,
                        drift_magnitude=abs(trend),
                        drift_rate=abs(trend),
                        significance=min(1.0, abs(trend) / 0.3),
                        recommended_action="recalibrate",
                    ))
            else:
                # Gentle regression toward default (0.0 for most weights)
                default = 0.0
                regression = self._regression_rate * (default - current_val)
                if abs(regression) > 0.001:
                    new_val = current_val + regression
                    weight_engine.behavioral.weights[key] = new_val
                    adjustments.append((key, current_val, new_val))

        # Analyze prediction accuracy drift
        if previous_snapshot.prediction_accuracy < 0.4:
            # Prediction accuracy was poor last session - increase learning
            drift_reports.append(DriftReport(
                dimension="prediction_accuracy",
                old_value=previous_snapshot.prediction_accuracy,
                new_value=0.0,  # Target: improve
                drift_magnitude=0.5 - previous_snapshot.prediction_accuracy,
                drift_rate=0.0,
                significance=0.8,
                recommended_action="recalibrate",
            ))

        # Compute calibration quality
        quality = 1.0 - (len(drift_reports) * 0.1)  # More drift = lower quality
        quality = max(0.0, min(1.0, quality))

        return CalibrationResult(
            adjustments=adjustments,
            drift_reports=drift_reports,
            calibration_quality=quality,
        )

    def take_snapshot(
        self,
        weight_engine: WeightEngine,
        prediction_engine: PredictionEngine,
        feedback_engine: FeedbackEngine,
        session_id: str,
        turn_count: int,
    ) -> CalibrationSnapshot:
        """Take a snapshot of current state for future calibration."""
        snapshot = CalibrationSnapshot(
            snapshot_id=f"snap_{int(time.time())}_{session_id}",
            timestamp=time.time(),
            behavioral_weights=dict(weight_engine.behavioral.weights),
            prediction_accuracy=1.0 - prediction_engine.get_calibration_score(),
            feedback_distribution={},
            avg_response_quality=0.5,
            avg_response_latency_ms=0.0,
            user_engagement_score=weight_engine.user_insights.get("engagement_level", 0.5),
            turn_count=turn_count,
            session_id=session_id,
        )
        self._session_snapshots.append(snapshot)
        if len(self._session_snapshots) > 20:
            self._session_snapshots = self._session_snapshots[-20:]
        return snapshot
```

### Class: DriftDetector

```python
class DriftDetector:
    """
    Continuously monitors for drift in the agent-user relationship.
    Runs during a session (not just at boundaries).

    Drift types detected:
    1. Behavioral drift: User's preferences change within a session
    2. Performance drift: Agent's accuracy degrades
    3. Engagement drift: User engagement level changes
    4. Complexity drift: Task complexity shifts up or down

    Detection method: Exponentially weighted moving average (EWMA)
    control chart. When a metric's EWMA crosses +/- 2 sigma from
    the running mean, drift is detected.

    Mathematical model:
      EWMA_t = lambda * x_t + (1 - lambda) * EWMA_{t-1}
      where lambda = 2 / (span + 1), span = 10

      Upper control limit = mean + L * sigma * sqrt(lambda / (2 - lambda))
      Lower control limit = mean - L * sigma * sqrt(lambda / (2 - lambda))
      where L = 2.5 (sensitivity parameter)
    """

    def __init__(self, ewma_span: int = 10, sensitivity: float = 2.5):
        self._lambda = 2.0 / (ewma_span + 1)
        self._sensitivity = sensitivity
        self._metrics: Dict[str, List[float]] = defaultdict(list)
        self._ewma: Dict[str, float] = {}
        self._running_mean: Dict[str, float] = {}
        self._running_var: Dict[str, float] = {}

    def record_metric(self, metric_name: str, value: float) -> Optional[DriftReport]:
        """
        Record a metric value and check for drift.
        Returns a DriftReport if drift is detected, else None.
        """
        self._metrics[metric_name].append(value)
        if len(self._metrics[metric_name]) > 200:
            self._metrics[metric_name] = self._metrics[metric_name][-200:]

        # Update EWMA
        if metric_name not in self._ewma:
            self._ewma[metric_name] = value
            self._running_mean[metric_name] = value
            self._running_var[metric_name] = 0.0
            return None

        prev_ewma = self._ewma[metric_name]
        self._ewma[metric_name] = self._lambda * value + (1 - self._lambda) * prev_ewma

        # Update running statistics
        n = len(self._metrics[metric_name])
        old_mean = self._running_mean[metric_name]
        new_mean = old_mean + (value - old_mean) / n
        self._running_mean[metric_name] = new_mean

        if n > 2:
            old_var = self._running_var[metric_name]
            new_var = old_var + (value - old_mean) * (value - new_mean)
            self._running_var[metric_name] = new_var
            sigma = math.sqrt(new_var / (n - 1)) if n > 1 else 1.0
        else:
            sigma = 1.0
            return None  # Not enough data yet

        # Control limits
        control_factor = sigma * math.sqrt(self._lambda / (2 - self._lambda))
        ucl = new_mean + self._sensitivity * control_factor
        lcl = new_mean - self._sensitivity * control_factor

        # Check for drift
        current_ewma = self._ewma[metric_name]
        if current_ewma > ucl or current_ewma < lcl:
            drift_mag = abs(current_ewma - new_mean) / max(sigma, 0.001)
            return DriftReport(
                dimension=metric_name,
                old_value=new_mean,
                new_value=current_ewma,
                drift_magnitude=abs(current_ewma - new_mean),
                drift_rate=abs(value - prev_ewma),
                significance=min(1.0, drift_mag / self._sensitivity),
                recommended_action="recalibrate" if drift_mag > self._sensitivity else "monitor",
            )

        return None

    def get_drift_summary(self) -> Dict[str, Any]:
        """Get summary of all tracked metrics and their drift status."""
        summary = {}
        for metric_name in self._metrics:
            values = self._metrics[metric_name]
            summary[metric_name] = {
                "current_ewma": self._ewma.get(metric_name, 0.0),
                "running_mean": self._running_mean.get(metric_name, 0.0),
                "sample_count": len(values),
                "recent_values": values[-5:],
            }
        return summary
```

### Class: ContinuousCalibrationEngine

```python
class ContinuousCalibrationEngine:
    """
    Main calibration engine coordinating all calibration components.

    The BCI insight: calibration is not a one-time setup.
    It is a continuous process running throughout the interaction.

    Calibration triggers:
    1. Session start: Full micro-calibration
    2. Every N turns: Drift check on key metrics
    3. After significant surprise: Re-evaluate calibration
    4. On explicit user correction: Immediate recalibration

    Usage:
        calibration = ContinuousCalibrationEngine(
            weight_engine=weights,
            prediction_engine=prediction,
            feedback_engine=feedback,
        )

        # At session start
        result = calibration.on_session_start(session_id)

        # After each turn
        drift = calibration.on_turn_complete(
            quality=0.8,
            latency_ms=1500,
            user_engagement=0.7,
        )

        # Periodic deep calibration
        if turn_count % 10 == 0:
            calibration.deep_calibrate()
    """

    def __init__(
        self,
        weight_engine: WeightEngine,
        prediction_engine: Optional[PredictionEngine] = None,
        feedback_engine: Optional[FeedbackEngine] = None,
        calibration_interval: int = 10,    # Turns between drift checks
    ):
        self._weights = weight_engine
        self._prediction = prediction_engine
        self._feedback = feedback_engine
        self._interval = calibration_interval

        self._session_calibrator = SessionStartCalibrator()
        self._drift_detector = DriftDetector()

        self._turn_count: int = 0
        self._session_id: str = ""
        self._calibration_history: List[CalibrationResult] = []

    def on_session_start(self, session_id: str) -> CalibrationResult:
        """
        Run session-start calibration.
        "You wake up in the morning - your brain needs to re-learn itself."
        """
        self._session_id = session_id
        self._turn_count = 0

        # Get previous snapshots
        snapshots = self._session_calibrator._session_snapshots
        previous = snapshots[-1] if snapshots else None
        older = snapshots[-2] if len(snapshots) >= 2 else None

        result = self._session_calibrator.calibrate(
            self._weights, previous, older
        )

        self._calibration_history.append(result)
        if result.adjustments:
            logger.info(
                f"Session-start calibration: {len(result.adjustments)} adjustments, "
                f"quality={result.calibration_quality:.2f}"
            )

        return result

    def on_turn_complete(
        self,
        quality: float = 0.5,
        latency_ms: float = 0.0,
        user_engagement: float = 0.5,
        surprise_magnitude: float = 0.0,
        task_complexity: float = 0.5,
    ) -> List[DriftReport]:
        """
        Called after each turn to check for drift.
        Returns any drift reports detected.
        """
        self._turn_count += 1
        drift_reports: List[DriftReport] = []

        # Record metrics for drift detection
        metrics = {
            "response_quality": quality,
            "response_latency_ms": latency_ms / 5000.0,  # Normalize
            "user_engagement": user_engagement,
            "surprise": surprise_magnitude,
            "task_complexity": task_complexity,
        }

        for name, value in metrics.items():
            report = self._drift_detector.record_metric(name, value)
            if report:
                drift_reports.append(report)

        # Apply corrective actions for significant drift
        for report in drift_reports:
            if report.recommended_action == "recalibrate":
                self._apply_drift_correction(report)

        return drift_reports

    def _apply_drift_correction(self, report: DriftReport) -> None:
        """Apply a corrective weight adjustment for detected drift."""
        if report.dimension == "response_quality" and report.new_value < report.old_value:
            # Quality declining - increase caution
            self._weights.behavioral.update("risk_tolerance", -0.05)
            self._weights.behavioral.update("speed_vs_quality", -0.05)
            logger.info(f"Drift correction: quality declining, reducing risk tolerance")

        elif report.dimension == "user_engagement" and report.new_value < report.old_value:
            # Engagement dropping - increase initiative
            self._weights.behavioral.update("initiative", 0.05)
            logger.info(f"Drift correction: engagement dropping, increasing initiative")

        elif report.dimension == "response_latency_ms" and report.new_value > report.old_value:
            # Getting slower - shift toward speed
            self._weights.behavioral.update("speed_vs_quality", 0.03)
            logger.info(f"Drift correction: latency increasing, shifting toward speed")

    def on_session_end(self) -> CalibrationSnapshot:
        """Take an end-of-session snapshot for future calibration."""
        return self._session_calibrator.take_snapshot(
            weight_engine=self._weights,
            prediction_engine=self._prediction or PredictionEngine(),
            feedback_engine=self._feedback or FeedbackEngine(self._weights),
            session_id=self._session_id,
            turn_count=self._turn_count,
        )

    def deep_calibrate(self) -> CalibrationResult:
        """
        Run a deep calibration cycle.
        Examines all weight categories and adjusts learning rates.

        This is more expensive than per-turn drift checking
        and should run every N turns (default: 10).
        """
        adjustments: List[Tuple[str, float, float]] = []
        drift_reports: List[DriftReport] = []

        # Check if prediction engine is well-calibrated
        if self._prediction:
            cal_error = self._prediction.get_calibration_score()
            if cal_error > 0.4:
                # Predictions are poor - increase plasticity
                drift_reports.append(DriftReport(
                    dimension="prediction_calibration",
                    old_value=0.0,
                    new_value=cal_error,
                    drift_magnitude=cal_error,
                    drift_rate=0.0,
                    significance=cal_error,
                    recommended_action="recalibrate",
                ))

        # Check behavioral weight entropy (all weights near zero = under-adapted)
        behavioral = self._weights.behavioral.weights
        weight_magnitudes = [abs(v) for v in behavioral.values()]
        avg_magnitude = sum(weight_magnitudes) / max(len(weight_magnitudes), 1)

        if avg_magnitude < 0.05:
            # System hasn't adapted enough - might be stuck at defaults
            drift_reports.append(DriftReport(
                dimension="behavioral_entropy",
                old_value=0.0,
                new_value=avg_magnitude,
                drift_magnitude=0.05 - avg_magnitude,
                drift_rate=0.0,
                significance=0.5,
                recommended_action="alert",
            ))

        result = CalibrationResult(
            adjustments=adjustments,
            drift_reports=drift_reports,
            calibration_quality=1.0 - len(drift_reports) * 0.15,
        )
        self._calibration_history.append(result)
        return result

    def get_stats(self) -> Dict[str, Any]:
        """Get calibration engine statistics."""
        return {
            "session_id": self._session_id,
            "turn_count": self._turn_count,
            "calibration_cycles": len(self._calibration_history),
            "drift_summary": self._drift_detector.get_drift_summary(),
            "snapshots_stored": len(self._session_calibrator._session_snapshots),
        }
```

### Integration Points

| Existing File | Integration Point | Change Description |
|---|---|---|
| `corteX/engine/calibration.py` | New file | Contains ContinuousCalibrationEngine and supporting classes |
| `corteX/sdk.py` | `Session.__init__()` | Initialize `self.calibration = ContinuousCalibrationEngine(self.weights, self.prediction, self.feedback)` |
| `corteX/sdk.py` | `Session.__init__()` | Call `self.calibration.on_session_start(self.session_id)` |
| `corteX/sdk.py` | `Session.run()` after step 11 | Call `self.calibration.on_turn_complete(quality=estimated_quality, latency_ms=latency_ms, user_engagement=..., surprise_magnitude=surprise.surprise_magnitude)` |
| `corteX/sdk.py` | `Session.run()` periodically | Every 10 turns: `self.calibration.deep_calibrate()` |
| `corteX/sdk.py` | `Session.close()` | Call `self.calibration.on_session_end()` to save snapshot |
| `corteX/engine/plasticity.py` | `CriticalPeriodModulator` | Calibration can reset or extend critical period based on drift magnitude |

### Test Scenarios (10+)

1. **First session**: No previous snapshots. Calibration is a no-op with default quality.
2. **Session continuity**: Save snapshot at session end. Verify it is loaded at next session start.
3. **Drift momentum**: User's verbosity preference trending upward across sessions. Session-start calibration should continue the trend.
4. **Regression to mean**: Weight at extreme (0.9) with no drift trend. Should regress slightly toward 0.0.
5. **Quality drift detection**: Agent quality drops for 5 consecutive turns. DriftDetector fires.
6. **Engagement drift correction**: Engagement drops below control limit. Initiative weight should increase.
7. **Latency drift**: Response times increasing. speed_vs_quality should shift.
8. **Deep calibration low entropy**: All weights near 0. Should detect under-adaptation.
9. **Prediction accuracy decline**: Calibration score > 0.4 last session. Should flag for recalibration.
10. **No drift**: Stable metrics for 20 turns. No drift reports should be generated.
11. **Correction trigger**: User sends explicit correction. Immediate recalibration path.
12. **Multiple sessions**: Run 5 sessions, verify snapshot chain and trend detection across them.
13. **EWMA control chart**: Inject a step-change in metrics. Verify detection within 3-5 turns.

### Edge Cases and Failure Modes

- **Missing snapshots**: Storage backend loses snapshots. Graceful fallback to no calibration.
- **Rapid session cycling**: Many very short sessions. Snapshots may not have enough data. Solution: minimum turn count for snapshot.
- **Contradictory drift**: Quality improving but engagement dropping. Each dimension calibrated independently.
- **EWMA cold start**: First few data points produce unstable EWMA. Solution: require minimum sample count before checking control limits.

---

## P2: Functional Columns

### High-Level Design

> "In the primary visual cortex there are functional columns, meaning 10,000 cells now fire
> when I show you this angle." - Prof. Segev, Lecture 3

A functional column bundles tools + models + behavioral weights into a coherent unit that
specializes in a task domain. Columns compete for activation based on the incoming request.

**Proposed Architecture:**

```python
@dataclass
class FunctionalColumn:
    """A coherent processing bundle."""
    column_id: str
    domain: str                         # e.g., "coding", "research"
    preferred_tools: List[str]
    preferred_model: str
    behavioral_overrides: Dict[str, float]  # Override weights when this column is active
    activation_score: float = 0.0
    specialization_level: float = 0.5   # How specialized this column has become
    success_rate: float = 0.5
```

**Key Ideas:**
- Columns are learned, not manually defined (though initial columns can be seeded)
- Column competition: multiple columns score the request, highest activation wins
- Winner-take-all with inhibition: winning column suppresses others (lateral inhibition)
- Over time, columns specialize through Hebbian learning
- New columns can emerge when existing ones fail to cover a task type

**Relationship to P0/P1:**
- Proactive Prediction predicts which column will activate next (pre-warm the column)
- Cross-Modal Association creates links between columns (coding column activates partial debugging column)
- Continuous Calibration recalibrates column activation thresholds

**Dependencies:**
- Requires Population Coding (columns are scored by population decoder)
- Requires Adaptation (column activation patterns habituate)
- Requires the full weight engine API

### File: `corteX/engine/columns.py` (Future)

---

## P2: Attentional Filter

### High-Level Design

> "We are very unconscious creatures... what I describe to you are things I AM conscious of,
> and that's only a small part of brain activity." - Prof. Segev, Lecture 3

The change blindness experiment demonstrates that we process far less than we think.
The Attentional Filter implements a "subconscious" processing tier that handles routine
patterns without consuming full orchestrator resources.

**Proposed Architecture:**

```python
class AttentionalFilter:
    """
    Two-tier processing:
    1. Subconscious: Fast pattern-matched responses for routine queries
    2. Conscious: Full orchestrator pipeline for novel/complex queries
    """

    def classify(self, user_message: str, context: Dict) -> str:
        """Returns 'subconscious' or 'conscious'."""
        ...

    def compute_delta(self, previous_context: Dict, current_context: Dict) -> Dict:
        """Compute only what changed between turns."""
        ...
```

**Key Ideas:**
- Delta processing: Only feed the orchestrator what CHANGED between turns
- Routine routing: Queries matching established patterns get fast-tracked
- Novelty detection: Novel requests trigger full "conscious" processing
- Reduces token consumption by 30-50% for habituated conversation patterns

**Relationship to P0/P1:**
- Uses Adaptation's habituation system to determine what's "routine"
- Proactive Prediction feeds the filter: predicted queries can be pre-classified
- Continuous Calibration tracks what percentage of queries go to each tier

**Dependencies:**
- Requires Adaptation engine (habituation detection)
- Requires Population Coding (classify novelty via population vote)

### File: `corteX/engine/attention.py` (Future)

---

## Cross-Pattern Interactions

### 1. Proactive Prediction feeds Adaptation

The prediction system generates a continuous stream of predictions. If the same prediction
is made repeatedly (user always follows coding with debugging), the Adaptation engine should
habituate to this prediction -- reducing the "surprise" signal when it proves correct.

```
ProactivePredictionEngine.predict_next_turn()
  -> prediction_type = "debugging" (for the 10th time)
  -> AdaptationFilter.process("prediction:debugging", confidence)
  -> Returns adaptation_weight = 0.1 (habituated, low novelty)

But if prediction is WRONG (user sends "research" instead):
  -> AdaptationFilter detects CHANGE
  -> adaptation_weight = 2.0 (novelty bonus!)
  -> Triggers increased learning signal via SurpriseSignal
```

**Integration code (in Session.run()):**
```python
# After proactive prediction
prediction = self.proactive.predict_next_turn()
adapted_pred = self.adaptation.process(
    f"proactive:{prediction.predicted_task_type}",
    prediction.confidence
)
# If adapted_pred is None (habituated prediction), reduce pre-warming priority
# If adapted_pred.is_novel, amplify pre-warming and increase plasticity
```

### 2. Cross-Modal Association interacts with Population Coding

Echo updates from cross-modal transfer should appear as weak votes in the
PopulationDecoder when making tool/model selection decisions.

```
CrossModalAssociator.propagate_learning("coding", ...)
  -> Generates echo to "debugging" domain
  -> PopulationToolSelector receives echo as a weak vote:
     decoder.add_vote("cross_modal_echo", echo_strength, confidence=0.2)
```

This means cross-modal learning doesn't override population-level decisions --
it adds a gentle bias that the population can accept or reject.

**Integration code (in PopulationToolSelector.select()):**
```python
# When scoring a tool for a domain, add cross-modal echoes as weak votes
echo_score = associator.get_echo_influence(domain, tool_name)
if echo_score > 0:
    decoder.add_vote("cross_modal", echo_score, confidence=0.2)
```

### 3. Continuous Calibration affects Plasticity

When the calibration engine detects significant drift, it should adjust the
plasticity system's learning rates and critical period modulator.

```
ContinuousCalibrationEngine.on_turn_complete(...)
  -> DriftDetector fires: quality declining
  -> PlasticityManager gets signal to increase learning rate temporarily
  -> CriticalPeriodModulator: if drift is severe, re-open critical period

Specifically:
  if drift.significance > 0.7:
    plasticity.critical_period.reset()  # Re-enter critical period
    plasticity._homeostasis_interval = 5  # More frequent homeostasis
```

**Integration code (in PlasticityManager):**
```python
def on_drift_detected(self, drift_report: DriftReport) -> None:
    """Adjust plasticity parameters in response to detected drift."""
    if drift_report.significance > 0.7:
        # Severe drift: re-enter critical period for faster adaptation
        self.critical_period.reset()
        self._homeostasis_interval = max(5, self._homeostasis_interval - 2)
    elif drift_report.significance > 0.4:
        # Moderate drift: increase learning rates temporarily
        self.weights.lr.behavioral *= 1.2
        self.weights.lr.tool_preference *= 1.2
```

### 4. Prediction-Calibration Feedback Loop

The calibration engine monitors prediction accuracy. When accuracy drops,
it signals the proactive prediction engine to widen its prediction distribution
(less confident predictions, consider more alternatives).

```
ContinuousCalibrationEngine.deep_calibrate()
  -> prediction accuracy = 0.3 (poor)
  -> ProactivePredictionEngine adjusts:
     - Reduce chain cache minimum_occurrences (consider rarer patterns)
     - Increase trajectory model's unigram_weight (less reliance on specific sequences)
     - Lower pre-warming confidence threshold
```

### 5. Adaptation feeds Cross-Modal Association

When the Adaptation engine detects that a signal has been habituated in one domain,
the Cross-Modal Associator should note this. A habituated pattern in domain A should
not be re-introduced via echo from a related domain B.

```
AdaptationFilter: signal "brevity" habituated in "coding" domain
CrossModalAssociator: suppress echo of "brevity" signal to "debugging" domain
```

---

## New Brain Pattern Proposals

### Proposal 1: Neuromodulatory System (Dopamine / Norepinephrine / Serotonin)

**Neuroscience Basis:**
The brain's behavior is globally modulated by diffuse neuromodulatory systems:
- **Dopamine**: Reward prediction error, motivation, reinforcement learning
- **Norepinephrine**: Arousal, urgency, fight-or-flight, attentional focus
- **Serotonin**: Patience, long-term planning, impulse control

These are NOT specific to any neural circuit -- they modulate the ENTIRE brain's behavior.
A dopamine surge makes ALL learning stronger. Norepinephrine makes ALL processing faster
but less careful. Serotonin makes ALL decisions more patient.

**SDK Pattern: `NeuromodulatorSystem`**

```python
@dataclass
class NeuromodulatoryState:
    """Global modulatory state affecting all engine components."""
    dopamine: float = 0.5      # 0=apathetic, 1=highly motivated
    norepinephrine: float = 0.5  # 0=calm, 1=urgent/aroused
    serotonin: float = 0.5     # 0=impulsive, 1=patient/planful

class NeuromodulatorSystem:
    """
    Global modulator that affects ALL engine components simultaneously.

    Dopamine effects:
    - High: Increase learning rates, increase risk tolerance, increase initiative
    - Low: Decrease learning rates, decrease exploration, prefer safe options

    Norepinephrine effects:
    - High: Favor speed over quality, reduce explanation depth, increase autonomy
    - Low: Favor quality over speed, increase verification, more cautious

    Serotonin effects:
    - High: Favor long-term planning, increase patience, prefer thorough approaches
    - Low: Favor immediate gratification, shortcuts, impulsive tool selection
    """

    def update_from_context(self, surprise: SurpriseSignal, feedback: List[FeedbackSignal]) -> None:
        """Update neuromodulatory state based on current context."""
        # Positive surprise -> dopamine burst
        # User frustration -> norepinephrine spike
        # Successful long task -> serotonin increase
        ...

    def get_modulation_multipliers(self) -> Dict[str, float]:
        """Get multipliers for all engine parameters."""
        # Returns: learning_rate_mult, speed_quality_bias, planning_depth_mult, etc.
        ...
```

**Rationale:** Currently, corteX modulates individual components independently. A neuromodulatory
system provides coherent global state changes. When the user is frustrated (norepinephrine spike),
EVERYTHING should shift: faster responses, less explanation, more direct action. This creates
the kind of emergent coherent behavior seen in biological systems.

**Priority:** P1 -- Medium complexity, high impact on behavioral coherence.

### Proposal 2: Inhibitory Interneurons (Lateral Inhibition / Winner-Take-All)

**Neuroscience Basis:**
The brain is not all excitation. About 20% of cortical neurons are inhibitory.
Lateral inhibition creates contrast enhancement: the strongest signal suppresses
nearby weaker signals. This is how the brain achieves sharp decisions from
distributed noisy representations.

**SDK Pattern: `LateralInhibitionNetwork`**

```python
class LateralInhibitionNetwork:
    """
    When multiple tools/models/strategies compete for selection,
    the winning candidate actively INHIBITS the runners-up.

    This prevents the "tyranny of small differences" where
    two nearly-equal candidates oscillate. Once a winner emerges,
    it suppresses alternatives for this turn.

    Also implements:
    - Surround inhibition: Similar tools suppress each other more than
      dissimilar tools (code_interpreter suppresses code_sandbox more
      than it suppresses web_search)
    - Rebound inhibition: After prolonged suppression, a tool gets a
      brief excitatory rebound (prevents permanent neglect)
    """

    def compete(self, candidates: Dict[str, float]) -> Tuple[str, Dict[str, float]]:
        """
        Run winner-take-all competition.
        Returns: (winner, inhibited_scores)
        """
        ...

    def apply_surround_inhibition(self, winner: str, similarity_map: Dict[str, float]) -> None:
        """Inhibit similar candidates more than dissimilar ones."""
        ...
```

**Rationale:** The current PopulationDecoder aggregates votes but doesn't implement
competitive dynamics. Adding lateral inhibition creates sharper, more decisive tool
selection and prevents the system from "waffling" between similar alternatives.

**Priority:** P2 -- Medium complexity, medium impact. Depends on Population Coding and Functional Columns.

### Proposal 3: Place Cells / Grid Cells (Cognitive Map of Task Space)

**Neuroscience Basis:**
Place cells in the hippocampus fire when the organism is at a specific location.
Grid cells in the entorhinal cortex create a hexagonal coordinate system for navigation.
Together, they form a cognitive map that enables spatial reasoning: "Where am I?
Where have I been? How do I get to the goal?"

Recent research (Behrens et al., 2018) shows these systems also encode abstract
cognitive spaces -- not just physical locations. The brain navigates "concept space"
using the same neural machinery as physical space.

**SDK Pattern: `CognitiveNavigator`**

```python
@dataclass
class TaskSpacePosition:
    """Agent's current position in abstract task space."""
    coordinates: List[float]   # N-dimensional embedding
    domain: str
    complexity: float
    progress: float
    nearest_landmarks: List[str]  # Known successful positions

class CognitiveNavigator:
    """
    Navigates the abstract space of task types, user states, and agent capabilities.

    Place cells = specific task-state configurations that the agent recognizes
    Grid cells = a coordinate system for interpolating between known states

    Enables:
    1. "Where am I?" - Current position in task space
    2. "Have I been here before?" - Episodic recall (connects to MemoryFabric)
    3. "What's the shortest path to the goal?" - Planning via spatial analogy
    4. "I'm lost" - Detection of being in an unknown region of task space

    The cognitive map is built incrementally:
    - Each successful task completion places a "landmark" in task space
    - Navigation between landmarks uses learned shortcuts
    - Unknown regions trigger exploration behavior (increased creativity weight)
    """

    def locate(self, context: Dict[str, Any]) -> TaskSpacePosition:
        """Determine current position in task space."""
        ...

    def plan_route(self, current: TaskSpacePosition, goal: str) -> List[str]:
        """Plan a sequence of intermediate task-states to reach the goal."""
        ...

    def is_lost(self) -> bool:
        """Am I in an unmapped region of task space?"""
        ...
```

**Rationale:** The current GoalTracker tracks linear progress (0% to 100%), but real
problem-solving is spatial -- you navigate between states, backtrack, take shortcuts.
A cognitive map enables the agent to reason about WHERE it is in the problem space,
detect when it's "lost," and find paths through known territory. This is especially
valuable for multi-step tasks where the linear progress model breaks down.

**Priority:** P3 -- High complexity, high impact. Depends on Memory Fabric, GoalTracker, and Episodic Memory.

### Proposal 4: Mirror Neuron System (Learning from Observation)

**Neuroscience Basis:**
Mirror neurons fire both when an organism performs an action AND when it observes
another organism performing the same action. They enable learning by imitation
and understanding others' intentions.

**SDK Pattern: `MirrorLearningSystem`**

```python
class MirrorLearningSystem:
    """
    Learn from observing OTHER agents or sessions.
    When Agent A successfully solves a task, Agent B can "mirror"
    the successful strategy without having experienced it directly.

    Enables:
    - Multi-agent knowledge sharing within a deployment
    - Learning from demonstrated examples (few-shot from traces)
    - Transfer learning between agents serving different users

    The mirror signal is weaker than direct experience (like
    cross-modal echo), but provides a bootstrap mechanism.
    """
    ...
```

**Rationale:** In multi-agent deployments, each agent currently learns independently.
Mirror neurons enable agents to learn from each other's successes, dramatically
accelerating adaptation. This connects to the Tier 4 (Global) feedback system
but operates at the local deployment level.

**Priority:** P3 -- Medium complexity, high impact for multi-agent deployments. Depends on Episodic Memory and Cross-Modal Association.

---

## Implementation Roadmap

### Phase 1: P0 Proactive Prediction
**Estimated effort:** 3-4 days
**Files modified:** `corteX/engine/prediction.py`, `corteX/sdk.py`
**Dependencies:** None (builds on existing PredictionEngine)
**Tests:** `tests/test_proactive_prediction.py` (13+ test cases)

### Phase 2: P1 Cross-Modal Association
**Estimated effort:** 2-3 days
**Files created:** `corteX/engine/association.py`
**Files modified:** `corteX/engine/plasticity.py`, `corteX/sdk.py`
**Dependencies:** Existing PlasticityManager, WeightEngine
**Tests:** `tests/test_association.py` (13+ test cases)

### Phase 3: P1 Continuous Calibration
**Estimated effort:** 2-3 days
**Files created:** `corteX/engine/calibration.py`
**Files modified:** `corteX/sdk.py`
**Dependencies:** PredictionEngine, FeedbackEngine, WeightEngine
**Tests:** `tests/test_calibration.py` (13+ test cases)

### Phase 4: Cross-Pattern Integration
**Estimated effort:** 2 days
**Files modified:** Multiple engine files
**Dependencies:** All P0/P1 patterns complete
**Tests:** `tests/test_cross_pattern.py` (integration tests)

### Phase 5: P2 Patterns (Future)
**Estimated effort:** 5-7 days total
**Dependencies:** All P0/P1 patterns complete

---

## Mathematical Model Summary

### Proactive Prediction

| Formula | Description |
|---|---|
| P(next) = w1*P_uni + w2*P_bi + w3*P_tri | Blended n-gram prediction |
| confidence(chain) = n/(n+k) * success_rate * decay | Bayesian chain confidence |
| H = -sum(p * log2(p)) | Trajectory entropy |
| decay = exp(-0.693 * age / halflife) | Time-based chain decay |

### Cross-Modal Association

| Formula | Description |
|---|---|
| echo_delta = delta * similarity * echo_decay | Echo transfer strength |
| sim = alpha*prior + (1-alpha)*learned | Blended similarity |
| sim_learned = co_occ/(count_a+count_b)*2 * success/(success+failures+k) | Learned similarity |

### Continuous Calibration

| Formula | Description |
|---|---|
| EWMA_t = lambda*x_t + (1-lambda)*EWMA_{t-1} | Exponentially weighted moving average |
| UCL = mean + L*sigma*sqrt(lambda/(2-lambda)) | Upper control limit |
| w_start = w_end + momentum*drift (if \|drift\| > threshold) | Session-start momentum |
| w_start = w_end*(1-r) + w_prior*r (otherwise) | Session-start regression |

---

## Appendix: Existing Engine Summary

For reference, the implemented engine modules and their brain analogues:

| Module | Brain Analogue | File |
|---|---|---|
| WeightEngine (7 categories) | Synaptic weights across brain regions | `engine/weights.py` |
| PredictionEngine (reactive) | Cerebellum + predictive coding | `engine/prediction.py` |
| PlasticityManager | Synaptic plasticity (LTP/LTD/Hebbian) | `engine/plasticity.py` |
| FeedbackEngine (4 tiers) | Amygdala/Hippocampus/PFC/Collective | `engine/feedback.py` |
| AdaptationFilter | Sensory adaptation (Meissner/Merkel) | `engine/adaptation.py` |
| PopulationDecoder | Motor cortex population coding | `engine/population.py` |
| GoalTracker | ACC + Hippocampus (deja vu) | `engine/goal_tracker.py` |
| MemoryFabric | Working/Episodic/Semantic memory | `engine/memory.py` |
| Orchestrator | Central nervous system routing | `runtime/orchestrator.py` |

With the P0/P1 additions, the engine gains:

| New Module | Brain Analogue | File |
|---|---|---|
| ProactivePredictionEngine | Cortical prediction (goalkeeper) | `engine/prediction.py` (enhanced) |
| CrossModalAssociator | Cross-modal integration (touch->vision) | `engine/association.py` (new) |
| ContinuousCalibrationEngine | BCI adaptive algorithm | `engine/calibration.py` (new) |

---

*"This machine imagines all the time. Sometimes it succeeds in imagining well,
and if not, it updates its imagination and works like this all the time."*
*-- Prof. Idan Segev*
