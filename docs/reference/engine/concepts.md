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
| `spreading_activation` | `(seed_nodes: List[str], depth: int = 2, decay_per_hop: float = 0.5, min_activation: float = 0.01) -> Dict[str, float]` | Propagate activation through the graph via BFS spreading activation (Collins & Loftus 1975). Activation flows along edges, decaying per hop. Inhibitory edges propagate negative activation. Returns `{concept_id: activation}` for all affected nodes. |
| `hebbian_update_edges` | `() -> int` | Apply Hebbian learning to all edges based on current activations. Edges connecting co-active concepts (both > 0.05 activation) are strengthened ("fire together, wire together"). Returns the number of edges updated. |
| `inhibit_competing_concepts` | `(active_concept_id: str) -> List[str]` | Apply lateral inhibition from the most active concept. Suppresses concepts connected via INHIBITORY edges and concepts with significant member overlap (Jaccard > 0.2). Returns list of inhibited concept IDs. |
| `get_concept_signature` | `(items: List[str]) -> Dict[str, float]` | Get the concept fingerprint for a set of active items. Returns `{concept_id: match_score}` for all matching concepts. |
| `auto_discover_concepts` | `(co_occurrence_matrix, min_co_occurrences=3, min_cluster_size=2, max_cluster_size=10) -> List[ConceptNode]` | Automatically discover concepts from co-occurrence data using greedy agglomerative clustering. Returns newly created ConceptNodes. |

---

## Class: ConceptFormationEngine

Monitors tool/model co-occurrence patterns and automatically proposes new concepts when patterns stabilize. Implements a two-stage formation process (candidate to concept) modeled after the biological distinction between short-term and long-term memory.

### Constructor

```python
ConceptFormationEngine(
    formation_threshold: int = 5,
    stabilization_count: int = 3,
    max_concept_size: int = 8,
    min_concept_size: int = 2,
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `formation_threshold` | `int` | `5` | Minimum co-occurrence count for a pair to contribute to concept formation |
| `stabilization_count` | `int` | `3` | Number of consecutive proposal rounds a candidate must survive before becoming a full concept |
| `max_concept_size` | `int` | `8` | Maximum members in a proposed concept |
| `min_concept_size` | `int` | `2` | Minimum members for a valid concept |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record_co_occurrence` | `(items: List[str]) -> None` | Record that items were used together. Increments co-occurrence counts for all pairs. |
| `propose_concepts` | `() -> List[ConceptNode]` | Analyze co-occurrence data and return newly-formed ConceptNodes ready to be added to a ConceptGraph. |

---

## Class: GraphQueryEngine

Efficient query interface for the ConceptGraph. Provides high-level query patterns for downstream consumers.

### Constructor

```python
GraphQueryEngine(graph: ConceptGraph)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `find_concepts_for_item` | `(item_id: str) -> List[ConceptNode]` | Find all concepts that include a given item as a member, sorted by participation weight descending. |
| `find_related_concepts` | `(concept_id: str, depth: int = 2, min_strength: float = 0.05) -> List[Tuple[ConceptNode, float]]` | Find concepts related by graph distance via BFS traversal. Returns `(node, strength)` tuples sorted by strength descending. |

---

## Class: ConceptGraphManager

Main entry point for the Concept Graph subsystem. Wraps the ConceptGraph, ConceptFormationEngine, and GraphQueryEngine into a unified API.

### Constructor

```python
ConceptGraphManager(
    hebbian_learning_rate: float = 0.1,
    formation_threshold: int = 5,
    stabilization_count: int = 3,
    decay_factor: float = 0.95,
    merge_threshold: float = 0.7,
    prune_min_usage: int = 3,
    auto_form_concepts: bool = True,
)
```

### Properties

| Name | Type | Description |
|------|------|-------------|
| `graph` | `ConceptGraph` | Direct access to the underlying ConceptGraph. |
| `query` | `GraphQueryEngine` | Direct access to the GraphQueryEngine. |
| `formation` | `ConceptFormationEngine` | Direct access to the ConceptFormationEngine. |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `activate` | `(items: List[str]) -> List[ConceptNode]` | Activate concepts, run spreading activation, lateral inhibition, and Hebbian edge updates. Primary per-step call. |
| `record_usage` | `(items: List[str], success: bool = True, quality: float = 1.0) -> None` | Record item co-occurrence and update concept reliability posteriors. |
| `get_recommendations` | `(active_items: List[str]) -> Dict[str, Any]` | Get concept-based recommendations including suggested tools, model tier, related concepts, and confidence. |

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
