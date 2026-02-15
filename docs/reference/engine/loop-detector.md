# Loop Detector API Reference

## Module: `corteX.engine.loop_detector`

Multi-resolution loop detection with four parallel strategies. Catches exact repeats, semantic paraphrases, oscillation patterns, and dead-end cycling. Any detector above its threshold triggers a loop event.

Brain analogy: Hippocampus (deja vu detection), entorhinal cortex (state mapping), basal ganglia (action-selection loops).

---

## Classes

### `LoopType`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `EXACT_REPEAT` | Identical state hash seen multiple times |
| `SEMANTIC_REPEAT` | Semantically similar actions (Jaccard similarity) |
| `OSCILLATION` | A->B->A->B flip-flop pattern |
| `DEAD_END` | Same error repeated multiple times |

### `LoopSignal`

**Type**: `@dataclass`

Signal emitted by an individual detector.

| Attribute | Type | Description |
|-----------|------|-------------|
| `detector_name` | `str` | Which detector fired |
| `confidence` | `float` | Detection confidence [0.0, 1.0] |
| `evidence` | `str` | Human-readable evidence |
| `loop_type` | `LoopType` | Type of loop detected |
| `repeat_count` | `int` | Number of repetitions |
| `timestamp` | `float` | Detection timestamp |

### `LoopEvent`

**Type**: `@dataclass`

Aggregated loop detection result when a loop is confirmed.

| Attribute | Type | Description |
|-----------|------|-------------|
| `loop_type` | `LoopType` | Primary loop type (highest confidence) |
| `signals` | `List[LoopSignal]` | All detector signals |
| `combined_confidence` | `float` | Fused confidence score |
| `tried_approaches` | `List[str]` | Approaches already tried |
| `recommended_action` | `str` | `"replan"`, `"backtrack"`, or `"escalate"` |
| `step_number` | `int` | Step where loop was detected |

---

## Individual Detectors

### `ExactHashDetector`

Detects exact state repetitions via SHA-256 hashing. Normalizes `description|output` text, computes 16-char hash, tracks counts in a sliding window.

- **Threshold**: 3 (same hash seen 3+ times)
- **Window**: 500 entries
- **Confidence**: `0.5 + 0.15 * (count - threshold)`

### `SemanticJaccardDetector`

Detects semantically similar actions via token-set Jaccard similarity. Tokenizes text (lowercase, remove stop words, length > 2), compares against sliding window.

- **Threshold**: 0.65 Jaccard similarity
- **Window**: 30 entries
- **Triggers**: 2+ similar past actions
- **Confidence**: `0.4 + 0.2 * similar_count`

### `OscillationDetector`

Detects A->B->A->B flip-flop patterns in action type sequences. Checks periods of 2, 3, and 4.

- **Minimum cycles**: 2 (e.g., A->B->A->B)
- **Window**: 20 entries
- **Confidence**: `0.5 + 0.15 * (cycles - min_cycles)`

### `DeadEndDetector`

Detects repeated errors of the same type -- the agent is stuck hitting the same wall.

- **Threshold**: 3 (same error 3+ times)
- **Window**: 15 entries
- **Confidence**: `0.6 + 0.1 * (count - threshold)`

---

### `MultiResolutionLoopDetector`

Fuses four parallel loop detectors into a unified detection system.

#### Constructor

```python
MultiResolutionLoopDetector(
    exact_threshold: int = 3,
    jaccard_threshold: float = 0.65,
    oscillation_min_cycles: int = 2,
    dead_end_threshold: int = 3,
)
```

#### Methods

##### `check`

```python
def check(
    self, action_type: str, description: str, output: str,
    error: Optional[str] = None,
) -> Optional[LoopEvent]
```

Run all four detectors in parallel. Returns `LoopEvent` if any detector triggers, `None` otherwise.

**Combined confidence**: `average(signals) + 0.1 * (num_detectors - 1)` (multi-detector agreement bonus).

**Recommended action**:

| Condition | Action |
|-----------|--------|
| confidence > 0.85 or repeat > 5 | `"escalate"` |
| Dead end loop | `"backtrack"` |
| Other | `"replan"` |

##### `reset`

```python
def reset(self) -> None
```

Reset all detectors (e.g., after a successful replan).

##### `get_stats`

Returns: step_number, total_loops_detected, loop_history_size, recent_loop_types.

---

## Usage Example

```python
from corteX.engine.loop_detector import MultiResolutionLoopDetector

detector = MultiResolutionLoopDetector()

# Check each agent step
event = detector.check(
    action_type="code_write",
    description="Writing user model",
    output="class User(Model): ...",
    error=None,
)

if event:
    print(f"Loop detected: {event.loop_type.value}")
    print(f"Confidence: {event.combined_confidence:.2f}")
    print(f"Action: {event.recommended_action}")
    print(f"Tried: {event.tried_approaches}")

    if event.recommended_action == "replan":
        detector.reset()  # Reset after replan
```

---

## See Also

- [Drift Engine API](./drift-engine.md)
- [Adaptive Budget API](./adaptive-budget.md)
- [Recovery Engine API](./recovery.md)
