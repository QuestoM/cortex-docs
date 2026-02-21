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
- **GraphQueryEngine** -- Efficient concept graph queries
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
| `spreading_activation` | `(seed_nodes, depth, decay_per_hop, min_activation) -> Dict[str, float]` | Propagate activation through the graph (Collins & Loftus, 1975). |
| `hebbian_update_edges` | `() -> int` | Apply Hebbian learning to all edges based on current activations. |
| `inhibit_competing_concepts` | `(active_concept_id: str) -> List[str]` | Apply lateral inhibition from the active concept to competitors. |

#### `spreading_activation`

```python
def spreading_activation(
    self,
    seed_nodes: List[str],
    depth: int = 2,
    decay_per_hop: float = 0.5,
    min_activation: float = 0.01,
) -> Dict[str, float]
```

Propagate activation through the graph via spreading activation (Collins & Loftus, 1975). Starting from seed nodes, activation flows along edges to connected concepts. At each hop, activation is multiplied by edge strength and decay factor. INHIBITORY edges propagate negative activation (suppression).

**Parameters**:

- `seed_nodes` (`List[str]`): List of concept IDs to start from.
- `depth` (`int`): Maximum number of hops to propagate. Default: 2.
- `decay_per_hop` (`float`): Multiplicative decay per hop. Default: 0.5.
- `min_activation` (`float`): Minimum activation to propagate (below this, propagation stops). Default: 0.01.

**Returns**: `Dict[str, float]` mapping concept_id to final activation level for all affected concepts (including seeds).

#### `hebbian_update_edges`

```python
def hebbian_update_edges(self) -> int
```

Apply Hebbian learning to all edges based on current node activations. For each edge where both source and target concepts have activation > 0.05, the edge strength is increased via the saturating Hebbian rule. Implements the "fire together, wire together" principle at the concept level.

**Returns**: `int` -- Number of edges that were updated.

#### `inhibit_competing_concepts`

```python
def inhibit_competing_concepts(self, active_concept_id: str) -> List[str]
```

Apply lateral inhibition from the active (winning) concept to its competitors. Suppression occurs via two mechanisms:

1. **Explicit inhibitory edges**: Concepts connected by INHIBITORY edges are suppressed proportionally to the edge strength and winner's activation.
2. **Implicit overlap inhibition**: Active concepts with Jaccard member overlap > 0.2 are suppressed at half the strength of explicit inhibition.

This sharpens the activation pattern into a winner-take-most distribution.

**Parameters**:

- `active_concept_id` (`str`): ID of the most active (winning) concept.

**Returns**: `List[str]` -- List of concept IDs that were inhibited.

### Additional ConceptGraph Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `get_node` | `(concept_id: str) -> Optional[ConceptNode]` | Get a ConceptNode by ID. |
| `list_nodes` | `() -> List[ConceptNode]` | Get all ConceptNodes in the graph. |
| `get_concept_signature` | `(items: List[str]) -> Dict[str, float]` | Concept fingerprint for an item set (concept_id -> match_score). |
| `auto_discover_concepts` | `(co_occurrence_matrix, min_co_occurrences, ...) -> List[ConceptNode]` | Greedy agglomerative concept discovery from co-occurrence data. |
| `merge_similar_concepts` | `(threshold: float = 0.7) -> List[ConceptNode]` | Merge concepts with high Jaccard overlap. |
| `prune_weak_concepts` | `(min_usage: int = 3) -> List[str]` | Remove low-reliability, low-usage concepts. |
| `decay_all` | `(factor: float = 0.95) -> None` | Temporal decay of all activations and edge strengths. |
| `time_decay_edges` | `() -> int` | Time-based exponential decay of edges based on inactivity. |
| `get_stats` | `() -> Dict[str, Any]` | Comprehensive graph statistics. |
| `to_dict` / `from_dict` | | Serialization and deserialization. |

---

## Class: ConceptFormationEngine

Monitors tool/model co-occurrence patterns and automatically proposes new concepts when patterns stabilize. Models the biological distinction between short-term and long-term memory: a pattern must be rehearsed enough times to consolidate from labile short-term storage into stable long-term representation.

### Constructor

```python
ConceptFormationEngine(
    formation_threshold: int = 5,
    stabilization_count: int = 3,
    max_concept_size: int = 8,
    min_concept_size: int = 2,
)
```

**Parameters**:

- `formation_threshold` (`int`): Minimum co-occurrence count for a pair to contribute to concept formation. Default: 5.
- `stabilization_count` (`int`): Number of consecutive proposal rounds a candidate must survive before becoming a full concept. Default: 3.
- `max_concept_size` (`int`): Maximum members in a proposed concept. Default: 8.
- `min_concept_size` (`int`): Minimum members for a valid concept. Default: 2.

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record_co_occurrence` | `(items: List[str]) -> None` | Record that items were used together. Updates the symmetric co-occurrence matrix. |
| `propose_concepts` | `() -> List[ConceptNode]` | Analyze co-occurrence data and return newly-promoted ConceptNodes. |
| `get_candidates` | `() -> List[Dict[str, Any]]` | Get current candidate patterns not yet promoted to full concepts. |
| `get_co_occurrence_matrix` | `() -> Dict[str, Dict[str, int]]` | Get the raw co-occurrence matrix. |
| `get_stats` | `() -> Dict[str, Any]` | Formation engine statistics (recordings, proposals, formed counts). |
| `to_dict` / `from_dict` | | Serialization and deserialization. |

### Formation Process

1. **Record**: `record_co_occurrence()` is called on each step with the set of co-active items. All pairwise co-occurrence counts are incremented (symmetric).
2. **Propose**: `propose_concepts()` identifies strongly co-occurring clusters using union-find over pairs exceeding `formation_threshold`. Each cluster becomes a candidate.
3. **Stabilize**: Candidates that appear in consecutive proposal rounds accumulate stability. Once `stability >= stabilization_count`, the candidate is promoted to a full `ConceptNode`.
4. **Deduplicate**: Recently formed concepts are tracked to avoid re-proposing identical clusters.

---

## Class: GraphQueryEngine

Efficient query interface for the ConceptGraph. Provides high-level read-out patterns for downstream consumers without requiring direct graph traversal.

### Constructor

```python
GraphQueryEngine(graph: ConceptGraph)
```

**Parameters**:

- `graph` (`ConceptGraph`): The ConceptGraph to query.

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `find_concepts_for_item` | `(item_id: str) -> List[ConceptNode]` | Find all concepts containing an item, sorted by participation weight. |
| `find_related_concepts` | `(concept_id, depth=2, min_strength=0.05) -> List[Tuple[ConceptNode, float]]` | BFS traversal to find related concepts by graph distance. |
| `get_concept_overlap` | `(concept_a_id, concept_b_id) -> float` | Jaccard similarity between two concepts' member sets. |
| `get_weighted_overlap` | `(concept_a_id, concept_b_id) -> float` | Weighted overlap considering participation weights. |
| `get_activation_pattern` | `() -> Dict[str, float]` | Snapshot of all concept activations (the "state of mind"). |
| `get_active_concepts` | `(min_activation=0.1) -> List[Tuple[ConceptNode, float]]` | All currently active concepts above a threshold. |
| `get_concept_neighborhood` | `(concept_id: str) -> Dict[str, Any]` | Detailed neighborhood: members, outgoing/incoming edges, connections. |
| `cosine_similarity` | `(items_a, items_b) -> float` | Cosine similarity between two item sets in concept space. |

#### `find_concepts_for_item`

```python
def find_concepts_for_item(self, item_id: str) -> List[ConceptNode]
```

Find all concepts that include a given item as a member. Implements the distributed representation lookup: a single item participates in multiple concepts. Results sorted by participation weight descending.

#### `find_related_concepts`

```python
def find_related_concepts(
    self, concept_id: str, depth: int = 2, min_strength: float = 0.05
) -> List[Tuple[ConceptNode, float]]
```

BFS traversal from the source concept, accumulating connection strength along edges. INHIBITORY edges contribute negative strength (scaled by -0.5). Returns concepts reachable within `depth` hops sorted by accumulated strength descending.

#### `cosine_similarity`

```python
def cosine_similarity(self, items_a: List[str], items_b: List[str]) -> float
```

Compute cosine similarity between two item sets in concept space. Maps each item set to a concept activation vector (via `get_concept_signature`), then computes cosine similarity. Two item sets that share no items can still have high similarity if they activate similar concepts.

---

## Class: ConceptGraphManager

Main entry point for the Concept Graph subsystem. Wraps `ConceptGraph`, `ConceptFormationEngine`, and `GraphQueryEngine` into a unified API.

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

**Parameters**:

- `hebbian_learning_rate` (`float`): Learning rate for Hebbian edge updates. Default: 0.1.
- `formation_threshold` (`int`): Minimum co-occurrence count for concept formation. Default: 5.
- `stabilization_count` (`int`): Required stability rounds for candidate promotion. Default: 3.
- `decay_factor` (`float`): Base decay factor for activation (per maintenance). Default: 0.95.
- `merge_threshold` (`float`): Jaccard similarity threshold for merging concepts. Default: 0.7.
- `prune_min_usage` (`int`): Minimum usage before pruning is considered. Default: 3.
- `auto_form_concepts` (`bool`): Whether to automatically form concepts during maintenance. Default: True.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `graph` | `ConceptGraph` | Direct access to the underlying ConceptGraph. |
| `query` | `GraphQueryEngine` | Direct access to the GraphQueryEngine. |
| `formation` | `ConceptFormationEngine` | Direct access to the ConceptFormationEngine. |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `activate` | `(items: List[str]) -> List[ConceptNode]` | Activate, spread, inhibit, and Hebbian-update in one call. |
| `record_usage` | `(items, success=True, quality=1.0) -> None` | Record co-occurrence and update concept reliability. |
| `get_recommendations` | `(active_items: List[str]) -> Dict[str, Any]` | Get concept-based tool/model recommendations. |
| `maintenance` | `() -> Dict[str, Any]` | Decay, prune, merge, and auto-form concepts. |
| `create_concept` | `(name, members, metadata) -> ConceptNode` | Manually create a concept. |
| `add_inhibitory_edge` | `(concept_a_id, concept_b_id, strength=0.5) -> Optional[ConceptEdge]` | Create bidirectional inhibitory edge. |
| `add_hierarchical_edge` | `(parent_id, child_id, strength=0.5) -> Optional[ConceptEdge]` | Create parent-child edge (asymmetric: parent->child strong, child->parent 0.3x). |
| `get_stats` | `() -> Dict[str, Any]` | Full system statistics (graph + formation + manager). |
| `export_graph` / `from_export` | | Full serialization and restoration. |

#### `activate`

```python
def activate(self, items: List[str]) -> List[ConceptNode]
```

The primary per-step call. Performs 5 operations in sequence:

1. **Direct activation**: Find concepts whose members overlap with active items.
2. **Spreading activation**: Propagate from the top 5 activated concepts (depth=2, decay=0.4).
3. **Lateral inhibition**: The winner suppresses competing concepts.
4. **Edge creation**: Associative edges are created between all co-active concept pairs.
5. **Hebbian update**: All edges between co-active concepts are strengthened.

#### `record_usage`

```python
def record_usage(self, items: List[str], success: bool = True, quality: float = 1.0) -> None
```

Record that items were used together with an outcome. Updates co-occurrence data in the formation engine and reliability posteriors for matching concepts (match_score >= 0.2). For partial quality (success=True but quality < 1.0), a probabilistic update is applied.

#### `maintenance`

```python
def maintenance(self) -> Dict[str, Any]
```

Periodic maintenance (recommended every 10-50 steps). Performs:

1. Decay all activations and edge strengths.
2. Time-based exponential edge decay.
3. Prune weak concepts (reliability < 0.35, usage >= min_usage).
4. Merge similar concepts (Jaccard >= threshold).
5. Auto-form new concepts from co-occurrence data (if enabled).

Returns a summary dict of all actions taken.

---

## Example

```python
from corteX.engine.concepts import ConceptGraph, ConceptGraphManager

# -- Low-level: ConceptGraph directly --
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

# Spreading activation from activated seeds
seed_ids = [n.concept_id for n in activated]
activation_map = graph.spreading_activation(seed_ids, depth=2, decay_per_hop=0.5)

# Hebbian learning on co-active edges
updated = graph.hebbian_update_edges()
print(f"Updated {updated} edges")

# Lateral inhibition from the winner
if activated:
    inhibited = graph.inhibit_competing_concepts(activated[0].concept_id)
    print(f"Inhibited {len(inhibited)} competing concepts")

# -- High-level: ConceptGraphManager --
manager = ConceptGraphManager(auto_form_concepts=True)

# Create concepts
coding = manager.create_concept("complex_coding", {
    "code_interpreter": 0.9, "file_write": 0.7, "bash": 0.5,
})

# Per-step: activate + spread + inhibit + Hebbian update
activated = manager.activate(["code_interpreter", "file_write"])

# Record outcome
manager.record_usage(["code_interpreter", "file_write"], success=True, quality=0.9)

# Get recommendations
recs = manager.get_recommendations(["code_interpreter"])
print(f"Suggested tools: {recs['suggested_tools']}")
print(f"Model tier: {recs['suggested_model_tier']}")

# Periodic maintenance
results = manager.maintenance()
print(f"Pruned: {results['concepts_pruned']}, Merged: {results['concepts_merged']}")
```
