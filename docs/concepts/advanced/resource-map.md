# Resource Homunculus

The Resource Homunculus dynamically allocates computational resources based on usage patterns, inspired by the somatosensory cortex's distorted body map where frequently used body parts receive disproportionately more cortical territory.

## What It Does

The Resource Homunculus tracks which components of the agent consume the most resources (tokens, time, tool calls) and reallocates budgets accordingly. Components that are used more get more territory; underused components shrink.

## Why: The Neuroscience Inspiration

!!! note "Brain Science: The Cortical Homunculus"
    The somatosensory cortex contains a "map" of the body surface, but it is distorted: the hands and lips have vastly more cortical territory than the back or legs, because they have denser sensory innervation and are used more frequently.

    Penfield's cortical homunculus illustrates this: the map reflects *functional importance*, not physical size. The same principle applies in motor cortex -- the fingers get more motor neurons than the trunk.

    Merzenich's experiments showed this map is plastic: when a finger is amputated, its cortical territory is gradually colonized by neighboring fingers. When a finger is used intensively (e.g., in Braille reading), its territory expands.

    The Resource Homunculus implements this for the SDK: components that are used more frequently and more successfully get larger resource budgets. Unused components shrink.

## How It Works

The Resource Homunculus maintains a territory allocation map that grows and shrinks based on usage:

```python
# Conceptual resource allocation
allocations = {
    "coding_column": 0.35,      # 35% of resources
    "research_column": 0.25,    # 25%
    "testing_column": 0.15,     # 15%
    "conversation_column": 0.10, # 10%
    "debugging_column": 0.15,    # 15%
}
```

Territory expands when a component:
- Is activated frequently
- Produces successful outcomes
- Consumes tokens efficiently

Territory shrinks when a component:
- Is rarely activated
- Produces poor outcomes
- Is wasteful with tokens

The reallocation follows Hebbian principles: "use it and succeed, grow it; neglect it, lose it."

## When It Activates

- **After each turn**: Usage statistics are updated
- **Periodically**: Territory is reallocated based on accumulated statistics
- **At session boundaries**: Historical usage informs initial allocations for the next session

## API Reference

The Resource Homunculus integrates with the `AttentionalFilter` and `FunctionalColumns` to inform processing budget allocation. See `corteX.engine.attention` for the `ProcessingBudget` class that encodes resource allocations.
