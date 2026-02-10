# The Brain Engine

The corteX Brain Engine is the adaptive core of the SDK. It gives AI agents the ability to learn from every interaction, predict what comes next, and continuously improve -- all without explicit retraining or cloud dependencies.

## Architecture Overview

The Brain Engine is composed of seven tightly integrated subsystems, each inspired by a specific mechanism in biological neural systems:

```
User Interaction
       |
       v
+------------------+     +------------------+     +------------------+
| Feedback Engine  | --> | Adaptation Filter| --> | Weight Engine     |
| (Implicit signal |     | (Change detection|     | (7-category       |
|  detection)      |     |  & habituation)  |     |  synaptic weights)|
+------------------+     +------------------+     +------------------+
       |                                                 |
       v                                                 v
+------------------+     +------------------+     +------------------+
| Plasticity       | --> | Prediction       | --> | Goal Tracker     |
| (LTP/LTD/        |     | (Predict-compare-|     | (Drift detection,|
|  homeostasis)    |     |  surprise)       |     |  loop prevention)|
+------------------+     +------------------+     +------------------+
```

## Design Principles

**1. No explicit feedback required.** The system infers user satisfaction from implicit signals -- corrections, frustration patterns, brevity shifts -- rather than asking "Was this response helpful?"

**2. Bayesian foundations.** Uncertainty is tracked with proper probability distributions, not point estimates. When the system is uncertain, it explores. When it is confident, it exploits. This is Thompson Sampling, rooted in conjugate priors.

**3. Homeostatic regulation.** No weight can run away to an extreme. Like the brain's mechanisms that prevent seizures (runaway excitation) and coma (runaway inhibition), the system gently pulls all weights back toward baseline.

**4. Change sensitivity.** The system is fundamentally sensitive to *changes*, not absolute values. A user who always sends short messages is not signaling "brevity preference" -- that is just their style. A user who *switches* from long to short messages is sending a strong signal.

**5. Surprise-driven learning.** When the system's predictions are wrong, it learns faster. When predictions are confirmed, learning slows. This mirrors prediction error signaling in the dopaminergic system.

## Subsystem Summary

| Subsystem | Brain Analogy | What It Does |
|-----------|--------------|--------------|
| [Weight Engine](weights.md) | Synaptic weights | Maintains 7 categories of adaptive weights with Bayesian priors |
| [Dual-Process Router](dual-process.md) | Anterior Cingulate Cortex | Routes decisions through fast (System 1) or slow (System 2) paths |
| [Goal Tracker](goal-tracking.md) | Prefrontal Cortex | Monitors goal alignment, detects drift, prevents loops |
| [Prediction Engine](prediction.md) | Dopaminergic system | Predicts outcomes, computes surprise, modulates learning rate |
| [Feedback Engine](feedback.md) | Amygdala + Hippocampus | Detects implicit user signals across 4 tiers |
| [Plasticity Manager](plasticity.md) | Synaptic plasticity | Implements LTP, LTD, homeostasis, and critical periods |
| [Adaptation Filter](adaptation.md) | Sensory receptors | Filters repetitive signals, amplifies novel changes |

## Quick Start

```python
from corteX.engine.weights import WeightEngine
from corteX.engine.feedback import FeedbackEngine
from corteX.engine.plasticity import PlasticityManager

# Initialize the brain
weights = WeightEngine()
feedback = FeedbackEngine(weights)
plasticity = PlasticityManager(weights)

# Process a user message (implicit feedback detection)
signals = feedback.process_user_message("No, that's wrong. I meant the other approach.")
# Detected: CORRECTION signal -> autonomy decreased, risk_tolerance decreased

# After a tool execution (plasticity rules fire)
events = plasticity.on_step_complete(
    tool="code_interpreter",
    task_type="coding",
    model="gemini-flash",
    success=True,
    quality=0.85,
)
# Hebbian learning strengthens code_interpreter<->coding association

# At session end (sleep-like consolidation)
weights.consolidate()
weights.save("~/.cortex/weights.json")
```

## When It Activates

The Brain Engine runs continuously during every agent interaction. It is not an optional module -- it is the foundation that every other corteX subsystem reads from and writes to:

- **Every turn**: Feedback Engine analyzes the user message. Adaptation Filter checks for changes. Weight Engine applies updates.
- **Every tool execution**: Plasticity rules fire (Hebbian, LTP, LTD). Tool preference weights update.
- **Every N interactions**: Homeostatic regulation runs to prevent weight drift.
- **Session boundaries**: Consolidation prunes noise, critical period resets, weights persist to disk.
