# Component Simulator API Reference

## Module: `corteX.engine.simulator`

Digital twin entry point for cortical simulation. Fork live state, run scenarios, A/B test configurations, Monte Carlo analysis, and what-if experiments -- all in a sandboxed environment. Includes session recording and replay for live pipeline integration.

---

## Classes

### `SessionRecording`

**Type**: `@dataclass`

A recorded sequence of live agent interactions for replay.

| Attribute | Type | Description |
|-----------|------|-------------|
| `session_id` | `str` | Recording identifier |
| `turns` | `List[SimulatedTurn]` | Recorded interaction turns |
| `initial_state_dict` | `Dict[str, Any]` | State at recording start |
| `recorded_at` | `float` | Recording timestamp |
| `metadata` | `Dict[str, Any]` | Additional metadata |

---

### `ComponentSimulator`

Main entry point for cortical simulation with lazy initialization and session replay.

#### Constructor

```python
ComponentSimulator()
```

No parameters. Sub-components (`ScenarioRunner`, `ABTestManager`, `WhatIfAnalyzer`, `SimulationDashboard`) are lazily initialized on first use.

#### Methods

##### `fork`

```python
def fork(self, live_state: Dict[str, Any]) -> SimulationState
```

Create a `SimulationState` from a live WeightEngine snapshot. Deep-copies all state to prevent mutation. Extracts keys: `behavioral`, `tool_preference`, `model_selection`, `goal_alignment`, `user_insights`, `enterprise`, `global`.

##### `run`

```python
def run(
    self, sim_state: SimulationState, scenario: List[SimulatedTurn],
    record_trajectory: bool = True,
) -> SimulationResult
```

Run a simulated scenario from a given state. Returns `SimulationResult` with metrics, trajectory, and state delta.

##### `what_if`

```python
def what_if(
    self, sim_state: SimulationState, change: Dict[str, Any],
    scenario: Optional[List[SimulatedTurn]] = None,
) -> SimulationResult
```

Dispatch a what-if analysis based on `change["type"]`:

| Type | Required Keys | Description |
|------|--------------|-------------|
| `"change_param"` | `param_path`, `new_value` | Change a parameter value |
| `"remove_tool"` | `tool_name` | Remove a tool and observe impact |
| `"add_tool"` | `tool_name`, `initial_quality` | Add a new tool |

##### `ab_test`

```python
def ab_test(
    self, sim_state: SimulationState, config_a: Dict[str, Any],
    config_b: Dict[str, Any], scenario: List[SimulatedTurn],
    name: Optional[str] = None, n_monte_carlo: int = 30,
) -> ABTestResult
```

Run an A/B test comparing two configurations. Uses Monte Carlo sampling for statistical significance.

##### `monte_carlo`

```python
def monte_carlo(
    self, sim_state: SimulationState, scenario: List[SimulatedTurn],
    n_runs: int = 100, noise_std: float = 0.1,
    seed: Optional[int] = None,
) -> MonteCarloResult
```

Run N Monte Carlo simulations with stochastic outcome noise. Returns statistical aggregates (means, std devs, confidence intervals).

##### `summarize` / `compare_results` / `analyze_trajectory`

```python
def summarize(self, result: SimulationResult) -> Dict[str, Any]
def compare_results(self, results: List[SimulationResult]) -> Dict[str, Any]
def analyze_trajectory(self, result: SimulationResult) -> Dict[str, Any]
```

Dashboard methods for human-readable summaries, cross-comparison, and trajectory analysis.

#### Session Replay Methods

##### `start_recording`

```python
def start_recording(self, session_id: str, initial_state: Dict[str, Any]) -> None
```

Start recording a live session for later replay. Deep-copies initial state.

##### `record_turn`

```python
def record_turn(self, turn_data: Dict[str, Any]) -> None
```

Record a live turn. Expected keys: `message_type`, `tools_used`, `success`, `quality`, `task_type`, `model_used`, `latency_ms`, `metadata`. No-op if not recording.

##### `stop_recording`

```python
def stop_recording(self) -> Optional[SessionRecording]
```

Stop recording and return the completed recording. Stores in internal registry.

##### `replay`

```python
def replay(
    self, recording: SessionRecording,
    overrides: Optional[Dict[str, Any]] = None,
) -> SimulationResult
```

Replay a recorded session, optionally with state overrides.

##### `replay_with_ab`

```python
def replay_with_ab(
    self, recording: SessionRecording, config_a: Dict[str, Any],
    config_b: Dict[str, Any], n_monte_carlo: int = 30,
) -> ABTestResult
```

Replay a recorded session as an A/B test with two configurations.

##### `get_recordings` / `get_stats`

Get stored recordings and simulator usage statistics.

---

## Usage Example

```python
from corteX.engine.simulator import ComponentSimulator, SimulatedTurn, MessageType

sim = ComponentSimulator()

# Fork live state
state = sim.fork(weight_engine.get_all_weights())

# Define a scenario
scenario = [
    SimulatedTurn(message_type=MessageType.USER_QUERY, tools_used=["code_writer"],
                  success=True, quality=0.8, task_type="coding"),
    SimulatedTurn(message_type=MessageType.TOOL_CALL, tools_used=["test_runner"],
                  success=True, quality=0.9, task_type="testing"),
]

# Run simulation
result = sim.run(state, scenario)
print(sim.summarize(result))

# What-if: what if we removed code_writer?
what_if_result = sim.what_if(state, {"type": "remove_tool", "tool_name": "code_writer"})

# A/B test two configurations
ab_result = sim.ab_test(
    state,
    config_a={"behavioral.creativity": 0.3},
    config_b={"behavioral.creativity": 0.8},
    scenario=scenario,
)
print(f"Winner: {ab_result.winner}")

# Monte Carlo for confidence intervals
mc = sim.monte_carlo(state, scenario, n_runs=100)
```

---

## See Also

- [Weight Engine API](./weight-engine.md)
- [A/B Test Manager API](./ab-test-manager.md)
