# Prediction Engine

The Prediction Engine implements a predict-compare-surprise cycle that modulates how fast the system learns. When the agent's predictions are wrong, it learns faster. When predictions are confirmed, learning slows. This is the same mechanism the brain uses to allocate attention and drive synaptic plasticity.

## What It Does

Before each step, the Prediction Engine generates an expected outcome. After the step, it compares the expectation against reality and computes a surprise signal. This surprise signal then modulates the learning rate of the Plasticity Manager -- high surprise means faster weight updates.

```
Predict --> Execute --> Compare --> Surprise Signal --> Modulate Learning
```

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Prediction Error and Dopamine"
    The brain's dopaminergic system implements exactly this loop. Wolfram Schultz's research on reward prediction errors (RPEs) showed that dopamine neurons:

    - **Fire more** when an outcome is better than expected (positive surprise)
    - **Fire less** when an outcome is worse than expected (negative surprise)
    - **Do not fire** when the outcome matches expectation (no surprise)

    This is the computational basis of reinforcement learning. The prediction error signal is what drives synaptic plasticity in the striatum and cortex. The corteX Prediction Engine translates this: surprise -> faster learning, confirmation -> slower learning.

    Prof. Idan Segev describes this as the brain's fundamental operating principle: "The brain does not passively receive input. It actively PREDICTS what the next input will be, and only responds strongly when the prediction is violated."

## How It Works

### Bayesian Surprise Computation

The Weight Engine provides a `compute_surprise_signal` method that uses the `BayesianSurpriseCalculator` to compute KL divergence between predicted and actual outcomes:

```python
from corteX.engine.weights import WeightEngine

engine = WeightEngine()

# After a step completes, compute surprise
signal = engine.compute_surprise_signal(
    prediction_quality=0.9,   # We expected high quality
    actual_quality=0.4,       # We got low quality
)
# signal ≈ 0.85 (high surprise)

signal = engine.compute_surprise_signal(
    prediction_quality=0.7,   # We expected moderate quality
    actual_quality=0.72,      # We got close to prediction
)
# signal ≈ 0.15 (low surprise)
```

The surprise calculation uses:
- **Prior distribution**: Normal(prediction_quality, variance=0.04)
- **Posterior distribution**: Normal(actual_quality, variance=0.01)
- **KL divergence**: measures information gain from prior to posterior
- **Normalization**: maps KL divergence to [0, 1] learning signal

### Surprise-Modulated Plasticity

The `PlasticityManager` integrates the surprise signal as a learning rate multiplier:

```python
from corteX.engine.plasticity import PlasticityManager
from corteX.engine.prediction import SurpriseSignal

plasticity = PlasticityManager(engine)

# High surprise -> amplified learning
events = plasticity.on_step_complete(
    tool="code_interpreter",
    task_type="coding",
    model="gemini-flash",
    success=False,
    quality=0.3,
    surprise=SurpriseSignal(learning_signal_strength=0.8),
)
# LTD penalty is amplified by 1.8x (1.0 + 0.8)
```

The effective learning multiplier combines:
- **Critical period multiplier**: 2.0x early in session, decays to 1.0x
- **Surprise multiplier**: 1.0 + surprise_signal_strength
- **Combined**: `critical_period * surprise_multiplier`

### Proactive Prediction

The `ProactivePredictionEngine` extends basic prediction with Markov chain trajectory modeling:

```python
from corteX.engine.proactive import ProactivePredictionEngine

proactive = ProactivePredictionEngine()

# The engine learns conversation trajectories
# and pre-warms caches for likely next steps
```

The proactive engine:
1. Models conversation as a Markov chain over task states
2. Predicts the most likely next action based on transition probabilities
3. Pre-warms tool caches and context for the predicted action
4. Compares prediction against actual action to update the model

## When It Activates

- **Before each step**: The engine generates a quality prediction based on the current tool, model, and task type
- **After each step**: The engine compares prediction to actual outcome and computes surprise
- **During plasticity**: The surprise signal modulates all LTP/LTD weight updates
- **Proactively**: The trajectory model pre-warms resources for predicted next steps

## API Reference

```python
from corteX.engine.weights import WeightEngine
from corteX.engine.prediction import SurpriseSignal
from corteX.engine.plasticity import PlasticityManager
from corteX.engine.proactive import ProactivePredictionEngine
```
