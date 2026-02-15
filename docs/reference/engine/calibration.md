# Calibration

`corteX.engine.calibration` -- Continuous calibration system with ECE tracking, Platt scaling, and meta-cognition monitoring.

---

## Overview

Brain-inspired self-monitoring: the system tracks HOW WELL it is tracking reality, and
adjusts its own confidence accordingly. Based on Prof. Idan Segev's insight that "the brain
needs to re-learn itself -- this is REAL-TIME learning."

The calibration system has six components:

- **CalibrationDomain** -- Enum of independently tracked domains
- **CalibrationBin** -- Probability bins for predicted vs actual frequency
- **CalibrationTracker** -- Multi-domain Expected Calibration Error (ECE) tracking
- **ConfidenceAdjuster** -- Platt scaling to correct over/under-confidence
- **MetaCognitionMonitor** -- Meta-level "am I learning correctly?" monitoring
- **ContinuousCalibrationEngine** -- Coordinator wrapping all components

Constants: `NUM_BINS=10`, `MIN_OBS_PER_BIN=5`, `ECE_ALARM_THRESHOLD=0.15`, `MAX_BIN_HISTORY=2000`.

---

## Enum: CalibrationDomain

```python
from corteX.engine.calibration import CalibrationDomain
```

| Value | String | Description |
|-------|--------|-------------|
| `TOOL_SUCCESS` | `"tool_success"` | Prediction accuracy for tool call outcomes. |
| `MODEL_QUALITY` | `"model_quality"` | LLM response quality predictions. |
| `LATENCY` | `"latency"` | Response time predictions. |
| `GOAL_PROGRESS` | `"goal_progress"` | Goal completion confidence. |
| `USER_SATISFACTION` | `"user_satisfaction"` | User satisfaction predictions. |

---

## Dataclass: CalibrationBin

A single probability bin covering `[lower, upper)`. Accumulates predicted probabilities
and actual outcomes. ECE contribution = `|avg_predicted - actual_frequency| * (count / total)`.

### Attributes

| Name | Type | Description |
|------|------|-------------|
| `lower` | `float` | Lower bound of the bin range. |
| `upper` | `float` | Upper bound of the bin range. |
| `predicted_sum` | `float` | Sum of predicted probabilities recorded. |
| `actual_outcomes` | `list` | List of boolean actual outcomes. |
| `count` | `int` | Number of observations in this bin. |

### Properties

| Name | Type | Description |
|------|------|-------------|
| `avg_predicted` | `float` | Mean predicted probability in this bin. |
| `actual_frequency` | `float` | Fraction of outcomes that were `True`. |
| `is_calibrated` | `bool` | `True` when `count >= 5`. |
| `calibration_error` | `float` | `|avg_predicted - actual_frequency|`. |

### Methods

```python
def record(self, predicted_probability: float, actual_outcome: bool) -> None
def to_dict(self) -> Dict[str, Any]
@classmethod def from_dict(cls, d: Dict[str, Any]) -> CalibrationBin
```

---

## Class: CalibrationTracker

Tracks prediction accuracy across multiple domains using 10 bins per domain.
ECE = sum_b((n_b / N) * |avg_predicted_b - actual_freq_b|). Alarm when ECE > 0.15.

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record` | `(domain: CalibrationDomain, predicted: float, actual: bool) -> None` | Record a prediction-outcome pair in the appropriate bin. |
| `compute_ece` | `(domain: CalibrationDomain) -> float` | Compute ECE for one domain. |
| `compute_all_ece` | `() -> Dict[CalibrationDomain, float]` | ECE for all domains. |
| `update_ece_history` | `() -> Dict[CalibrationDomain, float]` | Snapshot ECE values into history. |
| `get_ece_trend` | `(domain: CalibrationDomain, window: int = 10) -> float` | Linear regression slope of ECE. Positive = degrading. |
| `is_alarm_triggered` | `(domain: CalibrationDomain) -> bool` | `True` if ECE > 0.15. |
| `get_alarming_domains` | `() -> List[CalibrationDomain]` | All domains currently in alarm. |
| `get_domain_stats` | `(domain: CalibrationDomain) -> Dict[str, Any]` | Full statistics including per-bin breakdown. |
| `to_dict` / `from_dict` | | Serialization. |

---

## Class: ConfidenceAdjuster

Platt scaling: `calibrated_p = sigmoid(a * raw_p + b)`. Parameters `(a, b)` learned per
domain via gradient descent on calibration bin summaries.

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `adjust` | `(domain: CalibrationDomain, raw_confidence: float) -> float` | Apply Platt scaling. |
| `fit_from_tracker` | `(tracker, domain, lr=0.1, iterations=20) -> Tuple[float, float]` | Learn `(a, b)` via gradient descent on MSE. |
| `fit_all_domains` | `(tracker, lr=0.1, iterations=20) -> Dict` | Fit all domains. |
| `get_adjustment_summary` | `(domain) -> Dict[str, Any]` | Direction (deflating/inflating/identity) and sample adjustments. |
| `to_dict` / `from_dict` | | Serialization. |

---

## Class: MetaCognitionMonitor

Meta-level "am I learning correctly?" monitor. Produces `MetaCognitionAlert` instances.

### Constructor

```python
MetaCognitionMonitor(
    oscillation_threshold: float = 0.3,
    stagnation_threshold: float = 0.02,
    history_window: int = 20,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record_weight_delta` | `(domain, delta) -> None` | Record a weight update delta. |
| `check_oscillation` | `(domain) -> Optional[MetaCognitionAlert]` | Detect >60% sign flips. Recommends reducing LR. |
| `check_stagnation` | `(domain) -> Optional[MetaCognitionAlert]` | Detect near-zero deltas. Recommends increasing LR. |
| `check_calibration_degradation` | `(tracker, domain) -> Optional[MetaCognitionAlert]` | Detect upward ECE trend. |
| `run_full_check` | `(tracker) -> List[MetaCognitionAlert]` | All checks across all domains. |
| `get_recommended_lr_factor` | `(domain) -> float` | LR multiplier from most recent alert. |

### Dataclass: MetaCognitionAlert

| Field | Type | Description |
|-------|------|-------------|
| `alert_type` | `str` | `"oscillation"`, `"stagnation"`, or `"degradation"`. |
| `domain` | `CalibrationDomain` | Affected domain. |
| `severity` | `float` | 0.0 to 1.0. |
| `message` | `str` | Human-readable explanation. |
| `recommended_lr_factor` | `float` | Suggested LR multiplier. |

---

## Class: ContinuousCalibrationEngine

Main coordinator. Auto-triggers calibration cycles every `calibration_interval` observations.

### Constructor

```python
ContinuousCalibrationEngine(calibration_interval: int = 20)
```

### Attributes

| Name | Type | Description |
|------|------|-------------|
| `tracker` | `CalibrationTracker` | ECE tracker. |
| `adjuster` | `ConfidenceAdjuster` | Platt scaling adjuster. |
| `monitor` | `MetaCognitionMonitor` | Meta-cognition monitor. |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record_outcome` | `(domain, predicted_probability, actual_outcome) -> None` | Record a pair. Auto-triggers calibration cycle. |
| `adjust_confidence` | `(domain, raw_confidence) -> float` | Platt-scaled confidence. |
| `calibration_cycle` | `() -> List[MetaCognitionAlert]` | Update ECE, refit Platt, run meta-cognition. |
| `get_lr_factor` | `(domain) -> float` | Recommended LR multiplier. |
| `report` | `() -> Dict[str, Any]` | Full JSON calibration report with health status. |
| `report_text` | `() -> str` | Human-readable text report. |
| `get_stats` | `() -> Dict[str, Any]` | Summary: observations, ECE by domain, alarms, alerts. |
| `to_dict` / `from_dict` | | Serialization. |

---

## Example

```python
from corteX.engine.calibration import ContinuousCalibrationEngine, CalibrationDomain

cal = ContinuousCalibrationEngine()

# Record predictions and outcomes
for _ in range(25):
    cal.record_outcome(CalibrationDomain.TOOL_SUCCESS, 0.85, True)
    cal.record_outcome(CalibrationDomain.TOOL_SUCCESS, 0.90, False)

# Adjust a raw confidence using learned Platt parameters
adjusted = cal.adjust_confidence(CalibrationDomain.TOOL_SUCCESS, 0.85)
print(f"Raw: 0.85 -> Adjusted: {adjusted:.3f}")

# Run a manual calibration cycle
alerts = cal.calibration_cycle()
for alert in alerts:
    print(f"[{alert.alert_type}] {alert.domain.value}: {alert.message}")

# Generate a report
print(cal.report_text())
```
