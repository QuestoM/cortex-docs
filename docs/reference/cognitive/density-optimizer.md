# Density Optimizer API Reference

## Module: `corteX.engine.cognitive.density_optimizer`

Information density optimization for context items. Packs maximum semantic content per token using structured encoding. Converts prose to key:value pairs, narrative tool results to structured tables, full stack traces to error class + message, and verbose conversation history to decision-points-only. Achieves 3-5x density improvement without semantic information loss.

## Classes

### `DensityRule`

**Type**: `@dataclass`

A compression rule for density optimization.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `rule_type` | `str` | Rule category: `"preference"`, `"tool_result"`, `"error"`, `"conversation"` |
| `pattern` | `str` | Regex pattern to match |
| `replacement_template` | `str` | Template for compressed form |

---

### `ScoredItem`

**Type**: `@dataclass`

Minimal scored item interface for density optimization.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_id` | `str` | Unique item identifier |
| `content` | `str` | Text content |
| `priority` | `float` | Priority score |
| `tokens` | `int` | Estimated token count |
| `metadata` | `Dict[str, Any]` | Additional metadata |

---

### `DensityOptimizer`

Packs maximum semantic content per token by applying four compression strategies in sequence, then domain abbreviations and inline redundancy removal.

#### Constructor

```python
DensityOptimizer(custom_rules: Optional[List[DensityRule]] = None)
```

**Parameters**:

- `custom_rules` (`Optional[List[DensityRule]]`): Additional custom compression rules.

#### Methods

##### `optimize`

```python
def optimize(self, items: List[ScoredItem]) -> List[ScoredItem]
```

Compress each item for maximum information density. Applies all compression strategies in sequence.

**Parameters**:

- `items` (`List[ScoredItem]`): Items to optimize.

**Returns**: `List[ScoredItem]` -- New items with compressed content and updated token counts.

**Compression pipeline**:

1. Preference compression (prose to key:value)
2. Tool result compression (narrative to structured)
3. Error compression (stack traces to error line)
4. Conversation compression (full exchange to decision points)
5. Domain abbreviations
6. Inline redundancy removal

##### `estimate_gain`

```python
def estimate_gain(self, original_tokens: int, optimized_tokens: int) -> float
```

Compute compression gain ratio.

**Returns**: `float` -- Ratio where 1.0 = no gain, 3.0 = 3x denser.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: `Dict[str, Any]` with keys `tokens_saved`, `items_optimized`, `avg_savings`.

---

## Compression Strategies

### 1. Preference Compression

Converts preference prose to structured key:value pairs.

| Original | Compressed |
|----------|-----------|
| `The user prefers PostgreSQL as their database` | `[PREF] database:PostgreSQL` |
| `Prefer Python over JavaScript` | `[PREF] use:Python (not JavaScript)` |
| `The API should use REST` | `[REQ] API:REST` |

### 2. Tool Result Compression

Converts narrative tool output to structured `key:value` pairs separated by `|`.

| Original | Compressed |
|----------|-----------|
| `The file contains 150 lines` | `file:150 lines` |
| `The status was running` | `status:running` |

### 3. Error Compression

Extracts just the error class and message from full stack traces.

| Original | Compressed |
|----------|-----------|
| Full Python traceback (20+ lines) | `[ERR] ValueError: invalid literal` |
| Generic error text | `[ERR] Connection timeout \| Failed to connect` |

### 4. Conversation Compression

Keeps only lines containing decision keywords (`decided`, `chose`, `created`, `deployed`, etc.) and removes filler phrases (`ok`, `sure`, `got it`, `thanks`).

---

## Domain Abbreviations

The optimizer applies 24 standard abbreviations:

| Long Form | Short Form |
|-----------|------------|
| `authentication` | `auth` |
| `configuration` | `config` |
| `environment` | `env` |
| `application` | `app` |
| `database` | `db` |
| `repository` | `repo` |
| `documentation` | `docs` |
| `implementation` | `impl` |
| `dependencies` | `deps` |
| `infrastructure` | `infra` |
| `successfully` | `OK` |
| `function` | `fn` |
| `parameter` | `param` |
| `request` / `response` | `req` / `resp` |

---

## Example

```python
from corteX.engine.cognitive.density_optimizer import DensityOptimizer, ScoredItem

optimizer = DensityOptimizer()

items = [
    ScoredItem(
        item_id="trace_1",
        content=(
            "Traceback (most recent call last):\n"
            "  File 'app.py', line 42\n"
            "  File 'db.py', line 15\n"
            "ValueError: invalid configuration parameter"
        ),
        priority=0.7,
        tokens=50,
    ),
    ScoredItem(
        item_id="pref_1",
        content="The user prefers PostgreSQL as their database",
        priority=0.5,
        tokens=12,
    ),
]

optimized = optimizer.optimize(items)
for item in optimized:
    print(f"{item.item_id}: {item.content}")
    # trace_1: [ERR] ValueError: invalid config param
    # pref_1: [PREF] db:PostgreSQL

stats = optimizer.get_stats()
print(f"Tokens saved: {stats['tokens_saved']}")
```

---

## Performance Notes

- All compression is regex-based -- no LLM calls
- Redundancy removal uses MD5 hashing for fast duplicate sentence detection
- Token estimation uses `len(text) // 4`
- Custom rules can extend the compression pipeline
- Abbreviations are case-insensitive

---

## See Also

- [Context Pyramid](./pyramid.md) -- Multi-resolution compression (complementary)
- [Cognitive Context Pipeline](./cognitive-pipeline.md) -- Uses density optimization in Phase 4
- [Context Quality Engine](./context-quality.md) -- Measures Information Density Index (IDI)
