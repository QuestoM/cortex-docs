# Multi-Resolution Loop Detector

The Multi-Resolution Loop Detector fuses four parallel detection strategies to catch agents stuck in repetitive behavior. It detects exact state repeats, semantic paraphrases, oscillation patterns (A-B-A-B), and dead-end cycling - any single detector above its threshold triggers a loop event with a recommended recovery action. This multi-layered approach catches loops that any single method would miss.

## How It Works

Every agent step is checked through all four detectors simultaneously. Each detector operates independently and can fire a `LoopSignal` with a confidence score. If any signal fires, a `LoopEvent` is created combining all active signals.

### The Four Detectors

**1. ExactHashDetector** - SHA-256 hashes a normalized `"description|output"` string. If the same hash appears 3+ times within a 500-step window, it fires. This catches byte-identical repeated actions.

**2. SemanticJaccardDetector** - Computes token-set Jaccard similarity (after stop-word removal) between the current step and each step in a 30-step window. If 2+ past steps have Jaccard similarity > 0.65, it fires. This catches paraphrased repetitions that evade exact matching.

**3. OscillationDetector** - Watches the sequence of action types for repeating patterns with periods of 2, 3, or 4. If a pattern like `search -> code_edit -> search -> code_edit` repeats 2+ cycles, it fires. This catches flip-flop behavior.

**4. DeadEndDetector** - Tracks error messages. If the same error (first 60 chars, normalized) occurs 3+ times within a 15-step window, it fires. This catches agents stuck retrying the same failing approach.

### Combined Confidence

When multiple detectors fire simultaneously, the combined confidence is:
`mean(all_confidences) + 0.1 * (num_signals - 1)`

This rewards multi-signal agreement - a loop caught by three detectors is rated higher than one caught by a single detector.

### Recommended Actions

| Combined Confidence | Recommendation |
|---------------------|----------------|
| > 0.85 or repeat count > 5 | `escalate` - hand off to user or supervisor |
| Dead-end type | `backtrack` - try a different approach |
| Other | `replan` - generate a new plan from scratch |

## Key Features

- **Four parallel detectors** - exact hash, semantic Jaccard, oscillation pattern, dead-end cycling
- **Four loop types** via `LoopType` enum: `EXACT_REPEAT`, `SEMANTIC_REPEAT`, `OSCILLATION`, `DEAD_END`
- **Tried approaches tracking** - the semantic detector records all seen approaches for the `LoopEvent.tried_approaches` list
- **Per-detector reset** via `reset()` - clear all detector state after successful replanning
- **Bounded history** - loop event history capped at 50 entries
- **Statistics** - `get_stats()` returns step count, total loops detected, and recent loop types

## Integration

The Loop Detector is wired into the corteX turn pipeline:

- **Session post-turn learning** - after each step, the orchestrator calls `check()` with the action type, description, output, and any error
- **Goal Tracker** - loop events can trigger forced replanning
- **Drift Engine** - loop detection and drift detection are complementary - loops indicate behavioral stagnation while drift indicates goal divergence
- **Decision Log** - loop recovery decisions are recorded as `DecisionType.LOOP_RECOVERY`
- **Adaptive Budget** - repeated loops can trigger budget contraction

## Usage Example

```python
from corteX.engine.loop_detector import MultiResolutionLoopDetector

detector = MultiResolutionLoopDetector(
    exact_threshold=3,
    jaccard_threshold=0.65,
    oscillation_min_cycles=2,
    dead_end_threshold=3,
)

# Check each step
event = detector.check(
    action_type="code_edit",
    description="Fix the login validation bug",
    output="Modified auth.py line 42",
    error=None,
)

if event:
    print(f"Loop detected: {event.loop_type.value}")
    print(f"Confidence: {event.combined_confidence:.2f}")
    print(f"Action: {event.recommended_action}")
    print(f"Tried approaches: {event.tried_approaches}")

    if event.recommended_action == "replan":
        # Generate new plan, then reset detectors
        detector.reset()
```

## See Also

- [Drift Engine](drift-engine.md) - complementary drift detection via five-signal fusion
- [Adaptive Budget](adaptive-budget.md) - budget contraction on repeated loops
- [Decision Log](../agent-intelligence/decision-log.md) - loop recovery audit trail
- [Brain-to-Behavior Loop](../behavior-loop.md) - where loop detection happens in the turn cycle
