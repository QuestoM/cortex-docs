# Memory Crystallizer API Reference

## Module: `corteX.engine.cognitive.crystallizer`

Extracts reusable cognitive patterns from successful task executions. A Crystal is a generalized pattern containing a goal template, decision chain, tool sequence, error patterns, and effectiveness score. Crystals are matched to new goals by keyword similarity and injected as few-shot examples.

## Classes

### `Crystal`

**Type**: `@dataclass`

A reusable cognitive pattern extracted from a successful execution.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `crystal_id` | `str` | Unique identifier (`crystal_{session_id}_{timestamp}`) |
| `goal_template` | `str` | Generalized goal with placeholders (`{file_path}`, `{N}`, `{url}`) |
| `decision_chain` | `List[str]` | Sequence of decisions made (max 20) |
| `tool_sequence` | `List[str]` | Ordered deduplicated tools used (max 15) |
| `error_patterns` | `List[Dict[str, str]]` | Error/resolution pairs encountered (max 10) |
| `success_score` | `float` | Quality score of the original execution (0.0-1.0) |
| `reuse_count` | `int` | Number of times this crystal has been reused |
| `effectiveness` | `float` | Running average success rate across reuses |
| `entity_types` | `List[str]` | Detected entity types (e.g., `python_file`, `api_endpoint`) |
| `created_from` | `str` | Source session identifier |
| `created_at` | `float` | Unix timestamp of creation |

---

### `MemoryCrystallizer`

Distills reusable patterns from successful task completions. After a task succeeds, extracts a Crystal with a generalized goal template, decision chain, tool sequence, and error patterns. Crystals are matched to new goals by keyword similarity.

#### Constructor

```python
MemoryCrystallizer(
    max_crystals: int = 200,
    min_success_score: float = 0.7
)
```

**Parameters**:

- `max_crystals` (int, default=200): Maximum stored crystals. Least effective are evicted.
- `min_success_score` (float, default=0.7): Minimum success score required to create a crystal.

#### Methods

##### `crystallize`

```python
def crystallize(
    self, goal: str,
    execution_trace: List[Dict[str, Any]],
    success_score: float,
    session_id: str = ""
) -> Optional[Crystal]
```

Extract a crystal from a successful task execution.

**Parameters**:

- `goal` (`str`): The original goal text.
- `execution_trace` (`List[Dict[str, Any]]`): Step dicts with `decision`, `tool`, `error`, `resolution` keys.
- `success_score` (`float`): Quality of the outcome (0.0-1.0).
- `session_id` (`str`): Source session identifier.

**Returns**: `Optional[Crystal]` -- The extracted crystal, or `None` if score is below threshold or the pattern is not novel enough.

##### `query`

```python
def query(self, goal_text: str, top_k: int = 3) -> List[Crystal]
```

Find matching crystals by keyword similarity with the goal.

**Parameters**:

- `goal_text` (`str`): Goal to match against stored crystals.
- `top_k` (`int`, default=3): Maximum number of results.

**Returns**: `List[Crystal]` -- Top matching crystals, sorted by similarity (boosted by effectiveness).

##### `apply`

```python
def apply(self, crystal: Crystal, context: str = "") -> str
```

Generate a few-shot prompt from a crystal for context injection.

**Parameters**:

- `crystal` (`Crystal`): The crystal to format.
- `context` (`str`): Optional current context for reference.

**Returns**: `str` -- Formatted markdown string for injection into the persistent zone.

**Output format**:

```
## Similar Past Success (85% effective, used 3x)
Goal pattern: Deploy {file_path} to {url}
Approach: read config -> validate -> deploy -> verify
Tools used: file_read, shell, browser
Known pitfalls:
  - Timeout on large files: increase timeout to 60s
```

##### `record_reuse`

```python
def record_reuse(self, crystal_id: str, success: bool) -> None
```

Update effectiveness after a crystal is reused. Uses running average.

##### `get_crystal`

```python
def get_crystal(self, crystal_id: str) -> Optional[Crystal]
```

Retrieve a crystal by ID.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: `Dict[str, Any]` with keys `total_crystals`, `effective_crystals`, `total_reuses`, `avg_effectiveness`.

---

## Goal Generalization

The crystallizer generalizes goals by replacing specific values with type placeholders:

| Original | Generalized |
|----------|-------------|
| `Deploy src/app.py` | `Deploy {file_path}` |
| `Process 150 records` | `Process {N} records` |
| `Call /api/users/list` | `Call {api_endpoint}` |
| `Visit https://example.com` | `Visit {url}` |
| `Set "production"` | `Set "{value}"` |

---

## Novelty Detection

A new crystal is rejected if an existing crystal has:
- Tool sequence Jaccard similarity > 0.8, AND
- Goal template word overlap > 0.7

This prevents redundant crystals from accumulating.

---

## Example

```python
from corteX.engine.cognitive.crystallizer import MemoryCrystallizer

crystallizer = MemoryCrystallizer(max_crystals=200)

# Extract a crystal from a successful task
crystal = crystallizer.crystallize(
    goal="Deploy app.py to https://prod.example.com",
    execution_trace=[
        {"decision": "Read config", "tool": "file_read"},
        {"decision": "Validate schema", "tool": "code_interpreter"},
        {"decision": "Deploy", "tool": "shell",
         "error": "Timeout", "resolution": "Increased timeout to 60s"},
    ],
    success_score=0.9,
    session_id="sess_001",
)

# Query for matching crystals
matches = crystallizer.query("Deploy server.py to staging")
for match in matches:
    prompt = crystallizer.apply(match)
    print(prompt)

# Record reuse outcome
if crystal:
    crystallizer.record_reuse(crystal.crystal_id, success=True)
```

---

## See Also

- [Active Forgetting Engine](./active-forgetting.md) -- Complementary memory management
- [State File Manager](./state-files.md) -- Persistent state across sessions
- [Cognitive Context Pipeline](./cognitive-pipeline.md) -- Crystal injection into context
