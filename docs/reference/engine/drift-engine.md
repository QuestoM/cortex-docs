# Drift Detection Engine API Reference

## Module: `corteX.engine.drift_engine`

Five-signal fusion for goal drift scoring. Monitors goal relevance, token budget consumption, topic divergence, output quality trend, and prediction surprise. Produces graduated response levels from "continue" to "ask user."

Brain analogy: Anterior cingulate cortex (conflict monitoring), orbitofrontal cortex (value evaluation), default mode network (off-task drift detection).

---

## Classes

### `DriftSeverity`

**Type**: `str, Enum`

| Value | Score Range | Description |
|-------|-------------|-------------|
| `NONE` | < 0.1 | No drift detected |
| `LOW` | 0.1 - 0.3 | Minor divergence |
| `MODERATE` | 0.3 - 0.5 | Significant divergence |
| `HIGH` | 0.5 - 0.7 | Serious drift |
| `CRITICAL` | 0.7 - 0.85 | Critical drift |
| `EMERGENCY` | >= 0.85 | Emergency -- immediate intervention needed |

### `DriftAction`

**Type**: `str, Enum`

Graduated response actions mapped to severity levels.

| Value | Triggered By | Description |
|-------|-------------|-------------|
| `CONTINUE` | NONE, LOW | No action needed |
| `INJECT_REMINDER` | MODERATE | Inject goal reminder into context |
| `SUMMARIZE_REPLAN` | HIGH | Summarize progress and replan |
| `CHECKPOINT_RESET` | CRITICAL | Checkpoint and reset to known good state |
| `ASK_USER` | EMERGENCY | Ask user for direction |

### `DriftSignals`

**Type**: `@dataclass`

Raw signal values from all five detectors.

| Attribute | Type | Description |
|-----------|------|-------------|
| `goal_relevance` | `float` | Jaccard similarity to goal DNA [0.0, 1.0] |
| `token_budget_ratio` | `float` | Fraction of token budget consumed |
| `topic_divergence` | `float` | New entities not in goal [0.0, 1.0] |
| `output_quality_trend` | `float` | Quality decline signal [0.0, 1.0] |
| `prediction_surprise` | `float` | Accumulated negative surprise |

### `DriftAssessment`

**Type**: `@dataclass`

Complete drift assessment result.

| Attribute | Type | Description |
|-----------|------|-------------|
| `score` | `float` | Composite drift score [0.0, 1.0] |
| `signals` | `DriftSignals` | Raw signal values |
| `severity` | `DriftSeverity` | Classified severity |
| `recommended_action` | `DriftAction` | Recommended response |
| `explanation` | `str` | Human-readable explanation |
| `step_number` | `int` | Step where assessed |
| `timestamp` | `float` | Assessment timestamp |

---

### `DriftEngine`

Five-signal drift scoring engine with graduated response levels.

#### Constructor

```python
DriftEngine(
    goal: str,
    token_budget: int = 50000,
    history_window: int = 20,
    consecutive_drift_threshold: int = 3,
)
```

**Parameters**:

- `goal` (str): The goal text (used to create GoalDNA fingerprint)
- `token_budget` (int): Total token budget for the task. Default: 50000
- `history_window` (int): Steps to retain in history. Default: 20
- `consecutive_drift_threshold` (int): Consecutive low-similarity steps to boost score. Default: 3

#### Methods

##### `assess_step`

```python
def assess_step(
    self, description: str, output: str,
    tokens_used: int = 0, prediction_surprise: float = 0.0,
) -> DriftAssessment
```

Assess drift for a single step against the original goal. Computes all five signals, fuses them with weighted combination, applies consecutive drift bonus, and returns a complete `DriftAssessment`.

##### `record_negative_surprise`

```python
def record_negative_surprise(self, surprise_direction: float) -> None
```

Record additional negative surprise signal. Accumulates if surprise_direction < -0.3.

##### `get_drift_trend`

```python
def get_drift_trend(self, last_n: int = 5) -> float
```

Average drift score over recent assessments.

##### `get_stats`

Returns: step_number, tokens_consumed, token_budget, current_drift_trend, consecutive_low_similarity, total_assessments, surprise_accumulator.

---

## Signal Weights

| Signal | Weight | Description |
|--------|--------|-------------|
| Goal relevance | **35%** | Jaccard similarity with goal DNA |
| Token budget ratio | **15%** | Fraction of budget consumed |
| Topic divergence | **20%** | New entities not in goal |
| Output quality trend | **15%** | Quality decline detection |
| Prediction surprise | **15%** | Accumulated negative surprise |

**Consecutive drift bonus**: +0.15 added when low similarity persists for `consecutive_drift_threshold` steps.

---

## Usage Example

```python
from corteX.engine.drift_engine import DriftEngine

engine = DriftEngine("Build a REST API for user management", token_budget=50000)

# Assess each step
assessment = engine.assess_step(
    description="Creating user model",
    output="class User(Model): name = CharField()...",
    tokens_used=500,
    prediction_surprise=0.1,
)
print(f"Drift: {assessment.score:.2f} ({assessment.severity.value})")
print(f"Action: {assessment.recommended_action.value}")
print(f"Explanation: {assessment.explanation}")

# Check trend
trend = engine.get_drift_trend(last_n=5)
print(f"Recent drift trend: {trend:.2f}")
```

---

## See Also

- [Goal DNA API](./goal-dna.md)
- [Loop Detector API](./loop-detector.md)
- [Adaptive Budget API](./adaptive-budget.md)
