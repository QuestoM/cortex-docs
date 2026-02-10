# Component Simulator

The Component Simulator is a "digital twin" engine that enables what-if analysis, A/B testing, and scenario exploration without affecting the live agent state. It creates lightweight copies of system components and runs hypothetical scenarios against them.

## What It Does

The simulator provides three capabilities:

1. **What-If Analysis**: "What would happen if we changed this weight?"
2. **A/B Testing**: Compare two configurations side-by-side
3. **Scenario Running**: Replay a sequence of events against modified state

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Mental Simulation"
    The brain has a remarkable ability to simulate future scenarios without acting on them. The prefrontal cortex can "mentally rehearse" a plan, evaluating likely outcomes before committing resources to execution.

    This is distinct from actual execution: the motor cortex generates a plan, but inhibitory circuits prevent it from reaching the muscles. The brain can then evaluate the simulated outcome and decide whether to proceed.

    fMRI studies show that mental simulation activates many of the same brain regions as actual execution, but with reduced intensity. The Component Simulator implements this: it uses the same code as the live system, but operates on a copy of state.

## How It Works

### What-If Analysis

```python
from corteX.engine.simulator import WhatIfAnalyzer, SimulationState

analyzer = WhatIfAnalyzer()

# Create a snapshot of current state
state = SimulationState.from_weight_engine(engine)

# Ask: "What if we increased risk_tolerance to 0.8?"
result = analyzer.what_if(
    base_state=state,
    modifications={"behavioral.risk_tolerance": 0.8},
    scenario=[
        {"tool": "experimental_tool", "success": True, "quality": 0.7},
        {"tool": "experimental_tool", "success": False, "quality": 0.2},
    ],
)
# result.final_state shows how weights would evolve
# result.quality_trajectory shows quality over the scenario
```

### A/B Testing

```python
from corteX.engine.simulator import ABTestManager

ab_manager = ABTestManager()

# Compare two configurations
result = ab_manager.test(
    config_a={"model": "gemini-flash", "temperature": 0.3},
    config_b={"model": "gemini-pro", "temperature": 0.7},
    scenario=scenario_steps,
)
# result.winner: "config_b"
# result.metrics_a: {quality: 0.72, speed: 0.9, cost: 0.3}
# result.metrics_b: {quality: 0.85, speed: 0.6, cost: 0.7}
```

### Scenario Running

```python
from corteX.engine.simulator import ScenarioRunner

runner = ScenarioRunner()

# Replay a saved trajectory against modified weights
result = runner.run(
    initial_state=modified_state,
    steps=saved_trajectory,
)
# result.outcomes: list of outcomes at each step
# result.divergence_point: where the modified path diverges from original
```

### Simulated Weight Engine

The simulator includes a `SimulatedWeightEngine` that mirrors the real `WeightEngine` but operates on a copy:

```python
from corteX.engine.simulator import SimulatedWeightEngine

sim_engine = SimulatedWeightEngine.from_snapshot(engine.snapshot())

# Modify and test without affecting the real engine
sim_engine.behavioral.update("risk_tolerance", 0.5)
sim_engine.tools.record_use("experimental", success=True, latency_ms=100)

# Compare with real engine
diff = sim_engine.diff(engine)
# Shows which weights diverged and by how much
```

## When It Activates

- **On demand**: Developers use the simulator for what-if analysis during development
- **During A/B testing**: Compare configuration options before deployment
- **For enterprise evaluation**: Test policy changes against recorded trajectories before applying them
- **During calibration**: Assess how weight modifications would affect calibration error

## API Reference

```python
from corteX.engine.simulator import (
    ComponentSimulator,
    SimulationState,
    StateDelta,
    SimulatedWeightEngine,
    ScenarioRunner,
    ABTestManager,
    WhatIfAnalyzer,
)
```
