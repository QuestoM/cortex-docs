# Concept Graphs

The Concept Graph engine implements distributed semantic representations -- concepts are not stored as single nodes but as patterns of activation across a network. This enables flexible association, analogical reasoning, and robust retrieval even when input is noisy or incomplete.

## What It Does

The Concept Graph maintains a network of `ConceptNode` objects connected by `ConceptEdge` links. Each concept is represented by a distributed set of members (not a single point), and edges are Hebbian-learned associations. The graph supports:

- **Concept formation**: Automatic creation of concepts from co-occurring items
- **Spreading activation**: Activating one concept partially activates related concepts
- **Lateral inhibition**: Competing concepts suppress each other
- **Pruning**: Unused concepts and weak edges are removed

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Distributed Representations"
    The brain does not store concepts in single "grandmother cells" -- individual neurons dedicated to specific concepts. Instead, each concept is encoded as a distributed pattern of activation across many neurons, and each neuron participates in encoding many concepts.

    This distributed representation has several advantages:
    - **Graceful degradation**: Losing a few neurons does not destroy a concept
    - **Generalization**: Similar concepts have overlapping representations, enabling analogical reasoning
    - **Capacity**: The combinatorial space of distributed patterns far exceeds the number of neurons

    The Concept Graph implements this: each `ConceptNode` has a set of members (the distributed representation), and concepts with overlapping members are naturally related.

## How It Works

### Concept Nodes

```python
from corteX.engine.concepts import ConceptGraph

graph = ConceptGraph()

# Concepts are formed from co-occurring items
graph.add_concept("authentication", members={"jwt", "oauth2", "session", "token"})
graph.add_concept("authorization", members={"rbac", "permission", "role", "acl"})

# Overlapping members create natural relationships
# "token" appears in auth concepts, creating implicit association
```

### Edge Types

Two types of edges connect concepts:

| Type | Learned By | Example |
|------|-----------|---------|
| **ASSOCIATIVE** | Hebbian co-activation | "authentication" <-> "database" (often discussed together) |
| **HIERARCHICAL** | Explicit or inferred | "OAuth2" is-a "authentication method" |

### Spreading Activation

When a concept is activated, related concepts receive partial activation:

```python
# Activate "authentication" and see what spreads
activated = graph.spread_activation("authentication", strength=1.0)
# {
#   "authorization": 0.65,    # Strongly associated
#   "database": 0.42,         # Moderately associated
#   "user_management": 0.38,  # Related domain
# }
```

### Automatic Concept Formation

The graph can automatically form new concepts when it detects recurring co-occurrence patterns in context items. This mirrors how the brain forms new categories through repeated exposure.

### Pruning

Weak edges and isolated concepts are periodically pruned to prevent bloat:

```python
graph.prune(min_edge_strength=0.05, min_activation_count=2)
```

## When It Activates

- **During context enrichment**: The concept graph provides semantic associations to the context engine
- **During task classification**: Concept activations inform which functional column to engage
- **After tool calls**: New co-occurrences trigger Hebbian edge strengthening
- **Periodically**: Pruning removes stale concepts and weak associations

## API Reference

```python
from corteX.engine.concepts import (
    ConceptGraph,
    ConceptNode,
    ConceptEdge,
    EdgeType,
)
```
