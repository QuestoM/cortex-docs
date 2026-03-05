# Entanglement Graph

The `EntanglementGraph` tracks co-reference relationships between context items, ensuring that items which depend on each other are never separated during context assembly. Two items may each appear unimportant individually, but become critical when present together - for example, a schema definition and an error message that references that schema.

## How It Works

The entanglement graph models context items as nodes and their co-reference relationships as weighted edges. When a new item is registered, the built-in `EntityExtractor` extracts named entities (file paths, API endpoints, Python identifiers, quoted strings, and environment-style variables) and checks for overlap with entities from previously registered items.

### Edge Scoring

Each shared entity between two items adds 0.25 to their entanglement score (capped at 1.0). So two items sharing 4+ entities reach maximum entanglement. Edges are keyed by sorted item ID pairs to ensure uniqueness, and each edge tracks:

- The two item IDs
- The entanglement score (0.0 to 1.0)
- The list of shared entities
- Causal direction (`"a->b"`, `"b->a"`, or `"bidirectional"`)
- When it was first detected and last reinforced

### Entity Extraction

The `EntityExtractor` uses five regex patterns with no external dependencies:

- File paths: `[\w/\\]+\.\w{1,5}`
- API endpoints: `/api/[\w/]+`
- Python identifiers: `(?:import|from|class|def)\s+([\w.]+)`
- Quoted strings: `"([\w._-]+)"`
- Environment variables: `\b[A-Z_]{2,}[A-Z0-9_]*\b`

### Pair Enforcement

During context assembly, `enforce_pairs()` boosts the priority of items whose entangled partners are already in the context. The boost is `partner_score * 0.4`, ensuring strongly entangled items float up together. Items below the `min_entanglement` threshold (default 0.3) are ignored.

## Key Features

- **Automatic entity-based edge detection** - No manual annotation needed
- **Eviction protection** via `eviction_check()` - Returns `True` if an item has a strong partner still in context (above configurable threshold, default 0.6)
- **Missing partner detection** via `get_missing_partners()` - Identifies entangled items that should be brought back into context
- **Completeness ratio** - Measures what fraction of relevant entangled pairs have both items present
- **Co-occurrence discovery** via `get_co_occurring_entities()` - Finds entities that frequently appear alongside a given entity
- **Capacity management** - Automatically trims weakest edges when exceeding `max_edges` (default 2000)
- **Graph statistics** - Track total/active edges, entity index size, and items tracked

## Integration

The `EntanglementGraph` feeds directly into the `CognitiveContextPipeline` and the `ContextQualityEngine`. During context compilation, the pipeline calls `enforce_pairs()` to boost entangled items and `get_missing_partners()` to recover separated pairs. The quality engine uses the `completeness_ratio()` to compute the Entanglement Completeness (EC) dimension score.

The `ActiveForgettingEngine` consults `eviction_check()` before removing items, preserving memories that have strong active partners even if they would otherwise be evicted.

## Usage Example

```python
from corteX.engine.cognitive import EntanglementGraph, ScoredItem

graph = EntanglementGraph(min_entanglement=0.3, max_edges=2000)

# Register items - edges are created automatically from shared entities
graph.register_item("item_1", "class UserService in user_service.py", step=1)
graph.register_item("item_2", "Error in user_service.py: UserService not found", step=2)

# Check entanglement (they share "user_service.py" and "UserService")
partners = graph.get_entangled_with("item_1")
# [("item_2", 0.5)]  - 2 shared entities * 0.25 = 0.5

# Boost entangled items during assembly
items = [
    ScoredItem(item_id="item_1", content="...", priority=0.4),
    ScoredItem(item_id="item_2", content="...", priority=0.3),
]
boosted = graph.enforce_pairs(items)
# item_1.priority boosted because item_2 is present (and vice versa)

# Check if an item should be kept due to entanglement
should_keep = graph.eviction_check(
    "item_1", active_ids={"item_2"}, threshold=0.5
)  # True

# Find missing partners
missing = graph.get_missing_partners(included_ids={"item_1"})
# [("item_2", "item_1", 0.5)] if item_2 is not included

# Measure completeness
ratio = graph.completeness_ratio(included_ids={"item_1", "item_2"})
```

## See Also

- [Context Quality Engine](quality-engine.md) - Uses entanglement completeness as one of six quality dimensions
- [Active Forgetting](active-forgetting.md) - Consults entanglement before evicting items
- [Context Pyramid](pyramid.md) - Adjusts resolution levels considering entanglement
- [Predictive Pre-Loader](predictive-loader.md) - Uses entity co-occurrence for prediction
