# Cross-Modal

`corteX.engine.cross_modal` -- Cross-modal association engine with Hebbian binding across information modalities.

---

## Overview

Inspired by the brain's multi-sensory integration: "In our brain there are such intensive
connections between areas that I immediately know how to associate something I touch with
something I see" (Prof. Segev). This module connects information across different modalities
-- code, documentation, error patterns, user preferences, tool results -- via Hebbian learning.

Architecture:

- **ModalityType** -- Enum of information modalities
- **AssociationLink** -- Directed Hebbian-updated link between items
- **CrossModalAssociator** -- Core association graph engine
- **AssociativeMemoryIndex** -- Modality-aware item registry with query patterns
- **ContextEnricher** -- Enriches LLM context with cross-modal annotations

---

## Enum: ModalityType

| Value | Description |
|-------|-------------|
| `CODE` | Source code files and snippets. |
| `DOCUMENTATION` | Documentation and guides. |
| `ERROR_PATTERN` | Error messages and stack traces. |
| `USER_PREFERENCE` | User settings and behavioral preferences. |
| `TOOL_RESULT` | Output from tool invocations. |
| `CONVERSATION` | Conversation history entries. |
| `SCHEMA` | Data schemas and API specs. |
| `TEST_OUTPUT` | Test results and coverage data. |

---

## Dataclass: AssociationLink

A directed association between items from different modalities.

| Name | Type | Description |
|------|------|-------------|
| `source_modality` | `ModalityType` | Source item's modality. |
| `target_modality` | `ModalityType` | Target item's modality. |
| `source_key` | `str` | Source item identifier. |
| `target_key` | `str` | Target item identifier. |
| `strength` | `float` | Hebbian-updated weight [0.0, 1.0]. |
| `co_activation_count` | `int` | Times co-activated. |
| `pair_key` | `str` | (property) Canonical key for this link. |

---

## Class: CrossModalAssociator

Core association graph engine. Maintains bidirectional Hebbian-learned associations.

### Constructor

```python
CrossModalAssociator(
    max_associations_per_item: int = 20,
    min_strength_threshold: float = 0.1,
    hebbian_learning_rate: float = 0.15,
    decay_rate: float = 0.001,
    decay_halflife_hours: float = 48.0,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `co_activate` | `(modality_a, key_a, modality_b, key_b, context_strength=1.0) -> float` | "Fire together, wire together." Creates/strengthens bidirectional association. Returns new strength. |
| `get_associated` | `(modality, key, target_modality=None, min_strength=None, max_results=10) -> List[Tuple]` | Retrieve associated items sorted by strength. Returns `(target_modality, target_key, strength)` tuples. |
| `spread_activation` | `(modality, key, initial_strength=1.0, depth=1, decay_per_hop=0.4) -> Dict` | BFS spreading activation. Returns `{(modality, key): activation}` for all reached nodes. |
| `apply_ltd` | `(current_time=None) -> int` | Long-Term Depression: decay unused associations and prune weak ones. Returns pruned count. |
| `get_link` | `(src_modality, src_key, tgt_modality, tgt_key) -> Optional[AssociationLink]` | Get a specific link. |
| `get_stats` | `() -> Dict[str, Any]` | Graph statistics: total links, nodes, co-activations, avg strength. |

---

## Class: AssociativeMemoryIndex

Wraps `CrossModalAssociator` with item registration and modality-aware queries.

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `register_item` | `(modality, key, metadata=None) -> None` | Register an item with metadata. |
| `unregister_item` | `(modality, key) -> bool` | Remove an item and all its associations. |
| `bind` | `(key_a, modality_a, key_b, modality_b, context_strength=1.0) -> float` | Bind two items. Auto-registers if needed. |
| `query` | `(modality, key, target_modality=None, max_results=10, min_strength=None) -> List[Dict]` | Query for associated items. Returns dicts with `modality`, `key`, `metadata`, `strength`. |
| `query_cross_modal` | `(modality, key, target_modalities=None, max_per_modality=3) -> Dict[ModalityType, List[Dict]]` | Query across multiple target modalities at once. |
| `force_prune` | `() -> int` | Force immediate LTD pass. |

---

## Class: ContextEnricher

Enriches LLM context with cross-modal association annotations before inference.

### Constructor

```python
ContextEnricher(
    index: Optional[AssociativeMemoryIndex] = None,
    memory_fabric: Optional[Any] = None,
    max_annotations_per_item: int = 3,
    min_association_strength: float = 0.15,
    spreading_depth: int = 1,
    spreading_decay: float = 0.4,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `enrich` | `(active_items: List[Tuple[ModalityType, str]], max_annotations=10) -> List[Dict]` | Get structured annotation dicts with source/target, strength, and description. Also Hebbian-binds all active items. |
| `enrich_as_text` | `(active_items, max_annotations=10) -> str` | Text block suitable for system prompt injection. |
| `record_co_occurrence_batch` | `(items: List[Tuple[ModalityType, str]], context_strength=0.5) -> None` | Bulk Hebbian binding for items appearing together. |

---

## Example

```python
from corteX.engine.cross_modal import (
    CrossModalAssociator, ModalityType, ContextEnricher,
    AssociativeMemoryIndex,
)

associator = CrossModalAssociator()

# Co-activate items from different modalities
associator.co_activate(
    ModalityType.CODE, "auth_handler.py",
    ModalityType.ERROR_PATTERN, "401_unauthorized",
)

# Retrieve associations
related = associator.get_associated(
    ModalityType.ERROR_PATTERN, "401_unauthorized"
)
# -> [(ModalityType.CODE, "auth_handler.py", 0.35), ...]

# Use the enricher for LLM context
index = AssociativeMemoryIndex()
enricher = ContextEnricher(index=index)

text = enricher.enrich_as_text([
    (ModalityType.CODE, "auth_handler.py"),
    (ModalityType.ERROR_PATTERN, "401_unauthorized"),
])
print(text)
# [Cross-Modal Associations]
# - code:auth_handler.py is moderately associated with ...
```
