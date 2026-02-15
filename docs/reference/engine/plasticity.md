# Plasticity Manager

`corteX.engine.plasticity`

Implements brain-inspired learning mechanisms that modify the weight system. Coordinates Hebbian learning ("fire together, wire together"), Long-Term Potentiation (repeated success strengthens connections), Long-Term Depression (repeated failure weakens connections), homeostatic regulation (prevents runaway weights), and critical period sensitivity (early interactions shape behavior more).

---

## PlasticityEvent

Record of a plasticity rule activation.

| Attribute | Type | Description |
|-----------|------|-------------|
| `rule` | `str` | Which rule fired (`"hebbian"`, `"ltp"`, `"ltd"`, `"homeostatic"`) |
| `affected_weights` | `List[str]` | Weight keys that were modified |
| `magnitude` | `float` | Strength of the change applied |
| `timestamp` | `float` | Unix timestamp (auto-populated) |
| `details` | `str` | Human-readable description of the event |

---

## HebbianRule

"Neurons that fire together, wire together." When a tool/model is selected AND the outcome is good, strengthen the association. When they fire together but outcome is bad, weaken it.

### Methods

#### `apply(weight_engine: WeightEngine, tool_name: str, task_type: str, model_name: str, outcome_quality: float) -> List[PlasticityEvent]`

Applies Hebbian learning based on co-activation of tool/model and task.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `weight_engine` | `WeightEngine` | The weight engine to update |
| `tool_name` | `str` | Tool that was used |
| `task_type` | `str` | Type of task performed |
| `model_name` | `str` | LLM model that was used |
| `outcome_quality` | `float` | Outcome quality (`0.0` to `1.0`); `0.5` is neutral |

**Returns:** List of `PlasticityEvent` records.

The Hebbian delta maps `[0, 1]` quality to `[-1, 1]`: values above `0.5` strengthen, below weaken.

---

## LTPRule

Long-Term Potentiation: repeated successful patterns get exponentially stronger. After N consecutive successes with the same approach, the association is "locked in."

### Constructor

The LTP threshold defaults to 3 consecutive successes before triggering.

### Methods

#### `apply(weight_engine: WeightEngine, pattern_key: str, success: bool) -> Optional[PlasticityEvent]`

Applies LTP based on success streaks. The `pattern_key` format is `"tool_name+task_type"`.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `weight_engine` | `WeightEngine` | The weight engine to update |
| `pattern_key` | `str` | Pattern identifier (e.g., `"code_interpreter+coding"`) |
| `success` | `bool` | Whether this attempt succeeded |

**Returns:** `PlasticityEvent` if LTP triggered, `None` otherwise. A failure resets the streak to zero.

**Bonus formula:** `min(0.2, 0.05 * log1p(streak - threshold + 1))`

---

## LTDRule

Long-Term Depression: repeated failures exponentially weaken the association. After N consecutive failures, the system actively seeks alternatives.

### Constructor

The LTD threshold defaults to 2 consecutive failures before triggering.

### Methods

#### `apply(weight_engine: WeightEngine, pattern_key: str, success: bool) -> Optional[PlasticityEvent]`

Applies LTD based on failure streaks. A success resets the streak.

**Penalty formula:** `min(0.3, 0.1 * log1p(streak - threshold + 1))`

---

## HomeostaticRegulation

Prevents any single weight from dominating. Maintains overall system stability. Like the brain's mechanisms that prevent seizures (runaway excitation) and coma (runaway inhibition).

### Constructor

```python
HomeostaticRegulation(target_mean: float = 0.5, regulation_strength: float = 0.02)
```

### Methods

#### `apply(weight_engine: WeightEngine) -> List[PlasticityEvent]`

Normalizes weights toward the target distribution. Pulls extreme behavioral weights (> `0.8`) back toward center. Prevents model selection monopolies where one model score exceeds `0.95`.

---

## CriticalPeriodModulator

Early in a session, the system is more plastic -- learning rates are higher. As the session matures, learning rates decrease. Like language acquisition critical periods in child development.

### Constructor

```python
CriticalPeriodModulator(critical_period_turns: int = 10)
```

### Methods

#### `get_plasticity_multiplier() -> float`

Returns a multiplier for learning rates. During the critical period (first 10 turns by default), returns `2.0` decaying to `1.0`. After the critical period, slowly decays to a floor of `0.5`.

#### `reset() -> None`

Resets the turn counter for a new session.

---

## PlasticityManager

Coordinates all plasticity rules. The main class that the orchestrator calls after each step.

### Constructor

```python
PlasticityManager(weight_engine: WeightEngine)
```

| Attribute | Type | Description |
|-----------|------|-------------|
| `weights` | `WeightEngine` | The weight engine to modify |
| `hebbian` | `HebbianRule` | Hebbian co-activation learning |
| `ltp` | `LTPRule` | Long-Term Potentiation |
| `ltd` | `LTDRule` | Long-Term Depression |
| `homeostasis` | `HomeostaticRegulation` | Stability regulation |
| `critical_period` | `CriticalPeriodModulator` | Session-phase sensitivity |

### Methods

#### `on_step_complete(tool: str = "", task_type: str = "", model: str = "", success: bool = True, quality: float = 0.5, surprise: Optional[SurpriseSignal] = None) -> List[PlasticityEvent]`

Applies all plasticity rules after a step completes. The effective learning multiplier combines critical period phase and surprise magnitude.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `tool` | `str` | Tool that was used |
| `task_type` | `str` | Type of task performed |
| `model` | `str` | LLM model used |
| `success` | `bool` | Whether the step succeeded |
| `quality` | `float` | Output quality (`0.0` to `1.0`) |
| `surprise` | `Optional[SurpriseSignal]` | Surprise signal from PredictionEngine |

**Returns:** List of all plasticity events that fired.

**Processing order:**

1. Hebbian learning (co-activation)
2. LTP (success streaks)
3. LTD (failure streaks)
4. Homeostatic regulation (every 10 steps)

#### `run_homeostasis() -> List[PlasticityEvent]`

Manually triggers homeostatic regulation outside the normal step cycle.

#### `new_session() -> None`

Resets the critical period modulator for a new session.

#### `get_stats() -> Dict[str, Any]`

Returns: `total_steps`, `plasticity_multiplier`, `total_events`, `ltp_streaks`, `ltd_streaks`.

### Example

```python
from corteX.engine.weights import WeightEngine
from corteX.engine.plasticity import PlasticityManager
from corteX.engine.prediction import SurpriseSignal

weights = WeightEngine()
plasticity = PlasticityManager(weights)

# After each agent step
events = plasticity.on_step_complete(
    tool="code_interpreter",
    task_type="coding",
    model="gemini-flash",
    success=True,
    quality=0.85,
    surprise=surprise_signal,  # from PredictionEngine
)

for event in events:
    print(f"Rule: {event.rule}, Magnitude: {event.magnitude:.3f}")

# At session end
weights.consolidate()

# New session
plasticity.new_session()
```
