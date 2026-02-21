# Columns

`corteX.engine.columns` -- Cortical column architecture for task specialization with winner-take-all competition.

---

## Overview

Modeled after functional columns in the primary visual cortex (V1), where groups of ~10,000
neurons fire selectively for specific stimulus features. Each `FunctionalColumn` bundles
tools, models, and weight configurations into a specialized processing unit. Columns compete
via winner-take-all dynamics with Thompson Sampling and lateral inhibition.

Architecture:

- **FunctionalColumn** -- A single specialized processing unit with Bayesian competence tracking
- **TaskClassifier** -- Maps incoming messages to column specializations (thalamic relay)
- **ColumnCompetition** -- Winner-take-all + lateral inhibition dynamics
- **ColumnManager** -- Full lifecycle: registration, selection, learning, merging, pruning

Default columns: `coding`, `debugging`, `testing`, `research`, `conversation`.

---

## Dataclass: FunctionalColumn

```python
from corteX.engine.columns import FunctionalColumn
```

### Attributes

| Name | Type | Description |
|------|------|-------------|
| `column_id` | `str` | Unique identifier. |
| `name` | `str` | Human-readable name (e.g., `"coding"`). |
| `specialization` | `str` | Domain of expertise. |
| `preferred_tools` | `List[str]` | Tool names this column excels with. |
| `preferred_model` | `str` | Best LLM model for this column's tasks. |
| `weight_overrides` | `Dict[str, float]` | Behavioral weight overrides when active. |
| `activation_level` | `float` | Current activation [0.0, 1.0]. |
| `competence` | `BetaDistribution` | Bayesian posterior over success probability. |
| `usage_count` | `int` | Total activations. |
| `success_count` | `int` | Successful outcomes. |
| `created_at` | `float` | Creation timestamp. |
| `last_activated` | `float` | Most recent activation timestamp. |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `activate` | `() -> None` | Set activation to 1.0, record timestamp, increment usage count. |
| `record_outcome` | `(success: bool, quality: float = 1.0) -> None` | Update Bayesian competence posterior. Graded quality via probabilistic update. |
| `record_coactivation` | `(other_id: str) -> None` | Track co-activation with another column (for merge detection). |
| `get_coactivation_count` | `(other_id: str) -> int` | Get number of co-activations with another column. Returns 0 if none recorded. |
| `decay` | `(factor: float = 0.95) -> None` | Apply temporal decay to activation and (slowly) competence. |
| `get_competence` | `() -> float` | Posterior mean of competence. |
| `get_competence_uncertainty` | `() -> float` | Posterior standard deviation. |
| `age_seconds` / `idle_seconds` | `() -> float` | Time since creation / last activation. |
| `to_dict` / `from_dict` | | Serialization. |

---

## Class: TaskClassifier

Maps incoming messages to column specializations. Combines keyword matching (40%),
regex pattern matching (30%), and learned Hebbian affinities (30%).

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `classify` | `(message: str, available_tools: Optional[List[str]] = None) -> List[Tuple[str, float]]` | Returns scored `(specialization, confidence)` pairs sorted descending. |
| `record_feedback` | `(message: str, specialization: str, success: bool) -> None` | Hebbian learning: words co-occurring with successful specializations strengthen. |
| `register_specialization` | `(specialization: str, keywords: List[str], patterns: Optional[List[str]] = None) -> None` | Add a new specialization with keywords and optional regex patterns. |

---

## Class: ColumnCompetition

Winner-take-all competition with Thompson Sampling and lateral inhibition.

### Constructor

```python
ColumnCompetition(
    activation_bonus_weight: float = 0.15,
    recruitment_threshold: float = 0.10,
    inhibition_factor: float = 0.3,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `compete` | `(columns, relevance_scores, available_tools) -> Optional[FunctionalColumn]` | Run competition. Returns winner or `None` (triggers recruitment). Score = relevance * Thompson_sample * activation_bonus * tool_fit. |
| `lateral_inhibit` | `(column) -> None` | Suppress a losing column's activation. |
| `get_activation_map` | `() -> Dict[str, float]` | Competition scores from the last round. |

---

## Class: ColumnManager

Full lifecycle manager for functional columns.

### Constructor

```python
ColumnManager(
    seed_defaults: bool = True,
    recruitment_threshold: float = 0.10,
    merge_coactivation_threshold: int = 20,
    prune_min_usage: int = 5,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `register_column` | `(column: FunctionalColumn) -> None` | Register a custom column. |
| `unregister_column` | `(column_id: str) -> Optional[FunctionalColumn]` | Remove a column. |
| `get_column` | `(column_id: str) -> Optional[FunctionalColumn]` | Retrieve by ID. |
| `list_columns` | `() -> List[FunctionalColumn]` | All registered columns. |
| `select_column` | `(task_type: str, message: str, available_tools: Optional[List[str]] = None) -> FunctionalColumn` | Main entry point. Classifies, competes, recruits if needed. Always returns a column. |
| `record_outcome` | `(column_id: str, success: bool, quality: float = 1.0, message: str = "") -> None` | Hebbian learning: update column competence and classifier affinities. |
| `get_column_recommendations` | `(column_id: str) -> Dict[str, Any]` | Get tools, model, weights, and competence for downstream systems. |
| `check_merge_candidates` | `() -> List[Tuple[str, str]]` | Find column pairs with high co-activation (Merzenich merging). |
| `merge_columns` | `(id_a: str, id_b: str) -> FunctionalColumn` | Merge two columns: union tools, average weights, pool competence. |
| `prune_weak_columns` | `(min_competence: float = 0.3) -> List[str]` | Remove low-competence non-default columns (use-it-or-lose-it). |
| `decay_all` | `(factor: float = 0.95) -> None` | Temporal decay on all columns. |
| `get_stats` | `() -> Dict[str, Any]` | Comprehensive system statistics. |
| `to_dict` / `from_dict` | | Full serialization. |

---

## Example

```python
from corteX.engine.columns import ColumnManager

mgr = ColumnManager(seed_defaults=True)

# Select a column for a task
winner = mgr.select_column(
    task_type="coding",
    message="Implement a retry decorator with exponential backoff",
    available_tools=["code_interpreter", "file_write"],
)
print(f"Selected: {winner.name}, competence: {winner.get_competence():.3f}")

# Record the outcome
mgr.record_outcome(winner.column_id, success=True, quality=0.9,
                    message="Implement a retry decorator")

# Get recommendations for the LLM pipeline
recs = mgr.get_column_recommendations(winner.column_id)
print(f"Model: {recs['preferred_model']}, Tools: {recs['preferred_tools']}")
```

```python
# Merging and pruning
candidates = mgr.check_merge_candidates()
for a, b in candidates:
    merged = mgr.merge_columns(a, b)
    print(f"Merged -> {merged.name}")

pruned = mgr.prune_weak_columns(min_competence=0.3)
print(f"Pruned {len(pruned)} weak columns")
```
