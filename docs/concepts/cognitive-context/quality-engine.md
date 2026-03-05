# Context Quality Engine

The `ContextQualityEngine` measures context quality across six dimensions, producing a comprehensive health score that drives adaptive context management. Rather than treating context as a black box, it quantifies exactly where quality is degrading and recommends specific corrective actions.

## How It Works

The engine evaluates assembled context messages against six scored dimensions, then combines them using a weighted harmonic mean to produce an overall health score. The harmonic mean ensures that a single critically low dimension drags down the overall score - you cannot mask a gap by excelling elsewhere.

### The Six Dimensions

| Dimension | Code | Weight | What It Measures |
|-----------|------|--------|------------------|
| Goal Retention Score | `grs` | 0.25 | How many goal keywords appear in the context |
| Information Density Index | `idi` | 0.10 | Unique entities per token (via regex extraction) |
| Entanglement Completeness | `ec` | 0.20 | Fraction of entangled pairs where both items are present |
| Temporal Coherence | `tc` | 0.15 | Whether causal chain steps appear in context |
| Decision Preservation Rate | `dpr` | 0.20 | Fraction of recent decisions still present in context |
| Anti-Hallucination Score | `ahs` | 0.10 | Ratio of tool/system-sourced content vs total tokens |

Each dimension is scored 0.0 to 1.0. Configurable per-dimension thresholds (`QualityThresholds`) trigger warnings and recommendations when scores drop below acceptable levels.

### Health Labels

The overall score maps to four labels:

- **optimal** - Score >= 0.7 (healthy threshold)
- **healthy** - Score >= 0.5 (degrading threshold)
- **degrading** - Score >= 0.3 (critical threshold)
- **critical** - Score < 0.3

### Degradation Velocity

The engine tracks how fast quality is dropping by comparing the current overall score to the previous evaluation. A velocity above 0.05 per step triggers an explicit warning, enabling preemptive intervention before quality becomes critical.

## Key Features

- **6-dimensional scoring** with configurable weights and thresholds
- **Weighted harmonic mean** ensures no single weak dimension is hidden
- **Trend tracking** via `get_trend()` returns per-dimension score history over a configurable window
- **Weakest dimension detection** via `get_weakest_dimension()` identifies the most urgent area to fix
- **Dimension correlation analysis** using Pearson correlation to find dimensions that move together
- **Actionable recommendations** generated per dimension when thresholds are breached
- **Lightweight entity extraction** using regex patterns for file paths, API endpoints, Python identifiers, and quoted strings

## Integration

The `ContextQualityEngine` is called by the `CognitiveContextPipeline` as one of its 8 compilation phases. The resulting `ContextQualityReport` feeds into the Orchestrator's decision loop - if quality drops below thresholds, the engine can trigger context recompilation, density optimization, or entanglement pair restoration.

The quality scores are also recorded by the `ContextVersioner` at each decision point, enabling causal diff analysis when failures occur.

## Usage Example

```python
from corteX.engine.cognitive import (
    ContextQualityEngine, QualityThresholds
)

# Custom thresholds for stricter quality
engine = ContextQualityEngine(
    thresholds=QualityThresholds(
        grs=0.4,    # Require 40% goal word retention
        dpr=0.9,    # Require 90% decision preservation
    ),
)

# Evaluate context quality
report = engine.evaluate(
    context_messages=[
        {"role": "system", "content": "You are a deployment agent."},
        {"role": "tool", "content": "Tests passed: 142/142"},
    ],
    goal="Deploy user service to production",
    total_tokens=500,
    decision_log=[{"decision": "Use blue-green deploy"}],
    entangled_pairs_total=5,
    entangled_pairs_complete=4,
)

print(report.health_label)          # "optimal", "healthy", etc.
print(report.overall_health)        # 0.0-1.0
print(report.warnings)             # List of warning strings
print(report.recommendations)      # List of corrective actions

# Track trends over time
trend = engine.get_trend(window=10)
weakest = engine.get_weakest_dimension()  # ("grs", 0.35)
correlated = engine.get_correlated_dimensions(threshold=0.7)
```

## See Also

- [Entanglement Graph](entanglement.md) - Provides the entanglement completeness input
- [Density Optimizer](density-optimizer.md) - Improves the information density index
- [State File Manager](state-file-manager.md) - Provides decision log for preservation scoring
- [Context Versioner](versioner.md) - Records quality scores at each decision point
