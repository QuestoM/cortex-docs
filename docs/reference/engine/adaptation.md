# Adaptation Filter

`corteX.engine.adaptation`

Sensory adaptation engine inspired by the nervous system's fundamental sensitivity to changes. "A brake light that stays on forever, you'll ignore it. Changes are what matter." Implements rapid adaptation (exponential decay for repeated signals) and sustained habituation (complete filtering after prolonged repetition).

The key insight: do not react to steady states, react to CHANGES. A user who always sends short messages is not signaling "brevity preference" -- that is just their style. A user who SWITCHES from long to short messages is a strong signal.

---

## AdaptationState

Tracks the adaptation state for a single signal type.

| Attribute | Type | Description |
|-----------|------|-------------|
| `signal_key` | `str` | Identifier for this signal type |
| `last_value` | `Optional[float]` | Last observed value |
| `repetition_count` | `int` | Consecutive repetitions of the same value |
| `last_change_time` | `float` | Timestamp of the last value change |
| `baseline` | `Optional[float]` | Learned baseline value (EMA) |
| `baseline_samples` | `int` | Number of samples in the baseline |
| `habituated` | `bool` | Whether this signal is fully habituated |

---

## ChangeSignal

A detected change in user behavior.

| Attribute | Type | Description |
|-----------|------|-------------|
| `signal_key` | `str` | Which signal changed |
| `change_magnitude` | `float` | Size of the change (`0.0` to `1.0`) |
| `change_direction` | `float` | `-1.0` (decreased) to `1.0` (increased) |
| `previous_value` | `float` | Value before the change |
| `current_value` | `float` | Current value |
| `adaptation_weight` | `float` | How much attention to pay (decreases with repetition) |
| `is_novel` | `bool` | First time seeing this signal type |
| `timestamp` | `float` | Unix timestamp (auto-populated) |

---

## RapidAdaptation

Rapidly adapting filter, like Meissner's corpuscles: fires strongly on first contact, response decays exponentially with repeated identical stimuli.

### Constructor

```python
RapidAdaptation(decay_rate: float = 0.5, novelty_bonus: float = 2.0)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `decay_rate` | `float` | `0.5` | How fast the response decays (`0` to `1`) |
| `novelty_bonus` | `float` | `2.0` | Multiplier for first occurrence or value change |

### Methods

#### `filter(signal_key: str, value: float) -> ChangeSignal`

Processes a signal through rapid adaptation. Returns a `ChangeSignal` with `adaptation_weight` reflecting how much attention to pay.

**Behavior:**

- Novel signal: `adaptation_weight = novelty_bonus` (2.0x)
- Repeated same value (delta < 0.05): `weight = decay_rate ^ (repetitions - 1)`
- Value changed: resets adaptation, returns `novelty_bonus`

#### `reset(signal_key: Optional[str] = None) -> None`

Resets adaptation state for a specific signal or all signals.

---

## SustainedAdaptation

Slowly adapting filter with eventual complete habituation, like Merkel's discs. Sustains response longer but eventually stops responding entirely after prolonged constant stimulus.

### Constructor

```python
SustainedAdaptation(habituation_threshold: int = 8, recovery_time: float = 300.0)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `habituation_threshold` | `int` | `8` | Repetitions before complete habituation |
| `recovery_time` | `float` | `300.0` | Seconds to recover from habituation |

### Methods

#### `filter(signal_key: str, value: float) -> Optional[ChangeSignal]`

Processes a signal through sustained adaptation. Returns `None` if the signal is habituated (should be ignored).

**Behavior:**

- Novel signal: full weight (`1.0`)
- Repeated: weight decays linearly to `0.2` before habituation
- After threshold: complete habituation (`None` returned)
- Value change: resets habituation, returns weight `1.5` (bonus for change after steady state)
- After recovery time: dishabituation occurs automatically

#### `is_habituated(signal_key: str) -> bool`

Returns whether a signal is currently habituated.

#### `get_baseline(signal_key: str) -> Optional[float]`

Returns the learned baseline value for a signal (EMA with alpha `0.1`).

#### `reset(signal_key: Optional[str] = None) -> None`

Resets adaptation state.

---

## AdaptationFilter

Main adaptation filter combining rapid and sustained adaptation. Wraps the feedback engine to filter out repetitive signals and amplify novel/changed signals.

### Constructor

```python
AdaptationFilter(
    rapid_decay: float = 0.5,
    habituation_threshold: int = 8,
    recovery_time: float = 300.0,
)
```

### Methods

#### `process(signal_key: str, value: float) -> Optional[ChangeSignal]`

Processes a signal through both adaptation filters. Returns a `ChangeSignal` with the minimum (most conservative) combined weight, or `None` if completely habituated.

#### `detect_behavioral_shift(signal_key: str, current_value: float, window: int = 5) -> Optional[ChangeSignal]`

Detects significant behavioral shifts by comparing the current value to the learned baseline. Returns a `ChangeSignal` only if deviation exceeds `0.15`. Larger shifts produce higher `adaptation_weight` values (`1.5 + |deviation|`).

#### `get_habituated_signals() -> List[str]`

Returns list of currently habituated signal types.

#### `get_active_signals() -> List[str]`

Returns list of signals that are NOT habituated.

#### `get_stats() -> Dict[str, Any]`

Returns: `total_signals_processed`, `habituated_signals`, `active_signals`, `baselines`.

#### `new_session() -> None`

Resets rapid adaptation and habituation counts for a new session. Preserves baselines for cross-session learning.

### Example

```python
from corteX.engine.adaptation import AdaptationFilter

adaptation = AdaptationFilter()

# First frustration signal: strong response
result = adaptation.process("frustration", 0.8)
# result.adaptation_weight ≈ 2.0 (novel)

# Same signal repeated: decaying response
result = adaptation.process("frustration", 0.8)
# result.adaptation_weight ≈ 1.0

# After 8 repetitions: habituated (None)
for _ in range(6):
    adaptation.process("frustration", 0.8)
result = adaptation.process("frustration", 0.8)
# result is None -- signal ignored

# Sudden change: strong response again
result = adaptation.process("frustration", 0.2)
# result.adaptation_weight ≈ 1.5 (change bonus)

# Detect behavioral shift from baseline
shift = adaptation.detect_behavioral_shift("frustration", 0.2)
if shift:
    print(f"Behavioral shift: {shift.change_magnitude:.2f}")
```
