# Cortical Context Engine

`corteX.engine.context`

Brain-inspired context management for long-running AI agent workflows (10,000+ steps). Implements a three-temperature memory hierarchy (hot/warm/cold) with progressive summarization (L0 verbatim through L3 digest), importance scoring, observation masking, and checkpoint-based recovery.

References: Anthropic Compaction API, OpenAI Agents SDK, Letta/MemGPT virtual context, Chroma context rot research (2025), JetBrains observation masking (NeurIPS 2025).

---

## CompressionLevel

Progressive summarization levels.

```python
class CompressionLevel(IntEnum):
    L0_VERBATIM = 0   # Full detail, no compression
    L1_CONDENSED = 1  # Observation masking (tool outputs replaced)
    L2_SUMMARY = 2    # LLM-generated summary of decision points
    L3_DIGEST = 3     # Structured digest (goals + lessons only)
```

---

## ContextConfig

SDK-configurable context management parameters. All have sane defaults.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `token_budget_ratio` | `float` | `0.80` | Use 80% of model's context window |
| `output_reservation` | `int` | `4096` | Tokens reserved for model response |
| `system_prompt_budget` | `int` | `2000` | Max tokens for system prompt |
| `hot_ratio` | `float` | `0.40` | Budget for recent turns |
| `warm_ratio` | `float` | `0.35` | Budget for compressed history |
| `cold_ratio` | `float` | `0.25` | Budget for archived retrieval |
| `summarize_every_n_steps` | `int` | `20` | Steps between compression cycles |
| `l1_age_steps` | `int` | `10` | Age before L1 compression |
| `l2_age_steps` | `int` | `50` | Age before L2 compression |
| `l3_age_steps` | `int` | `200` | Age before L3 compression |
| `tool_output_trim_chars` | `int` | `500` | Max chars for masked tool outputs |
| `checkpoint_every_n_steps` | `int` | `50` | Steps between checkpoints |
| `max_checkpoints` | `int` | `20` | Max checkpoints retained |
| `use_cheaper_model_for_summary` | `bool` | `True` | Use worker model for summaries |
| `enable_recovery` | `bool` | `True` | Enable checkpoint-based recovery |

---

## CompressionProfile

Domain-aware compression rules. Different task types need different strategies.

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Profile name (e.g., `"coding"`, `"research"`) |
| `high_importance` | `List[str]` | Patterns that should get high importance scores |
| `low_importance` | `List[str]` | Patterns that can be aggressively compressed |
| `preserve_verbatim` | `List[str]` | Patterns that should never be compressed |
| `tool_output_limits` | `Dict[str, int]` | Per-tool character limits for masking |

Built-in profiles: `CODING_PROFILE`, `RESEARCH_PROFILE`.

---

## ContextItem

A single item in the context window with importance metadata.

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_id` | `str` | Unique identifier |
| `item_type` | `ContextItemType` | Type (user message, tool result, etc.) |
| `content` | `str` | The actual text content |
| `step_number` | `int` | When this item was created |
| `token_count` | `int` | Estimated token count |
| `importance` | `float` | Computed importance score `[0, 1]` |
| `compression_level` | `CompressionLevel` | Current compression level |
| `original_token_count` | `int` | Token count before compression |
| `reference_count` | `int` | How often this item was referenced later |
| `is_decision` | `bool` | Whether this is a key decision point |
| `tool_name` | `Optional[str]` | Tool name for tool calls/results |

### Properties

- `compression_ratio -> float`: Original tokens / current tokens.

---

## TaskState

Extracted structured state from conversation history. Always included in warm memory.

| Attribute | Type | Description |
|-----------|------|-------------|
| `current_goal` | `str` | The active goal |
| `sub_goals` | `List[Dict]` | Sub-goals with status |
| `decisions_made` | `List[Dict]` | Key decisions with rationale and step |
| `active_entities` | `Dict[str, str]` | Active entities and their states |
| `constraints` | `List[str]` | Known constraints |
| `open_questions` | `List[str]` | Unresolved questions |
| `progress_percentage` | `float` | Overall progress |
| `error_patterns` | `List[Dict]` | Known errors and resolutions |
| `total_steps` | `int` | Total steps executed |
| `total_tokens_spent` | `int` | Cumulative token expenditure |

### Methods

#### `to_context_string() -> str`

Renders as structured text for inclusion in the context window.

#### `to_dict() -> Dict[str, Any]` / `from_dict(data) -> TaskState`

Serialization and deserialization.

---

## TokenCounter

Model-aware token counting with provider-specific character ratios.

### Constructor

```python
TokenCounter(model_family: str = "default")
```

Character ratios: Gemini = 4.0, GPT = 3.8, Claude = 3.9.

### Methods

#### `count(text: str) -> int`

Estimates token count for a text string.

#### `count_messages(messages: List[Dict[str, str]]) -> int`

Counts tokens across a list of chat messages, including framing overhead.

---

## ImportanceScorer

Computes composite importance scores for context items using six weighted factors.

### Constructor

```python
ImportanceScorer(
    weights: Optional[ImportanceWeights] = None,
    half_life_steps: int = 30,
)
```

### ImportanceWeights

| Factor | Default Weight | Description |
|--------|---------------|-------------|
| `recency` | `0.25` | Exponential decay by age |
| `relevance` | `0.25` | Keyword overlap with current goal |
| `causal` | `0.20` | Decision points and errors |
| `reference_frequency` | `0.10` | How often referenced later |
| `success_correlation` | `0.10` | Correlation with successful outcomes |
| `domain` | `0.10` | Domain-specific pattern matching |

### Methods

#### `score(item: ContextItem, current_step: int, current_goal: str = "", profile: Optional[CompressionProfile] = None) -> float`

Computes the composite importance score `[0, 1]`.

---

## ObservationMasker

L1 compression: replaces old tool outputs with compact placeholders. Per JetBrains NeurIPS 2025, observation masking achieves 50%+ cost reduction without trajectory elongation.

### Constructor

```python
ObservationMasker(max_chars: int = 500, profile: Optional[CompressionProfile] = None)
```

### Methods

#### `mask(item: ContextItem) -> ContextItem`

Applies observation masking to a tool result. Respects `preserve_verbatim` patterns from the profile and per-tool character limits.

#### `mask_batch(items: List[ContextItem], current_step: int) -> List[ContextItem]`

Applies masking to items older than 10 steps.

---

## ContextWindowPacker

Assembles the optimal context window from three temperature tiers.

### Constructor

```python
ContextWindowPacker(
    config: Optional[ContextConfig] = None,
    token_counter: Optional[TokenCounter] = None,
)
```

### Methods

#### `pack(system_prompt: str, task_state: Optional[TaskState], hot_items: List[ContextItem], warm_items: List[ContextItem], cold_items: List[ContextItem], model_context_window: int = 200000) -> List[ContextItem]`

Packs context items into the optimal window. Order: task state, warm context (by step), cold context (by importance), hot context (chronological). Recent turns go last for recency bias.

#### `get_packing_stats(packed: List[ContextItem], model_context_window: int = 200000) -> Dict[str, Any]`

Returns: `total_tokens`, `total_items`, `budget_utilization`, `by_type`, `by_compression_level`, `avg_importance`.

---

## ContextCheckpointer

Periodic full context snapshots for fault-tolerant recovery.

### Methods

#### `create_checkpoint(step_number, task_state, hot_item_ids, warm_item_ids, total_tokens_spent) -> ContextCheckpoint`

Creates a new checkpoint. Enforces max checkpoint limit by removing oldest.

#### `get_latest() -> Optional[ContextCheckpoint]`

Returns the most recent checkpoint.

#### `get_checkpoint_before_step(step: int) -> Optional[ContextCheckpoint]`

Returns the most recent checkpoint before a given step.

---

## CorticalContextEngine

The main context management engine orchestrating all operations.

### Constructor

```python
CorticalContextEngine(
    config: Optional[ContextConfig] = None,
    profile: Optional[CompressionProfile] = None,
    model_family: str = "default",
)
```

### Methods

#### `set_goal(goal: str) -> None`

Sets the current goal for relevance scoring.

#### `set_system_prompt(prompt: str) -> None`

Sets the system prompt (always included in context).

#### `add_user_message(step: int, content: str) -> ContextItem`

Adds a user message to hot memory (importance: `0.7`).

#### `add_assistant_message(step: int, content: str) -> ContextItem`

Adds an assistant message to hot memory (importance: `0.5`).

#### `add_tool_result(step: int, tool_name: str, content: str, is_error: bool = False) -> ContextItem`

Adds a tool result to hot memory (importance: `0.8` for errors, `0.4` otherwise).

#### `add_tool_call(step: int, tool_name: str, arguments: str) -> ContextItem`

Adds a tool call record to hot memory (importance: `0.3`).

#### `record_decision(step: int, decision: str, rationale: str) -> None`

Records a key decision point. Marks corresponding items with elevated importance (`0.9`).

#### `compress() -> int`

Runs progressive compression: L1 masking on old items, moves aged items from hot to warm to cold. Returns number of items compressed.

#### `get_context_window(model_context_window: int = 200000) -> List[ContextItem]`

Returns the optimally packed context window for the next LLM call. Auto-triggers compression and checkpointing as needed.

#### `get_stats() -> Dict[str, Any]`

Returns: `current_step`, `hot_items`, `warm_items`, `cold_items`, `total_items`, `hot_tokens`, `warm_tokens`, `total_tokens_spent`, `compression_count`, `checkpoints`, `task_progress`.

#### `get_token_budget_status(model_context_window: int = 200000) -> Dict[str, Any]`

Returns: `total_budget`, `used`, `available`, `utilization`, token breakdowns.

### Example

```python
from corteX.engine.context import CorticalContextEngine, ContextConfig, CODING_PROFILE

cce = CorticalContextEngine(
    config=ContextConfig(token_budget_ratio=0.80),
    profile=CODING_PROFILE,
    model_family="gemini",
)
cce.set_goal("Build a REST API with authentication")

# On each step
cce.add_user_message(step=1, content="Implement the login endpoint")
cce.add_tool_call(step=1, tool_name="file_write", arguments="auth.py")
cce.add_tool_result(step=1, tool_name="file_write", content="File written: auth.py")
cce.add_assistant_message(step=1, content="Created login endpoint in auth.py")
cce.record_decision(step=1, decision="Use JWT tokens", rationale="Stateless auth")

# Get packed context for LLM call
packed = cce.get_context_window(model_context_window=200000)

# Check stats
stats = cce.get_stats()
print(f"Items: {stats['total_items']}, Tokens: {stats['hot_tokens']}")
```
