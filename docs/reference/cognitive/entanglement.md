# Entanglement Graph API Reference

## Module: `corteX.engine.cognitive.entanglement`

Tracks co-reference relationships between context items. Two items may each score low individually but become critical when present together (e.g., a schema definition + an error referencing that schema). Maintains an entity index and weighted edge graph. Boosts entangled pairs during context assembly and prevents pair separation.

## Classes

### `EntanglementEdge`

**Type**: `@dataclass`

Weighted co-reference relationship between two context items.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_a_id` | `str` | ID of the first entangled item |
| `item_b_id` | `str` | ID of the second entangled item |
| `score` | `float` | Entanglement strength (0.0-1.0), computed as `shared_entities * 0.25` |
| `shared_entities` | `List[str]` | Entity names shared between both items |
| `causal_direction` | `str` | One of `"a->b"`, `"b->a"`, `"bidirectional"` |
| `detected_at_step` | `int` | Step number when the edge was first created |
| `last_reinforced` | `float` | Unix timestamp of last reinforcement |

---

### `ScoredItem`

**Type**: `@dataclass`

Minimal scored item interface for entanglement enforcement.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_id` | `str` | Unique item identifier |
| `content` | `str` | Text content of the item |
| `priority` | `float` | Current priority score (0.0-1.0) |
| `tokens` | `int` | Estimated token count (default 0) |
| `metadata` | `Dict[str, Any]` | Additional metadata |

---

### `EntityExtractor`

Regex-based entity extraction for entanglement detection. Extracts file paths, API endpoints, Python identifiers, quoted strings, and environment-style variable names without external dependencies.

#### Methods

##### `extract`

```python
def extract(self, content: str) -> List[str]
```

Extract named entities from content text using regex patterns.

**Parameters**:

- `content` (`str`): Text to extract entities from.

**Returns**: `List[str]` -- Deduplicated list of extracted entities.

**Extraction patterns**:

| Pattern | Example Match |
|---------|---------------|
| File paths | `config/app.yaml`, `src\main.py` |
| API endpoints | `/api/users/list` |
| Python identifiers | `import os`, `class MyClass`, `def handler` |
| Quoted strings | `"my_config"`, `"user.name"` |
| ENV-style variables | `DATABASE_URL`, `API_KEY` |

---

### `EntanglementGraph`

Tracks co-reference relationships between context items. Nodes are context items. Edges are weighted by shared entity count. When evicting item X, checks entangled partners and boosts importance if the partner is still active.

#### Constructor

```python
EntanglementGraph(
    min_entanglement: float = 0.3,
    max_edges: int = 2000
)
```

**Parameters**:

- `min_entanglement` (float, default=0.3): Minimum edge score to consider a pair entangled. Edges below this threshold are ignored in queries.
- `max_edges` (int, default=2000): Maximum number of edges before weakest are trimmed.

#### Methods

##### `register_item`

```python
def register_item(self, item_id: str, content: str, step: int) -> None
```

Register a new context item. Extracts entities from content, computes co-reference edges with all existing items, and updates the entity index.

**Parameters**:

- `item_id` (`str`): Unique item identifier.
- `content` (`str`): Text content to extract entities from.
- `step` (`int`): Current execution step number.

##### `enforce_pairs`

```python
def enforce_pairs(self, items: List[ScoredItem]) -> List[ScoredItem]
```

Boost entangled pairs to ensure they stay together. Items whose entangled partner is included get a priority boost of up to `edge.score * 0.4`. Does not add missing partners.

**Parameters**:

- `items` (`List[ScoredItem]`): Context items to evaluate.

**Returns**: `List[ScoredItem]` -- Same list with boosted priorities.

##### `get_entangled_with`

```python
def get_entangled_with(self, item_id: str) -> List[Tuple[str, float]]
```

Return all items entangled with the given item.

**Parameters**:

- `item_id` (`str`): Item to query relationships for.

**Returns**: `List[Tuple[str, float]]` -- Pairs of `(partner_id, score)` sorted by descending score.

##### `get_missing_partners`

```python
def get_missing_partners(self, included_ids: Set[str]) -> List[Tuple[str, str, float]]
```

Find entangled partners that are missing from the included set.

**Parameters**:

- `included_ids` (`Set[str]`): Set of item IDs currently included in context.

**Returns**: `List[Tuple[str, str, float]]` -- Triples of `(missing_id, present_partner_id, entanglement_score)` sorted by descending score.

##### `eviction_check`

```python
def eviction_check(self, item_id: str, active_ids: Set[str], threshold: float = 0.6) -> bool
```

Check whether an item should be kept due to entanglement with active items.

**Parameters**:

- `item_id` (`str`): Item being considered for eviction.
- `active_ids` (`Set[str]`): Currently active context items.
- `threshold` (`float`, default=0.6): Minimum edge score to trigger keep.

**Returns**: `bool` -- `True` if the item should be kept (has a strong active partner).

##### `completeness_ratio`

```python
def completeness_ratio(self, included_ids: Set[str]) -> float
```

Fraction of entangled pairs where both items are present.

**Parameters**:

- `included_ids` (`Set[str]`): Currently included item IDs.

**Returns**: `float` -- Ratio from 0.0 (no complete pairs) to 1.0 (all pairs complete).

##### `get_co_occurring_entities`

```python
def get_co_occurring_entities(self, entity: str) -> List[str]
```

Get entities that co-occur with the given entity across items.

**Parameters**:

- `entity` (`str`): Entity name to look up.

**Returns**: `List[str]` -- Top 20 co-occurring entities, sorted by frequency.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Graph statistics for monitoring.

**Returns**: `Dict[str, Any]` with keys `total_edges`, `active_edges`, `entity_index_size`, `items_tracked`, `min_entanglement`.

---

## Example

```python
from corteX.engine.cognitive.entanglement import EntanglementGraph, ScoredItem

graph = EntanglementGraph(min_entanglement=0.3, max_edges=2000)

# Register items that share entities
graph.register_item("schema_def", 'class User defined in models.py', step=1)
graph.register_item("error_msg", 'Error in models.py: User field missing', step=2)

# Check entanglement
partners = graph.get_entangled_with("schema_def")
# [("error_msg", 0.5)]  -- they share "models.py" and "User"

# Boost priorities for entangled pairs
items = [
    ScoredItem(item_id="schema_def", content="...", priority=0.4),
    ScoredItem(item_id="error_msg", content="...", priority=0.3),
]
boosted = graph.enforce_pairs(items)

# Check if evicting schema_def would break an entanglement
keep = graph.eviction_check("schema_def", active_ids={"error_msg"})
# True -- schema_def is entangled with active error_msg

# Find missing partners
missing = graph.get_missing_partners(included_ids={"error_msg"})
# [("schema_def", "error_msg", 0.5)]
```

---

## Edge Scoring

Entanglement edge scores are computed as:

```
score = min(1.0, len(shared_entities) * 0.25)
```

This means:

| Shared Entities | Score |
|----------------|-------|
| 1 | 0.25 |
| 2 | 0.50 |
| 3 | 0.75 |
| 4+ | 1.00 |

---

## Performance Notes

- All methods are synchronous and perform no I/O
- Entity extraction uses pre-compiled regex patterns
- Edge trimming is automatic when exceeding `max_edges` (weakest edges removed first)
- Entity index provides O(1) lookup for co-occurrence queries
- Priority boost is capped at `edge.score * 0.4` to prevent runaway inflation

---

## See Also

- [Cognitive Context Pipeline](./cognitive-pipeline.md) -- Uses entanglement in Phase 3
- [Context Quality Engine](./context-quality.md) -- Measures entanglement completeness (EC dimension)
- [Active Forgetting Engine](./active-forgetting.md) -- Consults entanglement before eviction
