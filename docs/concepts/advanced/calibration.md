# Continuous Calibration

The Continuous Calibration Engine ensures that the agent's confidence estimates are accurate. When the agent says it is 80% confident, it should be correct 80% of the time. The engine tracks calibration error, applies Platt scaling corrections, and monitors for metacognitive failures.

## What It Does

The calibration engine:

1. **Tracks prediction accuracy** across confidence bins (Expected Calibration Error)
2. **Adjusts confidence** using Platt scaling to correct systematic biases
3. **Monitors metacognition** for oscillation, stagnation, and degradation patterns

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Metacognition"
    Metacognition -- "thinking about thinking" -- is one of the brain's most sophisticated capabilities. The prefrontal cortex monitors the accuracy of its own predictions and adjusts confidence accordingly.

    Well-calibrated confidence is essential for good decision-making. Overconfidence leads to reckless actions; underconfidence leads to paralysis. The brain's metacognitive circuits continuously calibrate: "I thought I was sure, but I was wrong, so I should be less sure next time."

    Studies on "feeling of knowing" (FOK) show that humans are reasonably well-calibrated for familiar domains but poorly calibrated for novel ones -- matching the critical period modulator's behavior in corteX.

## How It Works

### Calibration Tracking

The `CalibrationTracker` bins predictions by confidence level and tracks accuracy in each bin:

```python
from corteX.engine.calibration import ContinuousCalibrationEngine

engine = ContinuousCalibrationEngine()

# Record predictions and outcomes
engine.record(confidence=0.9, correct=True)
engine.record(confidence=0.9, correct=True)
engine.record(confidence=0.9, correct=False)
# Bin [0.8-0.9]: 2/3 = 67% accuracy, but predicted 90% -> overconfident

engine.record(confidence=0.3, correct=False)
engine.record(confidence=0.3, correct=True)
# Bin [0.2-0.3]: 1/2 = 50% accuracy, but predicted 30% -> underconfident
```

### Expected Calibration Error (ECE)

ECE measures the average gap between predicted confidence and actual accuracy:

```python
report = engine.get_report()
# CalibrationReport with:
#   ece: 0.15          # Average miscalibration
#   max_calibration_error: 0.23  # Worst bin
#   bin_details: [...]  # Per-bin statistics
```

An ECE of 0.0 means perfect calibration. An ECE above 0.15 suggests systematic bias.

### Platt Scaling

The `ConfidenceAdjuster` applies Platt scaling to correct systematic overconfidence or underconfidence:

```python
from corteX.engine.calibration import ConfidenceAdjuster

adjuster = ConfidenceAdjuster()

# After enough observations, the adjuster learns a correction
adjusted = adjuster.adjust(raw_confidence=0.9)
# If the system is overconfident, adjusted might be 0.75
```

### Metacognition Monitor

The `MetaCognitionMonitor` detects three pathological patterns:

```python
from corteX.engine.calibration import MetaCognitionMonitor

monitor = MetaCognitionMonitor()

# Oscillation: confidence swings wildly between turns
# Stagnation: confidence never changes despite varied outcomes
# Degradation: calibration error trending upward over time

alerts = monitor.check()
# ["oscillation_detected", "degradation_trend"]
```

## When It Activates

- **After every prediction**: Confidence and outcome are recorded
- **Before decisions**: Raw confidence is adjusted via Platt scaling
- **Periodically**: ECE is computed and metacognition monitor checks for pathologies
- **At session boundaries**: Calibration report informs whether the system needs retuning

## API Reference

```python
from corteX.engine.calibration import (
    ContinuousCalibrationEngine,
    CalibrationTracker,
    ConfidenceAdjuster,
    MetaCognitionMonitor,
    CalibrationReport,
)
```
