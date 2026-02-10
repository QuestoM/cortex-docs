# Population Coding

Population Coding aggregates multiple weak signals into one strong decision. Instead of relying on a single evaluation, the system runs parallel lightweight evaluators and decodes the ensemble consensus -- just as the motor cortex uses ~200 neurons to collectively encode movement direction.

## What It Does

For critical decisions (tool selection, quality estimation, model routing), the Population Decoder:

1. Collects votes from multiple evaluators, each with a confidence level
2. Suppresses outliers (votes that deviate too far from the population mean)
3. Computes a confidence-weighted average (the "population vector")
4. Measures agreement across voters (unanimity vs. chaos)

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Population Vectors"
    "The code is distributed across the network... each one carries a little bit of information... but the collective represents something in the world." -- Prof. Idan Segev

    In the motor cortex, no single neuron encodes movement direction. Instead, ~200 neurons each have a "preferred direction" and fire proportionally to how close the actual movement is to their preference. The population vector -- the confidence-weighted average of all preferred directions -- predicts actual arm movement with remarkable accuracy (Georgopoulos et al., 1986).

    This principle has a critical property: no single evaluator needs to be accurate. The ensemble is robust to individual errors, noise, and even systematic bias -- as long as the biases are diverse.

## How It Works

### Basic Population Decoding

```python
from corteX.engine.population import PopulationDecoder

decoder = PopulationDecoder(outlier_threshold=2.0)

# Collect votes from multiple evaluators
decoder.add_vote("weight_engine", value=0.8, confidence=0.9)
decoder.add_vote("pattern_match", value=0.7, confidence=0.6)
decoder.add_vote("user_history", value=0.9, confidence=0.5)
decoder.add_vote("random_noise", value=0.1, confidence=0.3)  # Outlier

# Decode the population vector
result = decoder.decode()
# PopulationVector(
#   value=0.81,        # Confidence-weighted average
#   confidence=0.72,   # Average confidence * agreement
#   agreement=0.85,    # How much voters agree
#   voter_count=4,
#   outlier_count=1,   # "random_noise" suppressed
# )
```

### Outlier Suppression

Votes that deviate more than `outlier_threshold` standard deviations from the mean are suppressed (confidence reduced to 20%):

```python
# Z-score > 2.0 -> outlier
# Outlier's confidence: original * 0.2
# The vote is not removed, just heavily down-weighted
```

### Tool Selection via Population Coding

```python
from corteX.engine.population import PopulationToolSelector, EvaluatorResult

selector = PopulationToolSelector()

# Register evaluators that score each candidate tool
selector.add_evaluator("weight_score", weight_based_scorer)
selector.add_evaluator("history", history_based_scorer)
selector.add_evaluator("relevance", relevance_scorer)

best_tool, vector = selector.select(
    candidates=["code_interpreter", "web_search", "calculator"],
    context={"query": "Calculate revenue growth"},
)
# best_tool = "calculator", vector.confidence = 0.78
```

### Quality Estimation via Population Coding

The `PopulationQualityEstimator` replaces hardcoded quality scores with an actual ensemble estimate:

```python
from corteX.engine.population import PopulationQualityEstimator

estimator = PopulationQualityEstimator()

# Built-in heuristics: length, completeness, error_check
quality = estimator.estimate("Here is the implementation...")
# PopulationVector(value=0.75, confidence=0.45, agreement=0.82)

# Add custom heuristics
def domain_heuristic(response: str, **kwargs) -> EvaluatorResult:
    has_code = "```" in response
    return EvaluatorResult(
        score=0.8 if has_code else 0.4,
        confidence=0.6,
        label="has_code",
    )

estimator.add_heuristic("domain", domain_heuristic)
```

### Registered Evaluator Functions

```python
decoder = PopulationDecoder()

# Register evaluator functions (called automatically)
decoder.register_evaluator("fast_check", fast_quality_check)
decoder.register_evaluator("pattern", pattern_matcher)

# Run all evaluators and decode
result = decoder.evaluate(context={"query": "..."})
# Failed evaluators automatically get low-confidence neutral votes (0.5, confidence=0.1)
```

## When It Activates

- **Before tool selection**: PopulationToolSelector aggregates evaluator votes to pick the best tool
- **After LLM response**: PopulationQualityEstimator scores response quality
- **During routing decisions**: Multiple evaluators vote on which model to use
- **For critical decisions**: Any decision that benefits from ensemble robustness

## API Reference

```python
from corteX.engine.population import (
    PopulationDecoder,
    PopulationVector,
    PopulationToolSelector,
    PopulationQualityEstimator,
    Vote,
    EvaluatorResult,
)
```
