# Context Pyramid API Reference

## Module: `corteX.engine.cognitive.pyramid`

Multi-resolution context management. Maintains the same information at 4 resolution levels (R0 Full, R1 Standard, R2 Compact, R3 Micro). Dynamically selects resolution per token budget and goal proximity. Uses text compression heuristics -- no LLM calls. Top 10% items get R0, next 20% R1, next 30% R2, bottom 40% R3.

## Classes

### `ResolutionLevel`

**Type**: `IntEnum`

Context resolution levels from full to micro.

| Value | Name | Description | Token Ratio |
|-------|------|-------------|-------------|
| `0` | `R0_FULL` | Complete verbatim content | 1x |
| `1` | `R1_STANDARD` | Key sentences + structure | ~0.3x |
| `2` | `R2_COMPACT` | Entity-relationship summary | ~0.1x |
| `3` | `R3_MICRO` | Single-line semantic fingerprint | ~0.02x |

---

### `MultiResolutionItem`

**Type**: `@dataclass`

A context item stored at all four resolution levels.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_id` | `str` | Unique item identifier |
| `r0_content` | `str` | Full verbatim content |
| `r1_content` | `str` | Key sentences extracted |
| `r2_content` | `str` | Entity-relationship pairs |
| `r3_content` | `str` | Single-line fingerprint |
| `r0_tokens` | `int` | Token count at R0 |
| `r1_tokens` | `int` | Token count at R1 |
| `r2_tokens` | `int` | Token count at R2 |
| `r3_tokens` | `int` | Token count at R3 |
| `current_resolution` | `int` | Currently selected resolution level |
| `entities_referenced` | `List[str]` | Extracted entities from content |
| `goal_proximity` | `float` | Similarity to current goal (0.0-1.0) |
| `last_promoted_at` | `float` | Unix timestamp of last promotion |

---

### `ResolvedItem`

**Type**: `@dataclass`

An item at a chosen resolution level, ready for assembly.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_id` | `str` | Unique item identifier |
| `content` | `str` | Content at the resolved resolution |
| `tokens` | `int` | Token count of resolved content |
| `resolution` | `int` | Resolution level (0=R0, 1=R1, 2=R2, 3=R3) |
| `priority` | `float` | Priority score for ordering |

---

### `ContextPyramid`

Multi-resolution context with dynamic zoom level selection. Registers content at all 4 resolution levels and selects optimal resolution per item within a given token budget.

#### Constructor

```python
ContextPyramid(max_items: int = 500)
```

**Parameters**:

- `max_items` (int, default=500): Maximum items to store. Oldest items are evicted when exceeded.

#### Methods

##### `add_item`

```python
def add_item(
    self, item_id: str, full_content: str,
    goal_embedding_sim: float = 0.5
) -> MultiResolutionItem
```

Register content and generate all 4 resolution levels using heuristic compression.

**Parameters**:

- `item_id` (`str`): Unique identifier for this content.
- `full_content` (`str`): Full verbatim content (becomes R0).
- `goal_embedding_sim` (`float`, default=0.5): Goal proximity score.

**Returns**: `MultiResolutionItem` with all resolutions populated.

##### `generate_resolutions`

```python
def generate_resolutions(self, item_id: str) -> bool
```

Regenerate R1, R2, R3 from the R0 content of an existing item.

**Parameters**:

- `item_id` (`str`): Item to regenerate resolutions for.

**Returns**: `bool` -- `True` if regenerated, `False` if item not found.

##### `resolve`

```python
def resolve(
    self, items: List[ResolvedItem], budget_tokens: int, goal: str
) -> List[ResolvedItem]
```

Assign resolution levels per budget using priority tiers. Top 10% get R0, next 20% R1, next 30% R2, bottom 40% R3. Downgrades items that exceed remaining budget.

**Parameters**:

- `items` (`List[ResolvedItem]`): Items to resolve, ordered by priority.
- `budget_tokens` (`int`): Total token budget for all items.
- `goal` (`str`): Current goal text (for context).

**Returns**: `List[ResolvedItem]` -- Items with content set to the assigned resolution.

##### `get_at_resolution`

```python
def get_at_resolution(self, item_id: str, level: ResolutionLevel) -> Optional[str]
```

Get content at a specific resolution level.

**Parameters**:

- `item_id` (`str`): Item to retrieve.
- `level` (`ResolutionLevel`): Desired resolution.

**Returns**: `Optional[str]` -- Content at the requested level, or `None` if not found.

##### `get_item`

```python
def get_item(self, item_id: str) -> Optional[MultiResolutionItem]
```

Retrieve a multi-resolution item by ID.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Pyramid statistics with keys `total_items` and `by_resolution` (count per level).

---

## Compression Heuristics

All compression is performed without LLM calls:

| Level | Strategy | Example |
|-------|----------|---------|
| R1 (Standard) | Extract sentences containing entities, decisions, file paths, imports | `"Deployed app.py with 3 classes. Error in handler."` |
| R2 (Compact) | Entity-relationship pairs separated by `\|` | `"app.py: referenced \| handler: referenced"` |
| R3 (Micro) | First 8 words + `"..."` | `"Deployed the application server with three..."` |

---

## Example

```python
from corteX.engine.cognitive.pyramid import ContextPyramid, ResolutionLevel, ResolvedItem

pyramid = ContextPyramid(max_items=500)

# Register content at all resolutions
item = pyramid.add_item(
    "step_1",
    "The user deployed app.py to production using Docker. "
    "The deployment completed successfully with 3 replicas.",
    goal_embedding_sim=0.8,
)
print(f"R0: {item.r0_tokens} tokens")
print(f"R1: {item.r1_tokens} tokens")
print(f"R3: {item.r3_tokens} tokens")

# Resolve items to fit a budget
items = [
    ResolvedItem("step_1", item.r0_content, item.r0_tokens, 0, 0.9),
    ResolvedItem("step_2", "other content", 50, 0, 0.3),
]
resolved = pyramid.resolve(items, budget_tokens=200, goal="deploy app")

# Get specific resolution
r2 = pyramid.get_at_resolution("step_1", ResolutionLevel.R2_COMPACT)
```

---

## Performance Notes

- All compression is heuristic-based (regex) -- no LLM calls
- Token estimation uses `len(text) // 4` (4 characters per token)
- Oldest items are evicted automatically when exceeding `max_items`
- Resolution selection is deterministic based on priority rank

---

## See Also

- [Cognitive Context Pipeline](./cognitive-pipeline.md) -- Uses pyramid in Phase 2 (Resolve)
- [Density Optimizer](./density-optimizer.md) -- Complementary token compression
- [Context Quality Engine](./context-quality.md) -- Measures information density
