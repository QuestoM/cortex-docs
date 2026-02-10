# Adaptation Filter

The Adaptation Filter makes the system sensitive to *changes* rather than absolute values. Like biological sensory receptors that fire on change and go silent during constant stimulation, this filter amplifies novel signals and suppresses repetitive ones.

## What It Does

The Adaptation Filter wraps the Feedback Engine's output and modulates signal strength based on novelty:

- **First occurrence** of a signal: amplified (2x novelty bonus)
- **Repeated identical signals**: exponentially decayed
- **Prolonged constant signals**: fully habituated (ignored)
- **Change after steady state**: amplified (1.5x change bonus)

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Sensory Adaptation"
    "The nervous system is sensitive to CHANGES... a brake light that stays on forever, you'll ignore it. Changes are what matter." -- Prof. Idan Segev

    The nervous system has two types of sensory receptors with different adaptation profiles:

    - **Rapidly adapting receptors** (Meissner's corpuscles): Fire strongly on first contact, then go silent. Responsible for detecting texture, slip, and flutter. In corteX, this maps to the `RapidAdaptation` filter.

    - **Slowly adapting receptors** (Merkel's discs): Sustain response longer but eventually habituate completely. Responsible for detecting sustained pressure. In corteX, this maps to the `SustainedAdaptation` filter.

    The thalamus acts as a gateway, filtering out repetitive sensory information before it reaches the cortex. This is why you stop feeling your clothes after wearing them for a few minutes -- the signal hasn't changed, so the thalamus stops forwarding it. The Adaptation Filter implements this gating function for feedback signals.

## How It Works

### Rapid Adaptation

The `RapidAdaptation` filter implements exponential response decay for repeated identical signals:

```python
from corteX.engine.adaptation import RapidAdaptation

rapid = RapidAdaptation(decay_rate=0.5, novelty_bonus=2.0)

# First occurrence: novelty bonus (2.0x weight)
signal = rapid.filter("frustration", 0.6)
# adaptation_weight = 2.0, is_novel = True

# Second occurrence (same value): decayed
signal = rapid.filter("frustration", 0.6)
# adaptation_weight = 0.5^1 = 0.5

# Third occurrence: further decay
signal = rapid.filter("frustration", 0.6)
# adaptation_weight = 0.5^2 = 0.25

# VALUE CHANGES -> full reset
signal = rapid.filter("frustration", 0.9)
# adaptation_weight = 2.0 (novelty bonus again!)
```

### Sustained Adaptation (Habituation)

The `SustainedAdaptation` filter tracks how many times a signal has repeated and eventually habituates completely:

```python
from corteX.engine.adaptation import SustainedAdaptation

sustained = SustainedAdaptation(
    habituation_threshold=8,    # 8 repetitions to habituate
    recovery_time=300.0,        # 5 minutes to dishabituate
)

# Signals 1-7: gradual decay
for i in range(7):
    signal = sustained.filter("brevity", 0.3)
    # adaptation_weight: 1.0 -> 0.9 -> 0.8 -> ... -> 0.2

# Signal 8: HABITUATED - returns None
signal = sustained.filter("brevity", 0.3)
# None -- signal fully suppressed

# After 5 minutes of no signal: dishabituation
# signal = sustained.filter("brevity", 0.3)  # works again
```

This is critical for distinguishing user traits from user feedback:
- A user who *always* sends short messages is not signaling "brevity preference" -- that is their communication style
- A user who *switches* from long to short messages is sending a real signal

### Combined Filter

The `AdaptationFilter` combines both filters, using the most conservative (lowest) weight:

```python
from corteX.engine.adaptation import AdaptationFilter

adaptation = AdaptationFilter(
    rapid_decay=0.5,
    habituation_threshold=8,
    recovery_time=300.0,
)

# Process a feedback signal
result = adaptation.process("frustration", 0.7)
if result is not None:
    # Signal is worth processing
    effective_strength = 0.7 * result.adaptation_weight
else:
    # Signal habituated -- skip it
    pass
```

### Behavioral Shift Detection

The filter can detect significant deviations from a learned baseline:

```python
shift = adaptation.detect_behavioral_shift(
    signal_key="msg_length",
    current_value=0.9,    # Very long message
    window=5,
)
if shift is not None:
    # Significant shift from baseline detected
    # shift.change_magnitude tells you how big
    # shift.change_direction tells you which way
    pass
```

### Integration with the Feedback Engine

The Adaptation Filter sits between the Feedback Engine's signal detection and the Weight Engine's updates:

```python
from corteX.engine.feedback import FeedbackEngine
from corteX.engine.adaptation import AdaptationFilter

adaptation = AdaptationFilter()

# After Tier 1 detects signals:
for signal in feedback_signals:
    adapted = adaptation.process(signal.signal_type.value, signal.strength)
    if adapted:
        # Apply the adapted weight to the signal
        signal.strength *= adapted.adaptation_weight
    else:
        # Signal is habituated -- do not update weights
        pass
```

### Monitoring Habituated Signals

```python
stats = adaptation.get_stats()
# {
#   "total_signals_processed": 156,
#   "habituated_signals": ["brevity", "msg_length"],
#   "active_signals": ["frustration", "speed"],
#   "baselines": {"brevity": 0.3, "msg_length": 0.15, ...},
# }
```

### Session Management

At session boundaries, the filter resets repetition counts but preserves learned baselines:

```python
adaptation.new_session()
# Rapid adaptation: fully reset
# Sustained adaptation: reset repetition counts, keep baselines
# This means cross-session baselines persist while within-session
# habituation starts fresh
```

## When It Activates

- **Every feedback signal**: Before any signal updates weights, it passes through the adaptation filter
- **On change detection**: The filter amplifies signals that represent genuine behavioral shifts
- **Continuously**: Habituation state is updated on every processed signal
- **At session boundaries**: Repetition counts reset, baselines persist

## API Reference

```python
from corteX.engine.adaptation import (
    AdaptationFilter,
    RapidAdaptation,
    SustainedAdaptation,
    AdaptationState,
    ChangeSignal,
)
```
