# Cortical Map Reorganizer API Reference

## Module: `corteX.engine.reorganization`

Continuous cortical map reorganization for tool/model/behavior repertoire management. Implements usage-driven plasticity, territory allocation, fusion of co-activated entities, and redistribution when entities are removed.

Neuroscience basis: Prof. Idan Segev, Lecture 4 - When two monkey fingers are surgically joined, the brain's cortical map reorganizes to represent them as one. In blind individuals, the visual cortex is colonized by tactile processing.

---

## Key Concepts

### Territory Allocation

Each entity (tool, model, behavior) occupies a fraction of the "cortical map" - a priority weight governing computational resource, attention, and preference allocation. All territories sum to 1.0.

### Usage-Driven Plasticity

Entities used frequently and successfully expand their territory. Disused entities shrink. Territory is recomputed using a weighted formula:

```
raw_score = 0.40 * usage_frequency + 0.35 * quality_score + 0.25 * recency_score
territory = raw_score / sum(raw_scores)
```

### Fusion (Merging)

When two entities are always co-activated (like surgically joined fingers), their representations merge into a single combined entity - reducing overhead and strengthening the association. Fusion triggers when co-occurrence strength exceeds `merge_threshold` (default: 0.80) with at least `merge_min_observations` (default: 5) observations.

### Redistribution

When an entity is removed or falls into disuse, its territory is redistributed to the most similar remaining entities - like the visual cortex being colonized by tactile processing in blind individuals.

### Scheduled Reorganization

Reorganization is not continuous (too expensive) but triggered by accumulated "reorganization pressure" from usage pattern shifts, entity additions/removals, and periodic timers.

---

## Supporting Classes

### `EntityType`

**Type**: `Enum`

| Value | Description |
|-------|-------------|
| `TOOL` | A tool entity |
| `MODEL` | A model entity |
| `BEHAVIOR` | A behavior entity |
| `MERGED` | A merged entity (created from fusion) |

### `ReorganizationEventType`

**Type**: `Enum`

Events that contribute to reorganization pressure.

| Value | Description |
|-------|-------------|
| `ENTITY_ADDED` | New entity registered |
| `ENTITY_REMOVED` | Entity removed |
| `PATTERN_SHIFT` | Usage patterns shifted significantly |
| `MERGE_CANDIDATE_FOUND` | Potential merge pair detected |
| `DISUSE_DETECTED` | Disused entities found |
| `PERIODIC_TICK` | Periodic timer fired |
| `MANUAL_TRIGGER` | Manual reorganization requested |

### `TerritoryAllocation`

**Type**: `@dataclass`

Represents how much "brain territory" a tool/model/behavior receives. Quality is tracked via a Beta distribution conjugate prior.

| Attribute | Type | Description |
|-----------|------|-------------|
| `entity_id` | `str` | Unique identifier |
| `entity_type` | `EntityType` | Classification (tool, model, behavior, merged) |
| `territory_size` | `float` | Fraction of total cortical map [0.0, 1.0] |
| `usage_count` | `int` | Total number of times this entity has been used |
| `usage_frequency` | `float` | Relative frequency compared to all entities |
| `last_used_turn` | `int` | Turn number when last activated |
| `quality_alpha` | `float` | Beta distribution alpha (success pseudo-count) |
| `quality_beta` | `float` | Beta distribution beta (failure pseudo-count) |
| `created_at` | `float` | Timestamp of entity registration |
| `metadata` | `Dict[str, Any]` | Arbitrary metadata |

---

## Main Class

### `CorticalMapReorganizer`

The main reorganization engine managing territory allocation, fusion, and redistribution.

#### Constructor

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

- `decay_factor` (float): Temporal decay factor for usage tracking. Default: 0.97
- `merge_threshold` (float): Co-occurrence strength threshold for merging. Default: 0.80
- `merge_min_observations` (int): Minimum observations before merge is considered. Default: 5
- `split_threshold` (float): Co-occurrence threshold below which merges are split. Default: 0.30
- `pressure_threshold` (float): Accumulated pressure needed to trigger reorganization. Default: 0.70
- `periodic_interval` (int): Turns between periodic reorganization checks. Default: 25
- `disuse_threshold_turns` (int): Turns of inactivity before entity is considered disused. Default: 20
- `similarity_exponent` (float): Exponent to sharpen similarity for redistribution. Default: 2.0

#### Entity Registration and Removal

##### `register_entity`

```python
register_entity(
    entity_id: str,
    entity_type: EntityType = EntityType.TOOL,
    initial_territory: float = 0.1,
    metadata: Optional[Dict[str, Any]] = None,
) -> TerritoryAllocation
```

Register a new entity in the cortical map. Returns the created `TerritoryAllocation`. If the entity is already registered, returns the existing allocation.

##### `remove_entity`

```python
remove_entity(entity_id: str) -> Optional[Dict[str, float]]
```

Remove an entity and redistribute its territory to the most similar remaining entities. Returns a redistribution map (`{entity_id: territory_increment}`) or `None` if the entity was not found.

#### Usage Recording

##### `record_usage`

```python
record_usage(
    entities: List[str],
    success: bool = True,
    quality: float = 1.0,
) -> None
```

Record that a set of entities were used together in this turn. Updates individual usage stats and the co-occurrence matrix. Unknown entities are auto-registered. When multiple entities are passed, co-occurrence is automatically tracked (no separate call needed).

#### Reorganization

##### `should_reorganize`

```python
should_reorganize() -> bool
```

Check whether reorganization should be triggered based on accumulated pressure.

##### `reorganize`

```python
reorganize() -> Dict[str, Any]
```

Perform a full reorganization cycle. Steps: (1) Apply temporal decay, (2) Recompute usage frequencies, (3) Adjust territory sizes based on frequency/quality/recency, (4) Detect and execute merges, (5) Handle disuse candidates (shrink territory), (6) Check if existing merges should be split, (7) Normalize territories to sum to 1.0, (8) Mark reorganization in scheduler.

Returns a dict with: `turn`, `merges`, `splits`, `disuse_shrinks`, `territory_changes`.

##### `maintenance`

```python
maintenance() -> None
```

Lightweight per-turn maintenance. Advances turn counters, applies quality decay, checks for pattern shifts and disuse, and updates reorganization pressure. Should be called once per agent turn.

#### Queries

##### `get_territory`

```python
get_territory(entity_id: str) -> Optional[TerritoryAllocation]
```

Get the territory allocation for a specific entity.

##### `get_territory_map`

```python
get_territory_map() -> Dict[str, TerritoryAllocation]
```

Return the current full cortical territory map as a dictionary of entity ID to `TerritoryAllocation`.

##### `get_merge_groups`

```python
get_merge_groups() -> List[List[str]]
```

Return all current merge groups as lists of source entity IDs.

##### `get_ranked_entities`

```python
get_ranked_entities() -> List[Tuple[str, float]]
```

Return all entities ranked by territory size (descending). This is the cortical homunculus ranking.

#### Statistics

##### `get_stats`

```python
get_stats() -> Dict[str, Any]
```

Comprehensive statistics: `entity_count`, `total_usages`, `merge_count`, `merge_history_count`, `reorganization_count`, `current_turn`, `current_pressure`, `territory_entropy`, `max_entropy`, `territory_uniformity`, `top_entities`, `disuse_candidates`, `fusion_candidates`, `creation_time`.

#### Persistence

##### `export_map`

```python
export_map() -> Dict[str, Any]
```

Export the full cortical map state as a serializable dictionary for persistence across sessions.

##### `from_dict`

```python
@classmethod
from_dict(data: Dict[str, Any]) -> CorticalMapReorganizer
```

Reconstruct a `CorticalMapReorganizer` from an exported dictionary.

---

## Usage Example

```python
from corteX.engine.reorganization import CorticalMapReorganizer, EntityType

reorg = CorticalMapReorganizer(decay_factor=0.97)

# Register entities
reorg.register_entity("code_writer", EntityType.TOOL, initial_territory=0.3)
reorg.register_entity("test_runner", EntityType.TOOL, initial_territory=0.2)
reorg.register_entity("browser", EntityType.TOOL, initial_territory=0.1)

# Record usage - multiple entities records co-occurrence automatically
reorg.record_usage(["code_writer", "test_runner"], success=True, quality=0.9)
reorg.record_usage(["code_writer"], success=True, quality=0.8)

# Per-turn maintenance (call every turn)
reorg.maintenance()

# Check if reorganization is needed
if reorg.should_reorganize():
    result = reorg.reorganize()
    # result contains merges, splits, disuse_shrinks, territory_changes

# Query territory map
territories = reorg.get_territory_map()
ranked = reorg.get_ranked_entities()

# Persistence
state = reorg.export_map()
restored = CorticalMapReorganizer.from_dict(state)
```

---

## See Also

- [Weight Engine API](./weights.md)
- [Targeted Modulator API](./modulator.md)
