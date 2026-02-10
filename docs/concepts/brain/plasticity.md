# Plasticity Manager

The Plasticity Manager implements brain-inspired learning rules that modify the Weight Engine based on outcomes. It coordinates Hebbian learning, Long-Term Potentiation (LTP), Long-Term Depression (LTD), homeostatic regulation, and critical period modulation.

## What It Does

After every step the agent takes, the Plasticity Manager evaluates the outcome and applies the appropriate learning rules to update weights. It answers the question: "Given what just happened, how should we adjust our preferences?"

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Synaptic Plasticity"
    Synaptic plasticity is the biological mechanism of learning and memory. Eric Kandel received the Nobel Prize (2000) for discovering its molecular basis in the sea slug *Aplysia*. The key mechanisms are:

    - **Hebbian Learning**: "Neurons that fire together, wire together" (Hebb, 1949). When two neurons are active simultaneously, their connection strengthens.
    - **Long-Term Potentiation (LTP)**: Repeated high-frequency stimulation of a synapse produces a lasting increase in synaptic strength. This is the cellular basis of skill acquisition.
    - **Long-Term Depression (LTD)**: Repeated low-frequency stimulation weakens a synapse. This is how the brain unlearns maladaptive patterns.
    - **Homeostatic Plasticity**: The brain prevents runaway excitation (seizures) and inhibition (coma) through global regulatory mechanisms that maintain firing rates within a target range.
    - **Critical Periods**: Early in development, the brain is maximally plastic. Language acquisition, visual cortex tuning, and social bonding all have critical windows where learning is accelerated.

    The Bienenstock-Cooper-Munro (BCM) theory unifies these mechanisms: the threshold for LTP vs. LTD shifts based on the neuron's recent activity, creating an automatic stability mechanism.

## How It Works

### Hebbian Rule

When a tool/model is selected AND the outcome is good, the association between the task type and that selection is strengthened:

```python
from corteX.engine.plasticity import PlasticityManager
from corteX.engine.weights import WeightEngine

weights = WeightEngine()
plasticity = PlasticityManager(weights)

events = plasticity.on_step_complete(
    tool="code_interpreter",
    task_type="coding",
    model="gemini-flash",
    success=True,
    quality=0.85,
)
# Hebbian rule fires:
#   - code_interpreter preference strengthened
#   - gemini-flash<->coding association strengthened
#   - Delta proportional to (quality - 0.5) * 2 = 0.7
```

When the outcome is bad (quality < 0.5), the associations weaken instead.

### Long-Term Potentiation (LTP)

Repeated success with the same tool+task combination triggers exponential strengthening:

```python
# After 3+ consecutive successes with code_interpreter+coding:
# LTP fires with bonus = min(0.2, 0.05 * log(1 + streak - 2))

# Streak 3: bonus = 0.05 * log(2) = 0.035
# Streak 5: bonus = 0.05 * log(4) = 0.069
# Streak 10: bonus = 0.05 * log(9) = 0.110
```

The LTP threshold is 3 consecutive successes. Below this threshold, Hebbian learning operates normally. Above it, the exponential bonus kicks in -- this is how the system "locks in" reliably successful patterns.

### Long-Term Depression (LTD)

Repeated failure triggers exponential weakening:

```python
# After 2+ consecutive failures:
# LTD fires with penalty = min(0.3, 0.1 * log(1 + streak - 1))

# Streak 2: penalty = 0.1 * log(2) = 0.069
# Streak 4: penalty = 0.1 * log(4) = 0.139
# Streak 6: penalty = 0.1 * log(6) = 0.179
```

The LTD threshold is lower than LTP (2 vs. 3) -- the system is quicker to stop doing what does not work than to commit to what does. This asymmetry mirrors the brain's negativity bias and prospect theory's loss aversion.

### Homeostatic Regulation

Homeostasis runs periodically (every 10 interactions by default) and prevents any single weight from dominating:

```python
# Manually trigger
events = plasticity.run_homeostasis()

# What it does:
# 1. Behavioral weights > 0.8 are pulled back toward center
# 2. Model monopolies (one model score > 0.95) are redistributed
# 3. Target mean is 0.5 with regulation strength 0.02
```

```python
from corteX.engine.plasticity import HomeostaticRegulation

regulation = HomeostaticRegulation(
    target_mean=0.5,
    regulation_strength=0.02,
)
```

### Critical Period Modulation

Early in a session, learning rates are doubled. This decays over time as the system becomes more "set in its ways":

```python
from corteX.engine.plasticity import CriticalPeriodModulator

critical = CriticalPeriodModulator(critical_period_turns=10)

# Turn 1:  multiplier = 2.0 (maximum plasticity)
# Turn 5:  multiplier = 1.5
# Turn 10: multiplier = 1.0 (normal learning)
# Turn 20: multiplier = 0.9 (slightly reduced)
# Turn 60: multiplier = 0.5 (minimum)

# Reset at session start
critical.reset()
```

### Surprise Modulation

When the Prediction Engine reports surprise, all plasticity rules fire with amplified effect:

```python
from corteX.engine.prediction import SurpriseSignal

events = plasticity.on_step_complete(
    tool="web_search",
    task_type="research",
    success=False,
    quality=0.2,
    surprise=SurpriseSignal(learning_signal_strength=0.9),
)
# Effective multiplier = critical_period * (1.0 + 0.9) = ~1.5 * 1.9 = ~2.85
# LTD penalty is nearly 3x stronger than normal
```

### Full Pipeline

The `on_step_complete` method coordinates all rules in sequence:

```python
events = plasticity.on_step_complete(
    tool="code_interpreter",
    task_type="coding",
    model="gemini-flash",
    success=True,
    quality=0.85,
    surprise=None,
)
# Returns list of PlasticityEvent objects:
# [
#   PlasticityEvent(rule="hebbian", affected_weights=["tool.code_interpreter"], ...),
#   PlasticityEvent(rule="ltp", affected_weights=["code_interpreter+coding"], ...),
# ]
```

### Session Lifecycle

```python
# Start of session: high plasticity
plasticity.new_session()

# During session: process steps
for step in steps:
    events = plasticity.on_step_complete(...)

# End of session: consolidation
weights.consolidate()
```

## When It Activates

- **After every tool execution**: Hebbian learning, LTP/LTD rules fire
- **Every 10 interactions**: Homeostatic regulation normalizes weights
- **Continuously**: Critical period multiplier decays with each turn
- **When surprise is high**: All learning is amplified
- **At session end**: Consolidation cleans up noise

## API Reference

```python
from corteX.engine.plasticity import (
    PlasticityManager,
    PlasticityEvent,
    HebbianRule,
    LTPRule,
    LTDRule,
    HomeostaticRegulation,
    CriticalPeriodModulator,
)
```
