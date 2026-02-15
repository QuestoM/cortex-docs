# Population

`corteX.engine.population` -- Population coding and ensemble quality estimation.

---

## Overview

"The code is distributed across the network... each one carries a little bit of information...
but the collective represents something in the world." (Prof. Segev)

Instead of relying on a single LLM response or tool result, aggregate signals from MULTIPLE
lightweight evaluations. No single evaluation is trusted alone -- the ensemble creates
robustness, like the ~200 motor cortex neurons that collectively encode movement direction
via the population vector.

Architecture:

- **Vote** -- A single evaluator's signal (value + confidence)
- **PopulationVector** -- The decoded consensus (value, confidence, agreement)
- **EvaluatorResult** -- Result from an evaluator function
- **PopulationDecoder** -- Aggregates weak signals into one strong decision
- **PopulationToolSelector** -- Population coding for tool selection
- **PopulationQualityEstimator** -- Ensemble quality estimation with built-in heuristics

---

## Dataclass: Vote

A single vote from one evaluator.

| Field | Type | Description |
|-------|------|-------------|
| `voter_id` | `str` | Identifier of the evaluator. |
| `value` | `float` | Signal value [0.0, 1.0]. |
| `confidence` | `float` | Voter's confidence [0.0, 1.0]. |
| `metadata` | `Dict[str, Any]` | Optional metadata. |

## Dataclass: PopulationVector

The decoded population signal.

| Field | Type | Description |
|-------|------|-------------|
| `value` | `float` | Consensus value (confidence-weighted average). |
| `confidence` | `float` | Population confidence (avg_confidence * agreement). |
| `agreement` | `float` | Voter unanimity [0.0, 1.0]. 0 = chaos, 1 = unanimous. |
| `voter_count` | `int` | Total voters. |
| `outlier_count` | `int` | Voters suppressed as outliers. |
| `votes` | `List[Vote]` | All individual votes. |

## Dataclass: EvaluatorResult

| Field | Type | Description |
|-------|------|-------------|
| `score` | `float` | Evaluation score [0.0, 1.0]. |
| `confidence` | `float` | Evaluator confidence [0.0, 1.0]. |
| `label` | `str` | Optional label. |
| `details` | `Dict[str, Any]` | Optional details. |

---

## Class: PopulationDecoder

Aggregates multiple weak signals into one strong decision.

### Constructor

```python
PopulationDecoder(
    outlier_threshold: float = 2.0,   # Z-score for outlier suppression
    min_confidence: float = 0.1,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `add_vote` | `(voter_id: str, value: float, confidence: float = 0.5, metadata: Optional[Dict] = None) -> None` | Add a manual vote. |
| `register_evaluator` | `(name: str, evaluator: Evaluator) -> None` | Register an evaluator function for `evaluate()`. |
| `evaluate` | `(**kwargs) -> PopulationVector` | Run all registered evaluators, collect votes, and decode. Failed evaluators get low-confidence neutral votes. |
| `decode` | `() -> PopulationVector` | Decode the population vector from collected votes. Algorithm: (1) compute stats, (2) suppress outliers (Z > threshold), (3) confidence-weighted average, (4) measure agreement. |
| `clear` | `() -> None` | Clear votes, keep evaluators. |
| `reset` | `() -> None` | Clear everything. |

### Decoding Algorithm

1. Compute mean and standard deviation of all vote values
2. Identify outliers (votes deviating > `outlier_threshold` standard deviations from mean)
3. Suppress outlier confidence by 80% (do not remove)
4. Compute confidence-weighted average as the population vector value
5. Measure agreement: `1.0 - sqrt(weighted_variance) * 2` (clamped to [0, 1])
6. Overall confidence = mean_confidence * agreement

---

## Class: PopulationToolSelector

Uses population coding to select the best tool for a task. Multiple evaluators vote
on each candidate tool, and the tool with the highest population vector value wins.

### Constructor

```python
PopulationToolSelector()
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `add_evaluator` | `(name: str, evaluator: Callable) -> None` | Add a tool evaluator. Signature: `(tool_name: str, **context) -> EvaluatorResult`. |
| `select` | `(candidates: List[str], **context) -> Tuple[str, PopulationVector]` | Select the best tool. Returns `(tool_name, population_vector)`. |

---

## Class: PopulationQualityEstimator

Estimates response quality using population coding with built-in heuristics. Replaces
hardcoded quality values with actual ensemble estimation.

### Built-in Heuristics

| Name | What it Checks | Confidence |
|------|---------------|------------|
| `length` | Word count (short=low, medium=good, long=good) | 0.3 |
| `completeness` | Ends with proper punctuation or code block | 0.5 |
| `error_check` | Presence of error phrases ("I'm sorry", "I cannot", etc.) | 0.6 |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `add_heuristic` | `(name: str, heuristic: Callable) -> None` | Add a custom quality heuristic. Signature: `(response: str, **context) -> EvaluatorResult`. |
| `estimate` | `(response: str, **context) -> PopulationVector` | Estimate quality using all registered heuristics. Returns a PopulationVector. |

---

## Example

```python
from corteX.engine.population import PopulationDecoder, PopulationQualityEstimator

# Manual voting
decoder = PopulationDecoder()
decoder.add_vote("weight_engine", 0.8, confidence=0.9)
decoder.add_vote("pattern_match", 0.7, confidence=0.6)
decoder.add_vote("user_history", 0.9, confidence=0.5)
result = decoder.decode()
print(f"Value: {result.value:.3f}, Confidence: {result.confidence:.3f}")
print(f"Agreement: {result.agreement:.3f}, Outliers: {result.outlier_count}")
```

```python
# Quality estimation
estimator = PopulationQualityEstimator()

quality = estimator.estimate(
    response="Here's the implementation with full error handling and tests..."
)
print(f"Quality: {quality.value:.3f}, Confidence: {quality.confidence:.3f}")

# With a poor response
quality2 = estimator.estimate(response="I'm sorry, I cannot help with that.")
print(f"Quality: {quality2.value:.3f}")  # Lower score
```

```python
# Tool selection
from corteX.engine.population import PopulationToolSelector, EvaluatorResult

selector = PopulationToolSelector()
selector.add_evaluator("relevance", lambda tool_name, **ctx:
    EvaluatorResult(score=0.9 if "code" in tool_name else 0.3, confidence=0.7))

best_tool, vector = selector.select(
    candidates=["code_interpreter", "web_search", "calculator"],
    query="Fix the authentication bug",
)
print(f"Best tool: {best_tool} (score={vector.value:.3f})")
```
