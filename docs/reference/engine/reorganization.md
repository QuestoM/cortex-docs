# Cortical Map Reorganizer API Reference

## Module: `corteX.engine.reorganization`

Continuous cortical map reorganization for tool/model/behavior repertoire management. Implements usage-driven plasticity, territory allocation, fusion of co-activated entities, and redistribution when entities are removed.

Neuroscience basis: Prof. Idan Segev, Lecture 4 -- When two monkey fingers are surgically joined, the brain's cortical map reorganizes to represent them as one. In blind individuals, the visual cortex is colonized by tactile processing.

---

## Key Concepts

### Territory Allocation

Each entity (tool, model, behavior) occupies a fraction of the "cortical map" -- a priority weight governing computational resource, attention, and preference allocation. All territories are normalized to sum to approximately 1.0.

### Usage-Driven Plasticity

Territory is recomputed using a weighted formula based on three signals:

```
raw_score = 0.40 * usage_frequency + 0.35 * quality_score + 0.25 * recency_score
territory = raw_score / sum(all_raw_scores)
```

Quality is tracked via a **Beta distribution** conjugate prior (`alpha / (alpha + beta)`), providing principled uncertainty quantification.

### Fusion (Merging)

When two entities consistently co-occur (co-occurrence strength exceeds threshold, default 0.80), their representations merge into a single combined entity. Merges are reversible -- if co-occurrence drops below the split threshold (default 0.30), the merge is undone.

### Redistribution

When an entity is removed, its territory is redistributed to the most similar remaining entities using cosine + Jaccard similarity on co-occurrence usage vectors. A similarity exponent controls sharpness of redistribution.

### Scheduled Reorganization

Reorganization is triggered by accumulated "reorganization pressure" from events (entity added/removed, pattern shifts, disuse, periodic ticks). When pressure crosses the threshold (default 0.70), a full reorganization cycle runs.

---

## Enums

### `EntityType`

Types of entities that can occupy cortical territory.

| Value | Description |
|-------|------------|
| `TOOL` | A tool entity (default) |
| `MODEL` | A model entity |
| `BEHAVIOR` | A behavior entity |
| `MERGED` | A fused entity created by merging |

### `ReorganizationEventType`

Events that contribute to reorganization pressure.

| Value | Pressure |
|-------|----------|
| `ENTITY_ADDED` | +0.15 |
| `ENTITY_REMOVED` | +0.25 |
| `PATTERN_SHIFT` | +0.20 |
| `MERGE_CANDIDATE_FOUND` | +0.10 |
| `DISUSE_DETECTED` | +0.08 |
| `PERIODIC_TICK` | +0.03 |
| `MANUAL_TRIGGER` | +1.00 (immediate) |

---

## Data Classes

### `TerritoryAllocation`

Represents how much "brain territory" a tool/model/behavior receives. Quality is tracked via a Beta distribution conjugate prior.

```python
@dataclass
class TerritoryAllocation:
    entity_id: str
    entity_type: EntityType = EntityType.TOOL
    territory_size: float = 0.1          # Fraction of cortical map [0.0, 1.0]
    usage_count: int = 0
    usage_frequency: float = 0.0
    last_used_turn: int = 0
    quality_alpha: float = 1.0           # Beta distribution alpha
    quality_beta: float = 1.0            # Beta distribution beta
    created_at: float = time.time()
    metadata: Dict[str, Any] = field(default_factory=dict)
```

**Properties**:

| Property | Type | Description |
|----------|------|-------------|
| `quality_score` | `float` | Expected quality: `alpha / (alpha + beta)` |
| `quality_uncertainty` | `float` | Std deviation of the Beta posterior |
| `total_quality_observations` | `float` | Effective observation count (excluding prior) |

**Methods**:

| Method | Description |
|--------|-------------|
| `update_quality(success: bool, quality: float = 1.0)` | Update Beta distribution with a new observation |
| `decay_quality(factor: float = 0.97)` | Apply temporal decay, increasing uncertainty |
| `to_dict() -> Dict` | Serialize to dictionary |
| `from_dict(data) -> TerritoryAllocation` | Reconstruct from dictionary (classmethod) |

### `MergeRecord`

Record of a territory merge operation, capturing pre-merge state for undo.

```python
@dataclass
class MergeRecord:
    merged_entity_id: str
    source_ids: Tuple[str, ...]
    source_allocations: Dict[str, Dict[str, Any]]
    co_occurrence_strength: float = 0.0
    merge_turn: int = 0
    merge_timestamp: float = time.time()
```

### `MergedEntity`

A fused entity created by merging two or more co-occurring entities.

```python
@dataclass
class MergedEntity:
    entity_id: str
    source_ids: Tuple[str, ...]
    territory: TerritoryAllocation
    merge_record: Optional[MergeRecord] = None
```

---

## Supporting Classes

### `UsageTracker`

Tracks entity usage patterns, co-occurrence, and temporal dynamics. Builds the co-occurrence matrix used for merge detection.

```python
UsageTracker(decay_factor: float = 0.97)
```

| Method | Description |
|--------|-------------|
| `advance_turn()` | Advance the internal turn counter |
| `record_usage(entity_id, success=True, quality=1.0)` | Record a single entity usage event |
| `record_co_usage(entities: List[str])` | Record co-occurrence for a set of entities used together |
| `get_co_occurrence_strength(a, b) -> float` | Normalized co-occurrence strength in [0.0, 1.0] |
| `get_fusion_candidates(threshold, min_observations) -> List[Tuple]` | Find entity pairs that co-occur enough to merge |
| `get_disuse_candidates(threshold_turns) -> List[str]` | Find entities not used for N turns |
| `get_usage_frequency(entity_id) -> float` | Relative usage frequency |
| `get_usage_vector(entity_id) -> Dict[str, float]` | Co-occurrence-based "usage fingerprint" |
| `apply_decay()` | Apply temporal decay to all tracking counters |
| `remove_entity(entity_id)` | Remove all tracking data for an entity |

### `TerritoryMerger`

Handles merging and splitting of entity representations.

```python
TerritoryMerger(
    co_occurrence_threshold: float = 0.80,
    min_observations: int = 5,
    split_threshold: float = 0.30,
)
```

| Method | Description |
|--------|-------------|
| `should_merge(a, b, co_occurrence_strength, a_observations, b_observations) -> bool` | Decide whether two entities should be merged |
| `merge(a, b, alloc_a, alloc_b, co_occurrence_strength, current_turn) -> MergedEntity` | Execute a merge, creating a combined entity |
| `split(merged_entity_id) -> Optional[Tuple[TerritoryAllocation, TerritoryAllocation]]` | Undo a merge, restoring the original entities |
| `is_merged(entity_id) -> bool` | Check if an entity is part of a merged group |
| `get_merged_entity(merged_id) -> Optional[MergedEntity]` | Get a merged entity by ID |
| `get_merge_for_source(source_id) -> Optional[str]` | Get merged entity ID containing a source |
| `get_merge_history() -> List[MergeRecord]` | Full merge/split history |
| `get_merge_groups() -> List[List[str]]` | All current merge groups |

### `TerritoryRedistributor`

Redistributes territory from removed or disused entities to the most similar remaining entities.

```python
TerritoryRedistributor(
    similarity_exponent: float = 2.0,
    min_similarity: float = 0.05,
)
```

| Method | Description |
|--------|-------------|
| `compute_similarity(a_vector, b_vector) -> float` | Cosine + Jaccard similarity (70/30 blend) |
| `redistribute(removed_entity_id, removed_territory, remaining_entities) -> Dict[str, float]` | Compute territory redistribution map |
| `apply_redistribution(redistribution_map, territories)` | Apply redistribution to live territory allocations |

### `ReorganizationScheduler`

Decides when to trigger cortical map reorganization based on accumulated pressure.

```python
ReorganizationScheduler(
    pressure_threshold: float = 0.70,
    periodic_interval: int = 25,
    pressure_decay: float = 0.95,
)
```

| Method | Description |
|--------|-------------|
| `record_event(event_type, details=None)` | Record a pressure-contributing event |
| `advance_turn()` | Advance turn, apply decay, check periodic tick |
| `should_reorganize() -> bool` | Check if pressure exceeds threshold |
| `mark_reorganized()` | Reset pressure after reorganization |
| `get_pressure() -> float` | Current pressure in [0.0, 1.0] |
| `get_recent_events(n=20) -> List[Dict]` | Recent scheduler events |

---

## Main Class: `CorticalMapReorganizer`

The main orchestrator that coordinates all reorganization subsystems.

### Constructor

```python
CorticalMapReorganizer(
    decay_factor: float = 0.97,
    merge_threshold: float = 0.80,
    merge_min_observations: int = 5,
    split_threshold: float = 0.30,
    pressure_threshold: float = 0.70,
    periodic_interval: int = 25,
    disuse_threshold_turns: int = 20,
    similarity_exponent: float = 2.0,
)
```

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `decay_factor` | `float` | `0.97` | Temporal decay for usage tracking and quality |
| `merge_threshold` | `float` | `0.80` | Co-occurrence strength threshold for merging |
| `merge_min_observations` | `int` | `5` | Minimum observations before merge is considered |
| `split_threshold` | `float` | `0.30` | Co-occurrence below this triggers split |
| `pressure_threshold` | `float` | `0.70` | Pressure needed to trigger reorganization |
| `periodic_interval` | `int` | `25` | Turns between periodic pressure ticks |
| `disuse_threshold_turns` | `int` | `20` | Turns of inactivity before disuse detection |
| `similarity_exponent` | `float` | `2.0` | Sharpness of similarity-based redistribution |

### Methods

#### `register_entity`

Register a new entity in the cortical map.

```python
def register_entity(
    self,
    entity_id: str,
    entity_type: EntityType = EntityType.TOOL,
    initial_territory: float = 0.1,
    metadata: Optional[Dict[str, Any]] = None,
) -> TerritoryAllocation
```

#### `remove_entity`

Remove an entity and redistribute its territory to the most similar remaining entities.

```python
def remove_entity(self, entity_id: str) -> Optional[Dict[str, float]]
```

Returns a redistribution map (`entity_id -> territory increment`) or `None` if not found.

#### `record_usage`

Record that a set of entities were used together in this turn. Unknown entities are auto-registered.

```python
def record_usage(
    self,
    entities: List[str],
    success: bool = True,
    quality: float = 1.0,
) -> None
```

#### `maintenance`

Lightweight per-turn maintenance. Advances turn counters, applies quality decay, checks for pattern shifts and disuse, and updates reorganization pressure. Call once per agent turn.

```python
def maintenance(self) -> None
```

#### `should_reorganize`

Check whether reorganization should be triggered (pressure exceeds threshold or safety-valve timeout).

```python
def should_reorganize(self) -> bool
```

#### `reorganize`

Perform a full reorganization cycle: decay, recompute frequencies, adjust territories, detect/execute merges, handle disuse, check splits, normalize. Returns a dict of actions taken.

```python
def reorganize(self) -> Dict[str, Any]
```

**Return keys**: `turn`, `merges`, `splits`, `disuse_shrinks`, `territory_changes`.

#### `get_territory_map`

```python
def get_territory_map(self) -> Dict[str, TerritoryAllocation]
```

#### `get_territory`

```python
def get_territory(self, entity_id: str) -> Optional[TerritoryAllocation]
```

#### `get_ranked_entities`

Return all entities ranked by territory size (descending).

```python
def get_ranked_entities(self) -> List[Tuple[str, float]]
```

#### `get_merge_groups`

```python
def get_merge_groups(self) -> List[List[str]]
```

#### `get_stats`

Comprehensive statistics: `entity_count`, `total_usages`, `merge_count`, `merge_history_count`, `reorganization_count`, `current_turn`, `current_pressure`, `territory_entropy`, `territory_uniformity`, `top_entities`, `disuse_candidates`, `fusion_candidates`.

```python
def get_stats(self) -> Dict[str, Any]
```

#### `export_map` / `from_dict`

Full state serialization for persistence across sessions.

```python
def export_map(self) -> Dict[str, Any]

@classmethod
def from_dict(cls, data: Dict[str, Any]) -> CorticalMapReorganizer
```

---

## Usage Example

```python
from corteX.engine.reorganization import (
    CorticalMapReorganizer,
    EntityType,
)

reorg = CorticalMapReorganizer(
    decay_factor=0.97,
    merge_threshold=0.80,
    pressure_threshold=0.70,
)

# Register entities with types
reorg.register_entity("web_search", EntityType.TOOL, initial_territory=0.3)
reorg.register_entity("summarize", EntityType.TOOL, initial_territory=0.2)
reorg.register_entity("gpt4", EntityType.MODEL, initial_territory=0.2)

# Record usage -- entities used together in one turn
reorg.record_usage(["web_search", "summarize"], success=True, quality=0.9)
reorg.record_usage(["gpt4"], success=True, quality=0.85)

# Per-turn maintenance (advances turn, checks disuse/pressure)
reorg.maintenance()

# Check if reorganization is needed
if reorg.should_reorganize():
    actions = reorg.reorganize()
    print(f"Merges: {actions['merges']}, Shrinks: {actions['disuse_shrinks']}")

# Query territory allocation
territory_map = reorg.get_territory_map()
for eid, alloc in territory_map.items():
    print(f"{eid}: size={alloc.territory_size:.3f}, quality={alloc.quality_score:.3f}")

# Ranked entities (most territory first)
ranked = reorg.get_ranked_entities()

# Remove an entity -- territory redistributed to similar neighbors
redistribution = reorg.remove_entity("web_search")

# Persist state across sessions
state = reorg.export_map()
restored = CorticalMapReorganizer.from_dict(state)
```

---

## See Also

- [Weight Engine API](./weight-engine.md)
- [Targeted Modulator API](./modulator.md)
