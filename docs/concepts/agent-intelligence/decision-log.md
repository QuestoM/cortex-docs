# Decision Log

The Decision Log provides an immutable, append-only audit trail for every branch point an agent encounters. Every time the agent selects a model, picks a tool, plans a step, responds to drift, or escalates, the decision is recorded with its alternatives, reasoning, confidence, and eventual outcome. This enables enterprise compliance, post-hoc analysis, and "what-if" reasoning about paths not taken.

## How It Works

Each decision is captured as a `DecisionEntry` dataclass containing:

- **What was decided** - the chosen option and alternatives considered
- **Why** - reasoning text and confidence score (0.0-1.0)
- **Context** - step number, decision type, brain signals, model used
- **Outcome** - rated after the fact as EXCELLENT, GOOD, NEUTRAL, POOR, or FAILURE
- **Cost** - tokens consumed and latency in milliseconds

The log is bounded (default 1,000 entries) with FIFO eviction. Outcomes can be updated retroactively via `update_outcome()`, enabling the agent to learn which decisions worked and which did not.

## Key Features

- **Nine decision types** via `DecisionType` enum: model selection, tool selection, plan step, drift response, escalation, delegation, budget adjustment, loop recovery, pattern selection
- **Post-hoc outcome rating** - five-level `OutcomeRating` from EXCELLENT to FAILURE
- **Regret detection** - `get_regrets()` surfaces decisions with POOR or FAILURE outcomes for analysis
- **Calibration failure detection** - `get_high_confidence_failures()` finds decisions where the agent was confident (>= 0.7) but wrong, indicating confidence calibration issues
- **Pattern analysis** - `analyze_patterns()` computes per-type success rates, average confidence, most common choices, and overall statistics
- **JSON export** - `export(format="json")` serializes the full log for external analysis
- **Brain signal capture** - each entry can carry a `brain_signals` dict with numeric values from any brain component

## Integration

The Decision Log connects with virtually every corteX component:

- **Model Mosaic** - logs pattern selection decisions
- **A/B Test Manager** - logs variant assignment decisions
- **Drift Engine** - logs drift response decisions
- **Loop Detector** - logs loop recovery decisions
- **Adaptive Budget** - logs budget adjustment decisions
- **Goal Tracker** - provides step numbers and goal context for entries

## Usage Example

```python
from corteX.engine.decision_log import (
    DecisionLog, DecisionEntry, DecisionType, OutcomeRating,
)

log = DecisionLog(max_size=500)

# Record a model selection decision
idx = log.record(DecisionEntry(
    step_number=3,
    decision_type=DecisionType.MODEL_SELECTION,
    description="Select model for code generation task",
    chosen="gemini-2.5-pro",
    alternatives=["gemini-2.5-flash", "local-llama"],
    reasoning="High complexity task requires strong reasoning",
    confidence=0.85,
    brain_signals={"complexity": 0.9, "urgency": 0.3},
))

# Later, record the outcome
log.update_outcome(idx, OutcomeRating.GOOD, "Code was correct first try")

# Analyze decision patterns
patterns = log.analyze_patterns()
print(f"Overall success rate: {patterns['overall_success_rate']:.0%}")
print(f"Regrets: {patterns['regret_count']}")
print(f"Calibration failures: {patterns['calibration_failure_count']}")

# Find decisions that went wrong
for regret in log.get_regrets():
    print(f"Step {regret.step_number}: chose '{regret.chosen}', "
          f"alternatives were {regret.alternatives}")

# Export for compliance
json_export = log.export(format="json")
```

## See Also

- [A/B Test Manager](ab-test-manager.md) - systematic comparison of decision strategies
- [Drift Engine](../anti-drift/drift-engine.md) - drift response decisions
- [Loop Detector](../anti-drift/loop-detector.md) - loop recovery decisions
- [Brain-to-Behavior Loop](../behavior-loop.md) - where decisions happen in the agent cycle
