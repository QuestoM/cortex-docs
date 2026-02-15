# Context Quality Engine API Reference

## Module: `corteX.engine.cognitive.context_quality`

6-dimensional cognitive context quality scoring. Dimensions: Goal Retention (GRS), Information Density (IDI), Entanglement Completeness (EC), Temporal Coherence (TC), Decision Preservation (DPR), Anti-Hallucination (AHS). Overall health is a weighted harmonic mean. Tracks trends, degradation velocity, dimension correlations, and generates recommendations.

## Classes

### `ContextQualityReport`

**Type**: `@dataclass`

Comprehensive quality assessment of assembled context.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `goal_retention_score` | `float` | GRS: How much of the goal is retained in context (0.0-1.0) |
| `information_density_index` | `float` | IDI: Unique entities per token (0.0-1.0) |
| `entanglement_completeness` | `float` | EC: Fraction of entangled pairs that are complete (0.0-1.0) |
| `temporal_coherence` | `float` | TC: Causal chain steps present in context (0.0-1.0) |
| `decision_preservation_rate` | `float` | DPR: Recent decisions retained in context (0.0-1.0) |
| `anti_hallucination_score` | `float` | AHS: Ratio of grounded (tool/system) content (0.0-1.0) |
| `overall_health` | `float` | Weighted harmonic mean of all dimensions |
| `health_label` | `str` | One of `"optimal"`, `"healthy"`, `"degrading"`, `"critical"` |
| `warnings` | `List[str]` | Active quality warnings |
| `recommendations` | `List[str]` | Actionable improvement recommendations |
| `dimension_scores` | `Dict[str, float]` | All dimension scores keyed by abbreviation |
| `degradation_velocity` | `float` | Rate of quality decline per step |
| `timestamp` | `float` | Unix timestamp of evaluation |

---

### `QualityThresholds`

**Type**: `@dataclass`

Per-dimension thresholds with alert severity.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `grs` | `float` | `0.3` | Goal Retention critical threshold |
| `idi` | `float` | `0.4` | Information Density critical threshold |
| `ec` | `float` | `0.9` | Entanglement Completeness critical threshold |
| `tc` | `float` | `0.7` | Temporal Coherence critical threshold |
| `dpr` | `float` | `0.8` | Decision Preservation critical threshold |
| `ahs` | `float` | `0.5` | Anti-Hallucination critical threshold |
| `critical_overall` | `float` | `0.3` | Overall score below this is "critical" |
| `degrading_overall` | `float` | `0.5` | Overall score below this is "degrading" |
| `healthy_overall` | `float` | `0.7` | Overall score at or above this is "optimal" |

---

### `ContextQualityEngine`

Monitors and scores context quality across 6 dimensions.

#### Class Constants

```python
WEIGHTS = {
    "grs": 0.25,  # Goal Retention (most important)
    "idi": 0.10,  # Information Density
    "ec":  0.20,  # Entanglement Completeness
    "tc":  0.15,  # Temporal Coherence
    "dpr": 0.20,  # Decision Preservation
    "ahs": 0.10,  # Anti-Hallucination
}
```

#### Constructor

```python
ContextQualityEngine(
    thresholds: Optional[QualityThresholds] = None,
    weights: Optional[Dict[str, float]] = None
)
```

**Parameters**:

- `thresholds` (`Optional[QualityThresholds]`): Custom quality thresholds. Defaults to standard thresholds.
- `weights` (`Optional[Dict[str, float]]`): Custom dimension weights for the harmonic mean.

#### Methods

##### `evaluate`

```python
def evaluate(
    self, context_messages: List[Dict[str, str]], goal: str,
    total_tokens: int,
    decision_log: Optional[List[Dict[str, str]]] = None,
    entangled_pairs_total: int = 0,
    entangled_pairs_complete: int = 0,
    causal_chain: Optional[List[str]] = None
) -> ContextQualityReport
```

Evaluate context quality across all 6 dimensions.

**Parameters**:

- `context_messages` (`List[Dict[str, str]]`): Assembled context messages.
- `goal` (`str`): Current goal text.
- `total_tokens` (`int`): Total token count of context.
- `decision_log` (`Optional[List[Dict[str, str]]]`): Recent decisions with `decision` key.
- `entangled_pairs_total` (`int`): Total entangled pairs.
- `entangled_pairs_complete` (`int`): Pairs where both items are present.
- `causal_chain` (`Optional[List[str]]`): Expected causal chain steps.

**Returns**: `ContextQualityReport` with all dimension scores, warnings, and recommendations.

##### `get_trend`

```python
def get_trend(self, window: int = 10) -> Dict[str, List[float]]
```

Quality trends over recent evaluations. Returns dict with keys for each dimension plus `overall`.

##### `get_weakest_dimension`

```python
def get_weakest_dimension(self) -> Optional[Tuple[str, float]]
```

Return the weakest dimension name and score from the most recent evaluation.

##### `get_correlated_dimensions`

```python
def get_correlated_dimensions(self, threshold: float = 0.7) -> List[Tuple[str, str, float]]
```

Find dimension pairs with Pearson correlation above the threshold.

##### `is_healthy`

```python
def is_healthy(self) -> bool
```

Whether the most recent overall score is at or above the healthy threshold.

##### `get_history_count`

```python
def get_history_count(self) -> int
```

Number of evaluations in history.

---

## Dimension Scoring Details

| Dimension | Abbreviation | Scoring Method |
|-----------|-------------|----------------|
| Goal Retention | GRS | `\|goal_words & context_words\| / \|goal_words\|` |
| Information Density | IDI | `unique_entities / total_tokens * 100` (capped at 1.0) |
| Entanglement Completeness | EC | `complete_pairs / total_pairs` |
| Temporal Coherence | TC | `causal_steps_present / total_causal_steps` |
| Decision Preservation | DPR | `decisions_in_context / total_recent_decisions` |
| Anti-Hallucination | AHS | `grounded_tokens / total_tokens * 2.5` (tool + system content) |

Overall health is computed as a **weighted harmonic mean** of all dimensions.

---

## Health Labels

| Label | Overall Score Range |
|-------|-------------------|
| `"optimal"` | >= 0.7 |
| `"healthy"` | >= 0.5 |
| `"degrading"` | >= 0.3 |
| `"critical"` | < 0.3 |

---

## Example

```python
from corteX.engine.cognitive.context_quality import ContextQualityEngine

engine = ContextQualityEngine()

report = engine.evaluate(
    context_messages=[
        {"role": "system", "content": "Build a REST API for users"},
        {"role": "tool", "content": "[Tool] Read models.py: User class defined"},
        {"role": "assistant", "content": "I will create the User endpoint"},
    ],
    goal="Build a REST API for users",
    total_tokens=500,
    decision_log=[{"decision": "Create User endpoint"}],
)

print(f"Overall: {report.overall_health:.2f} ({report.health_label})")
print(f"GRS: {report.goal_retention_score:.2f}")
print(f"DPR: {report.decision_preservation_rate:.2f}")
print(f"Warnings: {report.warnings}")
print(f"Recommendations: {report.recommendations}")

# Track trends
trend = engine.get_trend(window=10)
weakest = engine.get_weakest_dimension()
if weakest:
    print(f"Weakest: {weakest[0]} = {weakest[1]:.2f}")
```

---

## See Also

- [Cognitive Context Pipeline](./cognitive-pipeline.md) -- Quality evaluation in Phase 6
- [Context Versioner](./versioner.md) -- Stores quality scores per version
- [Density Optimizer](./density-optimizer.md) -- Improves IDI dimension
- [Entanglement Graph](./entanglement.md) -- Feeds EC dimension
