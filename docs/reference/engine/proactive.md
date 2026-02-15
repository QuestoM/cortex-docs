# Proactive

`corteX.engine.proactive` -- Proactive prediction engine that anticipates the next user turn before any action.

---

## Overview

"The goalkeeper imagines what will happen in the next moment. He fantasizes about the future.
This machine constantly imagines." (Prof. Segev, Lecture 4)

While the reactive `PredictionEngine` predicts action outcomes, this module is PROACTIVE --
it predicts what the user will do next BEFORE any action, enabling speculative pre-warming
of tools, models, and context.

Architecture:

- **ConversationTurn** -- Featurized turn for trajectory prediction
- **NextTurnPrediction** -- Prediction of what the user will ask next
- **ConversationTrajectoryModel** -- Variable-order Markov chain (n=1,2,3) with Bayesian tracking
- **PredictionChainCache** -- Learned sequence patterns (hippocampal sequence completion)
- **PreWarmingScheduler** -- Speculative pre-loading of resources (Readiness Potential analog)
- **ProactivePredictionEngine** -- Main orchestrator wrapping all components

---

## Dataclass: ConversationTurn

A single featurized turn for trajectory prediction.

| Field | Type | Description |
|-------|------|-------------|
| `turn_id` | `str` | Unique turn identifier. |
| `task_type` | `str` | Category: coding, debugging, research, conversation, etc. |
| `tools_used` | `List[str]` | Tools invoked during this turn. |
| `model_used` | `str` | LLM model used. |
| `topic_hash` | `str` | Coarse hash for sequence matching. |
| `complexity` | `float` | Task complexity [0.0, 1.0]. |
| `user_message_length` | `int` | Length of user's message. |

## Dataclass: NextTurnPrediction

| Field | Type | Description |
|-------|------|-------------|
| `predicted_task_type` | `str` | Predicted task category. |
| `confidence` | `float` | Prediction confidence [0.0, 1.0]. |
| `predicted_tools` | `List[str]` | Tools likely needed. |
| `predicted_model` | `str` | Model likely needed. |
| `predicted_complexity` | `float` | Expected complexity. |
| `prediction_basis` | `str` | Why this prediction was made. |
| `pre_warm_actions` | `List[str]` | Specific pre-warming actions to take. |
| `chain_id` | `Optional[str]` | If prediction came from a chain match. |

---

## Class: ConversationTrajectoryModel

Variable-order Markov chain (unigram, bigram, trigram) with Bayesian confidence tracking
and Gamma-distributed timing predictions.

### Constructor

```python
ConversationTrajectoryModel(max_history: int = 200)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record_turn` | `(turn: ConversationTurn) -> None` | Record a turn and update all n-gram counts and Bayesian trackers. |
| `predict_next` | `(top_n: int = 5) -> List[Tuple[str, float]]` | Predict next task type. Weighted combination: 20% unigram + 50% bigram + 30% trigram. |
| `predict_timing` | `(task_type: str) -> float` | Expected inter-turn time (ms) from Gamma posterior. |
| `get_transition_confidence` | `(from_type, to_type) -> float` | Beta posterior mean for a specific transition. |
| `get_trajectory_entropy` | `() -> float` | Shannon entropy of predicted distribution. Low = predictable. |
| `decay_all` | `(factor: float = 0.99) -> None` | Temporal decay on all Bayesian trackers. |

---

## Class: PredictionChainCache

Stores learned conversation chains (e.g., `[coding, debugging, testing] -> documentation`).
Confidence = `occurrences / (occurrences + 3) * success_rate * time_decay`.

### Constructor

```python
PredictionChainCache(
    max_chain_length: int = 5,
    min_occurrences: int = 2,
    max_chains: int = 500,
    decay_halflife_hours: float = 24.0,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record_sequence` | `(task_types: List[str]) -> None` | Extract all sub-chains from a sequence. |
| `match` | `(recent_types: List[str]) -> Optional[Tuple[PredictionChain, float]]` | Find best chain match. Longer chains get length bonus. |
| `update_accuracy` | `(chain_id: str, was_correct: bool) -> None` | Update chain accuracy via EMA (alpha=0.2). |

---

## Class: PreWarmingScheduler

Speculative pre-loading. Cost budget scales with confidence: high-confidence predictions
can trigger expensive actions (memory pre-fetch), low-confidence only cheap actions (model
selection). Wrong predictions are discarded with no side effects.

### Constructor

```python
PreWarmingScheduler(max_cost_budget: float = 2.0, min_confidence: float = 0.2)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `pre_warm` | `async (prediction: NextTurnPrediction) -> Dict[str, Any]` | Execute async pre-warming. |
| `pre_warm_sync` | `(prediction: NextTurnPrediction) -> Dict[str, Any]` | Synchronous pre-warming. |
| `get_pre_warmed` | `() -> Dict[str, Any]` | Retrieve pre-warmed resources. |
| `invalidate` | `() -> None` | Discard all pre-warmed resources. |
| `was_prediction_useful` | `(actual_task_type: str) -> bool` | Check if prediction matched; updates hit counter. |
| `get_hit_rate` | `() -> float` | Pre-warming hit rate. |

---

## Class: ProactivePredictionEngine

Main orchestrator wrapping trajectory model, chain cache, and pre-warming scheduler.

### Constructor

```python
ProactivePredictionEngine(
    max_history: int = 200,
    max_chains: int = 500,
    pre_warm_budget: float = 2.0,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record_turn` | `(turn: ConversationTurn) -> None` | Record a turn. Updates trajectory, learns associations, records chains, verifies previous prediction. |
| `predict_next_turn` | `() -> NextTurnPrediction` | Generate proactive prediction. Combines chain match (70%) with trajectory model (30%), modulated by historical accuracy and external surprise. |
| `schedule_pre_warming` | `async (prediction) -> Dict[str, Any]` | Schedule async pre-warming if confidence > 0.3. |
| `schedule_pre_warming_sync` | `(prediction) -> Dict[str, Any]` | Synchronous pre-warming. |
| `get_pre_warmed_resources` | `() -> Dict[str, Any]` | Retrieve pre-warmed resources. |
| `set_external_surprise` | `(surprise: float) -> None` | Receive external surprise signal. High surprise dampens proactive confidence. |
| `verify_prediction` | `(actual_task_type: str) -> bool` | Check if current prediction matches. |
| `get_prediction_accuracy` | `() -> float` | Rolling accuracy from Beta posterior. |
| `get_prediction_confidence_interval` | `() -> Tuple[float, float]` | 95% credible interval for accuracy. |
| `get_stats` | `() -> Dict[str, Any]` | Comprehensive statistics. |
| `to_dict` / `from_dict` | | Serialization for cross-session persistence. |

---

## Example

```python
from corteX.engine.proactive import ProactivePredictionEngine, ConversationTurn

proactive = ProactivePredictionEngine()

# Record some conversation turns
turns = [
    ConversationTurn("t1", "coding", ["code_interpreter"], "gemini-3-pro",
                     "abc", 0.7, 150),
    ConversationTurn("t2", "debugging", ["code_interpreter", "bash"], "gemini-3-pro",
                     "def", 0.8, 200),
    ConversationTurn("t3", "testing", ["code_interpreter"], "gemini-3-flash",
                     "ghi", 0.5, 100),
]
for turn in turns:
    proactive.record_turn(turn)

# Predict what comes next
prediction = proactive.predict_next_turn()
print(f"Predicted: {prediction.predicted_task_type}")
print(f"Confidence: {prediction.confidence:.2f}")
print(f"Basis: {prediction.prediction_basis}")
print(f"Pre-warm: {prediction.pre_warm_actions}")

# Pre-warm resources
warmed = proactive.schedule_pre_warming_sync(prediction)
print(f"Pre-warmed: {warmed}")

# After the actual turn arrives, verify
was_correct = proactive.verify_prediction("documentation")
print(f"Prediction correct: {was_correct}")
print(f"Overall accuracy: {proactive.get_prediction_accuracy():.2f}")
```
