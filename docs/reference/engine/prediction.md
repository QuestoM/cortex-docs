# Prediction Engine

`corteX.engine.prediction`

Before each action, predicts the outcome confidence, latency, and quality. After execution, compares actual results against predictions and computes a surprise signal. High surprise drives large weight updates (learning), while low surprise confirms calibration.

Inspired by predictive coding (Karl Friston's Free Energy Principle), dopamine reward prediction error, and the cerebellum as a prediction machine.

---

## OutcomeType

Enum for categorizing action outcomes.

```python
class OutcomeType(str, Enum):
    SUCCESS = "success"
    PARTIAL = "partial"
    FAILURE = "failure"
    TIMEOUT = "timeout"
    UNEXPECTED = "unexpected"
```

Outcomes are ranked for error computation: `FAILURE` (0) < `TIMEOUT` (1) < `UNEXPECTED` (2) < `PARTIAL` (3) < `SUCCESS` (4).

---

## Prediction

A prediction made before an action executes.

| Attribute | Type | Description |
|-----------|------|-------------|
| `prediction_id` | `str` | Unique identifier for matching to outcomes |
| `action_description` | `str` | Description of the planned action |
| `predicted_outcome` | `OutcomeType` | Expected outcome type |
| `confidence` | `float` | Confidence in the prediction (`0.0` to `1.0`) |
| `predicted_latency_ms` | `float` | Expected execution time in milliseconds |
| `predicted_quality` | `float` | Expected output quality (`0.0` to `1.0`) |
| `context_hash` | `str` | Hash for correlating predictions with context |
| `timestamp` | `float` | Unix timestamp (auto-populated) |
| `model_used` | `str` | Which LLM model is being used |
| `tool_used` | `str` | Which tool is being invoked |

---

## Outcome

The actual result after an action completes.

| Attribute | Type | Description |
|-----------|------|-------------|
| `prediction_id` | `str` | Matches the corresponding `Prediction` |
| `actual_outcome` | `OutcomeType` | What actually happened |
| `actual_latency_ms` | `float` | Actual execution time |
| `actual_quality` | `float` | Actual output quality (`0.0` to `1.0`) |
| `error_message` | `Optional[str]` | Error details if applicable |
| `timestamp` | `float` | Unix timestamp (auto-populated) |

---

## SurpriseSignal

The prediction error signal -- the core learning driver.

| Attribute | Type | Description |
|-----------|------|-------------|
| `prediction_id` | `str` | Links back to the prediction |
| `surprise_magnitude` | `float` | Absolute surprise (`0.0` to `1.0`) |
| `surprise_direction` | `float` | `-1.0` (worse) to `1.0` (better than expected) |
| `outcome_error` | `float` | Predicted vs actual outcome rank difference |
| `latency_error` | `float` | Predicted vs actual latency (log-scale) |
| `quality_error` | `float` | Actual minus predicted quality |
| `learning_signal_strength` | `float` | How much to update weights (`0.0` to `1.0`) |
| `explanation` | `str` | Human-readable surprise explanation |

---

## PredictionEngine

Manages predictions and computes surprise signals. Maintains running statistics per tool and model for increasingly accurate predictions over time.

### Constructor

```python
PredictionEngine()
```

### Methods

#### `predict(action: str, tool: str = "", model: str = "", context: Optional[Dict[str, Any]] = None) -> Prediction`

Generates a prediction for an upcoming action using historical tool statistics. Confidence grows logarithmically with the number of data points.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `action` | `str` | Description of the planned action |
| `tool` | `str` | Tool being used (for stats lookup) |
| `model` | `str` | LLM model being used |
| `context` | `Optional[Dict]` | Additional context |

**Returns:** A `Prediction` object stored internally until `compare()` is called.

#### `compare(prediction_id: str, actual_outcome: OutcomeType, actual_latency_ms: float, actual_quality: float, error_message: Optional[str] = None) -> SurpriseSignal`

Compares a prediction to the actual outcome and computes the surprise signal.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `prediction_id` | `str` | ID from the original `Prediction` |
| `actual_outcome` | `OutcomeType` | What actually happened |
| `actual_latency_ms` | `float` | Actual execution time |
| `actual_quality` | `float` | Assessed output quality |
| `error_message` | `Optional[str]` | Error details |

**Returns:** `SurpriseSignal` with magnitude, direction, and learning signal strength.

**Surprise computation:**

- Magnitude = `0.5 * |outcome_error| + 0.2 * |latency_error| + 0.3 * |quality_error|`
- Learning signal = `tanh(magnitude * confidence * 2)` -- small surprises suppressed, large ones amplified

#### `get_average_surprise() -> float`

Average surprise magnitude over the last 10 predictions. Used by `DualProcessRouter` to decide System 1 vs System 2 routing. `0.0` = predictions very accurate, `1.0` = constantly surprised.

#### `get_calibration_score() -> float`

How well-calibrated the predictions are overall. `0.0` = perfect (never surprised), `1.0` = terrible (always surprised).

#### `get_stats() -> Dict[str, Any]`

Returns: `total_predictions`, `calibration_error`, `tool_stats`, `pending_predictions`.

### Example

```python
from corteX.engine.prediction import PredictionEngine, OutcomeType

predictor = PredictionEngine()

# Before executing an action
prediction = predictor.predict(
    action="Execute code to fetch API data",
    tool="code_interpreter",
    model="gemini-flash",
)

# ... execute the action ...

# After execution, compute surprise
surprise = predictor.compare(
    prediction_id=prediction.prediction_id,
    actual_outcome=OutcomeType.SUCCESS,
    actual_latency_ms=1500,
    actual_quality=0.9,
)

# Use surprise to modulate learning
if surprise.learning_signal_strength > 0.3:
    print(f"Significant surprise: {surprise.explanation}")
    # Weight updates should be larger

print(f"Calibration: {predictor.get_calibration_score():.2f}")
```
