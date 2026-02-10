# Cortical Map Reorganization

The Cortical Map Reorganizer dynamically reallocates processing territory among functional columns based on usage patterns. Columns that handle frequent, successful tasks expand their territory. Unused columns shrink and may eventually be reclaimed.

## What It Does

The reorganizer manages territory allocation across functional columns:

- **Territory expansion**: Successful, frequently-used columns gain resources
- **Territory shrinkage**: Underused columns lose resources
- **Territory merging**: Columns that co-activate frequently are fused
- **Territory redistribution**: When a column is removed, its territory is reassigned

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Cortical Map Plasticity"
    Michael Merzenich's landmark experiments showed that the cortical map is not fixed -- it reorganizes based on experience:

    - **Amputation**: When a monkey's finger is amputated, the cortical territory that represented that finger is gradually colonized by neighboring fingers within weeks.
    - **Training**: When a monkey is trained to use one finger intensively, that finger's cortical territory expands at the expense of neighboring territories.
    - **Fusion**: When two fingers are surgically fused (sewn together), their cortical representations merge into a single territory -- because the brain only experiences them as one unit.

    The principle is "use it or lose it": cortical territory is allocated proportionally to functional importance, and this allocation is continuously updated.

## How It Works

### Territory Tracking

```python
from corteX.engine.reorganization import CorticalMapReorganizer

reorganizer = CorticalMapReorganizer()

# Track usage of each column
reorganizer.record_usage("coding", success=True, tokens=1500)
reorganizer.record_usage("research", success=True, tokens=2000)
reorganizer.record_usage("debugging", success=False, tokens=800)
```

### Scheduled Reorganization

Territory is redistributed periodically based on accumulated usage data:

```python
# Run reorganization
changes = reorganizer.reorganize()
# Territory allocated proportional to:
#   usage_frequency * success_rate * token_efficiency
```

### Territory Merging

When two columns are consistently co-activated, they are candidates for merging:

```python
# After detecting high co-activation between coding and debugging:
reorganizer.merge("coding", "debugging")
# Creates a combined "coding_debugging" territory
```

### Territory Redistribution

When a column is pruned, its territory is redistributed to remaining columns:

```python
reorganizer.remove_territory("unused_column")
# Territory redistributed proportionally to remaining columns
```

## When It Activates

- **After each task**: Usage statistics are updated
- **Periodically**: The `ReorganizationScheduler` triggers reallocation
- **On column merge/prune**: Territory is immediately redistributed
- **At session boundaries**: Historical usage informs initial allocations

## API Reference

```python
from corteX.engine.reorganization import (
    CorticalMapReorganizer,
    TerritoryAllocation,
    UsageTracker,
    TerritoryMerger,
    TerritoryRedistributor,
    ReorganizationScheduler,
)
```
