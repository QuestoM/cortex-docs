# Concepts

`corteX.engine.concepts` -- Distributed concept graph with Hebbian learning, spreading activation, and lateral inhibition.

---

## Overview

Based on the neuroscience insight that "there is no single grandmother cell, but there IS
a group of cells that represents grandmother" (Prof. Segev). Concepts are distributed
representations where individual members (tools, models, weight patterns) participate in
multiple concepts with different weights. The graph learns associative edges via Hebbian
rules and supports spreading activation for associative retrieval.

Architecture:

- **EdgeType** -- Enum: ASSOCIATIVE, HIERARCHICAL, INHIBITORY
- **ConceptNode** -- Emergent concept with distributed member set and Bayesian reliability
- **ConceptEdge** -- Hebbian-learned weighted edge with LTD decay
- **ConceptGraph** -- Full graph engine with activation, spreading, learning, merging, pruning
- **ConceptFormationEngine** -- Automatic concept discovery from co-occurrence
- **ConceptGraphManager** -- Main entry point wrapping all subsystems

---

## Enum: EdgeType

| Value | Description |
|-------|-------------|
| `ASSOCIATIVE` | Co-occurrence link. Strengthened by co-activation. |
| `HIERARCHICAL` | Parent-child relationship (e.g., `"software_engineering"` contains `"debugging_flow"`). |
| `INHIBITORY` | Competitive suppression between competing concepts. |

---

## Dataclass: ConceptNode

An emergent concept formed by a distributed group of members.

### Attributes

| Name | Type | Description |
|------|------|-------------|
| `concept_id` | `str` | Unique identifier. |
| `name` | `str` | Human-readable name (e.g., `"complex_coding"`). |
| `activation_level` | `float` | Current activation [0.0, 1.0]. |
| `members` | `Dict[str, float]` | Member ID to participation weight mapping. |
| `reliability` | `BetaDistribution` | Bayesian posterior over concept quality. |
| `usage_count` | `int` | Total activations. |
| `last_activated` | `float` | Timestamp of last activation. |
| `created_at` | `float` | Creation timestamp. |
| `metadata` | `Dict[str, Any]` | Arbitrary metadata. |

### Properties

| Name | Type | Description |
|------|------|-------------|
| `reliability_mean` | `float` | Posterior mean of reliability. |
| `reliability_uncertainty` | `float` | Posterior std of reliability. |
| `age_seconds` | `float` | Seconds since creation. |
| `idle_seconds` | `float` | Seconds since last activation. |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `activate` | `(strength: float = 1.0) -> None` | Activate with max-rule (accumulates evidence). |
| `deactivate` | `() -> None` | Force activation to 0.0. |
| `decay` | `(factor: float = 0.95) -> None` | Temporal decay. Below 0.001 snaps to 0. |
| `record_outcome` | `(success: bool) -> None` | Update Bayesian reliability posterior. |
| `add_member` / `remove_member` | | Manage member set. |
| `match_score` | `(active_items: Set[str]) -> float` | Weighted overlap score [0.0, 1.0]. |
| `get_weighted_signature` | `() -> Dict[str, float]` | Full (member_id, weight) vector. |
| `get_member_set` | `() -> FrozenSet[str]` | Member IDs as frozen set. |

---

## Dataclass: ConceptEdge

Weighted, typed edge between two ConceptNodes. Hebbian learning with saturating rule:
`delta = lr * a_source * a_target * (1.0 - strength)`.

### Attributes

| Name | Type | Description |
|------|------|-------------|
| `source_id` | `str` | Source concept ID. |
| `target_id` | `str` | Target concept ID. |
| `edge_type` | `EdgeType` | Type of the edge. |
| `strength` | `float` | Current weight [0.0, 1.0]. |
| `co_activation_count` | `int` | Times both endpoints were co-active. |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `hebbian_update` | `(source_activation, target_activation, lr=0.1) -> float` | Saturating Hebbian update with log-scaled co-activation bonus. |
| `decay` | `(factor: float = 0.995) -> None` | Per-step multiplicative decay. |
| `time_decay` | `(halflife_seconds: float = 86400.0) -> None` | Time-based exponential decay. |

---

## Class: ConceptGraph

Full graph with spreading activation, Hebbian learning, and lateral inhibition.

### Constructor

```python
ConceptGraph(
    hebbian_learning_rate: float = 0.1,
    decay_factor: float = 0.995,
    edge_halflife_seconds: float = 172800.0,
    min_edge_strength: float = 0.02,
    max_edges_per_node: int = 30,
    inhibition_strength: float = 0.3,
)
```

### Key Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `add_node` / `remove_node` | | Manage concepts in the graph. |
| `create_concept` | `(name, members, metadata) -> ConceptNode` | Create and add a new concept. |
| `add_edge` / `get_edge` | | Manage edges with auto-eviction of weakest. |
| `get_or_create_edge` | `(source_id, target_id, edge_type, initial_strength) -> Optional[ConceptEdge]` | Get existing or create new edge. |
| `activate` | `(items: List[str]) -> List[ConceptNode]` | Activate concepts matching active items via population coding readout. |

---

## Example

```python
from corteX.engine.concepts import ConceptGraph

graph = ConceptGraph()

# Create concepts with distributed members
coding = graph.create_concept("complex_coding", {
    "code_interpreter": 0.9, "file_write": 0.7, "bash": 0.5,
})
debugging = graph.create_concept("debugging_flow", {
    "code_interpreter": 0.8, "bash": 0.6, "browser": 0.4,
})

# Create associative edge
edge = graph.get_or_create_edge(coding.concept_id, debugging.concept_id)

# Activate concepts by providing active items
activated = graph.activate(["code_interpreter", "file_write"])
for node in activated:
    print(f"{node.name}: activation={node.activation_level:.3f}")
```
