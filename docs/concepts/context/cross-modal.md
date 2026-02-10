# Cross-Modal Associations

The Cross-Modal Association Engine connects information across different "modalities" -- code files, error patterns, documentation, user preferences, tool results -- using Hebbian learning. When two items appear together in the same context, their association strengthens. When associations go unused, they decay.

## What It Does

The engine maintains an associative graph where nodes are information items (tagged by modality) and edges are learned associations with Hebbian-updated strength:

```
CODE:auth_handler.py ---(0.85)--- ERROR:401_unauthorized
                     \
                      ---(0.72)--- DOCS:OAuth2_Setup_Guide
                      \
                       ---(0.45)--- TEST:test_auth_flow
```

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Cross-Modal Binding"
    "Everything is connected to everything through synapses. Almost any neuron you take, through a few relay stations, is connected to every other cell." -- Prof. Idan Segev

    In the brain, information from different sensory modalities (vision, hearing, touch) is processed in separate cortical areas. But these areas are densely interconnected, enabling cross-modal binding: you can recognize an object you have only touched by seeing it, and vice versa.

    Prof. Segev illustrates this with a compelling example: "The monkey trained to identify shapes by TOUCH cannot transfer that knowledge to VISION. But humans can, because in our brain there are such intensive connections between areas that I immediately know how to associate something I touch."

    The multi-sensory integration areas in the superior temporal sulcus and posterior parietal cortex bind visual, auditory, and tactile streams into unified percepts. The Cross-Modal Associator implements the same principle for software development modalities.

## How It Works

### Eight Modality Types

```python
from corteX.engine.cross_modal import ModalityType

# Information channels the agent processes
ModalityType.CODE              # Source code files
ModalityType.DOCUMENTATION     # Docs, READMEs, guides
ModalityType.ERROR_PATTERN     # Error messages, stack traces
ModalityType.USER_PREFERENCE   # User style/preference signals
ModalityType.TOOL_RESULT       # Tool execution outputs
ModalityType.CONVERSATION      # Conversation history
ModalityType.SCHEMA            # Data schemas, API specs
ModalityType.TEST_OUTPUT       # Test results
```

### Hebbian Binding (Co-activation)

When two items appear together in the same context, their association strengthens:

```python
from corteX.engine.cross_modal import CrossModalAssociator, ModalityType

associator = CrossModalAssociator(
    max_associations_per_item=20,
    hebbian_learning_rate=0.15,
    decay_halflife_hours=48.0,
)

# Items co-occur in the same context -> bind them
strength = associator.co_activate(
    ModalityType.CODE, "auth_handler.py",
    ModalityType.ERROR_PATTERN, "401_unauthorized",
    context_strength=1.0,   # Direct co-occurrence
)
# Creates bidirectional association, initial strength ~0.15
```

The Hebbian update formula includes saturation to prevent runaway weights:

```
delta = lr * context_strength * (1 - current_strength) * min(log(1 + count), 3) / 3
```

- `(1 - current_strength)`: stronger links are harder to strengthen further
- `log(1 + count)`: repeated co-activation has diminishing returns

### Retrieval

Query for associated items, optionally filtered by target modality:

```python
related = associator.get_associated(
    ModalityType.ERROR_PATTERN, "401_unauthorized",
    target_modality=ModalityType.CODE,
    max_results=5,
)
# [(ModalityType.CODE, "auth_handler.py", 0.85),
#  (ModalityType.CODE, "middleware.py", 0.42), ...]
```

### Spreading Activation

When one node is activated, associated nodes receive partial activation:

```python
activated = associator.spread_activation(
    ModalityType.CODE, "auth_handler.py",
    initial_strength=1.0,
    depth=2,              # How many hops to spread
    decay_per_hop=0.4,    # Activation decays 60% per hop
)
# {
#   (ERROR_PATTERN, "401_unauthorized"): 0.34,
#   (DOCS, "OAuth2_Setup_Guide"): 0.29,
#   (TEST, "test_auth"): 0.11,          # 2-hop indirect
# }
```

### Long-Term Depression (Decay)

Unused associations decay exponentially over time:

```python
# Run LTD pass (automatically called periodically)
pruned = associator.apply_ltd()
# Links that fall below half the minimum threshold are pruned entirely
```

### Associative Memory Index

The `AssociativeMemoryIndex` wraps the associator with item registration and metadata:

```python
from corteX.engine.cross_modal import AssociativeMemoryIndex

index = AssociativeMemoryIndex()

# Register items with metadata
index.register_item(ModalityType.CODE, "auth.py", {"path": "src/auth.py"})
index.register_item(ModalityType.ERROR_PATTERN, "401", {"msg": "Unauthorized"})

# Bind items
index.bind("auth.py", ModalityType.CODE, "401", ModalityType.ERROR_PATTERN)

# Query with metadata included in results
results = index.query(ModalityType.ERROR_PATTERN, "401")
# [{"modality": CODE, "key": "auth.py", "metadata": {"path": "src/auth.py"}, "strength": 0.35}]
```

### Context Enricher

The `ContextEnricher` bridges the association graph with the context engine. Before each LLM call, it enriches the context with relevant cross-modal associations:

```python
from corteX.engine.cross_modal import ContextEnricher

enricher = ContextEnricher(index=index)

# Enrich context with associations
annotations = enricher.enrich(
    active_items=[
        (ModalityType.CODE, "auth_handler.py"),
        (ModalityType.ERROR_PATTERN, "401_unauthorized"),
    ],
    max_annotations=5,
)

# As text for injection into context
text = enricher.enrich_as_text(active_items=[...])
# "[Cross-Modal Associations]
#  - code:auth_handler.py is strongly associated with docs:OAuth2_Guide (strength: 0.72)
#  - error:401 has episodic memory: 'Implement JWT auth' (success)"
```

The enricher also:
- **Co-activates all active items** with each other (Hebbian binding)
- **Queries episodic memory** for related past experiences
- **Queries semantic memory** for related domain knowledge
- **Deduplicates** results by target, keeping highest strength

### Serialization

The entire association graph serializes for persistence:

```python
data = associator.to_dict()
# Restore later
restored = CrossModalAssociator.from_dict(data)
```

## When It Activates

- **After every tool call**: Items from the tool call (code file, error, test output) are co-activated
- **Before LLM calls**: The ContextEnricher injects relevant associations into context
- **Periodically**: LTD pruning removes stale associations
- **On batch operations**: `record_co_occurrence_batch()` binds multiple items at once

## API Reference

```python
from corteX.engine.cross_modal import (
    CrossModalAssociator,
    AssociativeMemoryIndex,
    ContextEnricher,
    ModalityType,
    AssociationLink,
)
```
