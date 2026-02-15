# Cortical Map Reorganizer API Reference

## Module: `corteX.engine.reorganization`

Continuous cortical map reorganization for tool/model/behavior repertoire management. Implements usage-driven plasticity, territory allocation, fusion of co-activated entities, and redistribution when entities are removed.

Neuroscience basis: Prof. Idan Segev, Lecture 4 -- When two monkey fingers are surgically joined, the brain's cortical map reorganizes to represent them as one. In blind individuals, the visual cortex is colonized by tactile processing.

---

## Key Concepts

### Territory Allocation

Each entity (tool, model, behavior) occupies a fraction of the "cortical map" -- a priority weight governing computational resource, attention, and preference allocation. All territories sum to 1.0.

### Usage-Driven Plasticity

Entities used frequently and successfully expand their territory. Disused entities shrink. The reorganization formula:

```
new_territory = old_territory * (1 + learning_rate * reward_signal)
```

### Fusion (Merging)

When two entities are always co-activated (like surgically joined fingers), their representations merge into a single combined entity -- reducing overhead and strengthening the association. Fusion triggers when co-activation count exceeds threshold.

### Redistribution

When an entity is removed or falls into disuse, its territory is redistributed to the most similar remaining entities -- like the visual cortex being colonized by tactile processing in blind individuals.

### Scheduled Reorganization

Reorganization is not continuous (too expensive) but triggered by accumulated "reorganization pressure" from usage pattern shifts, entity additions/removals, and periodic timers.

---

## Classes

### `CorticalMapReorganizer`

The main reorganization engine managing territory allocation, fusion, and redistribution.

#### Constructor

```python
CorticalMapReorganizer(
    learning_rate: float = 0.05,
    fusion_threshold: int = 10,
    decay_rate: float = 0.01,
    min_territory: float = 0.01,
    reorganization_pressure_threshold: float = 1.0,
)
```

**Parameters**:

- `learning_rate` (float): Speed of territory adjustment. Default: 0.05
- `fusion_threshold` (int): Co-activation count for fusion. Default: 10
- `decay_rate` (float): Unused entity shrinkage rate. Default: 0.01
- `min_territory` (float): Minimum territory before pruning. Default: 0.01
- `reorganization_pressure_threshold` (float): Pressure needed to trigger reorg. Default: 1.0

#### Methods

##### `add_entity`

Add a new entity to the cortical map with initial territory allocation.

##### `remove_entity`

Remove an entity and redistribute its territory to the most similar remaining entities.

##### `record_usage`

Record a usage event for an entity. Increases territory based on reward signal and learning rate. Tracks co-activation patterns for potential fusion.

##### `record_co_activation`

Record that two entities were co-activated. Increments co-activation counter for potential fusion.

##### `check_fusion`

Check if any entity pairs should be fused based on co-activation threshold.

##### `apply_decay`

Apply decay to unused entities, shrinking their territory. Entities below `min_territory` may be flagged for removal.

##### `trigger_reorganization`

Force a reorganization cycle: apply decay, check fusion, normalize territories, and reset pressure accumulator.

##### `get_territory` / `get_all_territories`

Get territory allocation for a single entity or all entities.

##### `get_stats`

Returns: total_entities, total_reorganizations, pressure_accumulator, fusion_events, redistribution_events.

---

## Usage Example

```python
from corteX.engine.reorganization import CorticalMapReorganizer

reorg = CorticalMapReorganizer(learning_rate=0.05)

# Register entities
reorg.add_entity("code_writer", initial_territory=0.3)
reorg.add_entity("test_runner", initial_territory=0.2)
reorg.add_entity("browser", initial_territory=0.1)

# Record usage
reorg.record_usage("code_writer", reward=0.9)
reorg.record_usage("test_runner", reward=0.8)
reorg.record_co_activation("code_writer", "test_runner")

# Check if reorganization is needed
if reorg.should_reorganize():
    reorg.trigger_reorganization()

# code_writer territory has expanded, browser has shrunk
territories = reorg.get_all_territories()
```

---

## See Also

- [Weight Engine API](./weight-engine.md)
- [Targeted Modulator API](./modulator.md)
