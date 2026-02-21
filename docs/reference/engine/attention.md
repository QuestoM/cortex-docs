# Attention

`corteX.engine.attention` -- Attentional filter engine with change detection, priority classification, context compression, and spotlight gating.

---

## Overview

The brain does NOT process everything at the same depth. It allocates scarce resources based
on NOVELTY, RELEVANCE, and THREAT. Most sensory input is processed subconsciously -- only
deviations break through to conscious awareness. This module implements the same principle
for the corteX SDK pipeline.

Architecture:

- **AttentionalPriority** -- Enum: CRITICAL, FOREGROUND, BACKGROUND, SUBCONSCIOUS, SUPPRESSED
- **ChangeEvent** -- A detected change between consecutive states
- **StateFingerprint** -- Compact state representation at a point in time
- **ProcessingBudget** -- Recommended processing parameters per priority level
- **ChangeDetector** -- Tracks state and detects what has changed (Mismatch Negativity analog)
- **AttentionalFilter** -- Classifies messages by processing priority (thalamic gating)
- **ContextDeltaCompressor** -- Compresses stable context, highlights changes
- **AttentionalGate** -- Spotlight-based information flow control (Posner model)
- **AttentionSystem** -- Unified facade wiring all components

---

## Enum: AttentionalPriority

| Value | Description | Model Tier | Max Tokens |
|-------|-------------|------------|------------|
| `CRITICAL` | Errors, safety violations, goal drift. Cannot be suppressed. | orchestrator | 8192 |
| `FOREGROUND` | New topics, changed patterns. Spotlight of attention. | orchestrator | 4096 |
| `BACKGROUND` | Continuation of existing topic with minor variation. | worker | 2048 |
| `SUBCONSCIOUS` | Routine pattern matching. No tools, no memory retrieval. | worker | 1024 |
| `SUPPRESSED` | Fully habituated signal. Skip or serve from cache. | worker | 256 |

---

## Dataclass: ChangeEvent

A detected change between consecutive states.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `change_type` | `str` | *(required)* | One of: `topic_shift`, `behavior_shift`, `tool_shift`, `error_spike`, `quality_drift`. |
| `magnitude` | `float` | *(required)* | Strength of the change [0.0, 1.0]. |
| `old_value` / `new_value` | `Any` | *(required)* | Previous and current state of the changed feature. |
| `description` | `str` | *(required)* | Human-readable description. |
| `timestamp` | `float` | `time.time()` | When the change was detected. Auto-set on creation. |

**Methods**: `to_dict() -> Dict[str, Any]`, `from_dict(data) -> ChangeEvent` (classmethod).

---

## Dataclass: StateFingerprint

Compact representation of conversational state at a point in time.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `topic_hash` | `str` | `""` | Bag-of-words fingerprint of the topic. |
| `message_length` | `float` | `0.0` | Normalized message length (chars/1000). |
| `avg_word_length` | `float` | `0.0` | Proxy for vocabulary complexity. |
| `question_ratio` | `float` | `0.0` | Fraction of sentences that are questions. |
| `tools_hash` | `str` | `""` | Fingerprint of tools used. |
| `error_rate` | `float` | `0.0` | Running error rate. |
| `quality_score` | `float` | `0.0` | Running quality score. |
| `turn_number` | `int` | `0` | Sequential turn counter. |
| `timestamp` | `float` | `time.time()` | When the fingerprint was created. Auto-set on creation. |

**Methods**: `to_dict() -> Dict[str, Any]`, `from_dict(data) -> StateFingerprint` (classmethod).

---

## Dataclass: ProcessingBudget

Recommended processing parameters for a given attentional priority. The brain allocates metabolic resources differentially across cortical regions depending on task demands; this dataclass encodes the analogous computational budget.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_tokens` | `int` | `4096` | Maximum tokens to allocate for the LLM response. |
| `model_tier` | `str` | `"orchestrator"` | Which model tier to use: `orchestrator` (full) or `worker` (light). |
| `use_tools` | `bool` | `True` | Whether tool calling is allowed. |
| `max_tool_calls` | `int` | `10` | Upper bound on tool invocations. |
| `use_memory_retrieval` | `bool` | `True` | Whether to query episodic/semantic memory. |
| `use_prediction_engine` | `bool` | `True` | Whether to run predict-compare-surprise loop. |
| `allow_cache` | `bool` | `False` | Whether to serve from response cache if available. |
| `context_depth` | `str` | `"full"` | How much context to include: `full`, `summary`, `minimal`. |

**Methods**: `to_dict() -> Dict[str, Any]`

### Budget Presets by Priority

| Priority | max_tokens | model_tier | use_tools | max_tool_calls | memory | prediction | cache | context_depth |
|----------|-----------|------------|-----------|---------------|--------|------------|-------|---------------|
| CRITICAL | 8192 | orchestrator | Yes | 20 | Yes | Yes | No | full |
| FOREGROUND | 4096 | orchestrator | Yes | 10 | Yes | Yes | No | full |
| BACKGROUND | 2048 | worker | Yes | 5 | Yes | No | Yes | summary |
| SUBCONSCIOUS | 1024 | worker | No | 0 | No | No | Yes | minimal |
| SUPPRESSED | 256 | worker | No | 0 | No | No | Yes | minimal |

---

## Class: ChangeDetector

Tracks conversational state and detects changes across multiple dimensions. Like Mismatch
Negativity (MMN) in auditory cortex -- builds an implicit model of "normal" and fires when
reality deviates.

### Constructor

```python
ChangeDetector(
    history_size: int = 20,
    topic_threshold: float = 0.40,
    behavior_threshold: float = 0.30,
    tool_threshold: float = 0.50,
    error_threshold: float = 0.60,
    quality_drift_threshold: float = 0.20,
    quality_drift_window: int = 6,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record_state` | `(state: Dict[str, Any]) -> None` | Record current state. Keys: `message`, `tools_used`, `error_count`, `quality`. |
| `detect_changes` | `() -> List[ChangeEvent]` | Compare two most recent states. Returns detected changes. |
| `get_delta_magnitude` | `() -> float` | Composite change magnitude [0.0, 1.0]. |
| `get_stable_features` | `() -> Dict[str, Any]` | Features unchanged over recent history. |
| `get_changing_features` | `() -> Dict[str, Any]` | Features that have changed recently. |
| `get_stats` | `() -> Dict[str, Any]` | Diagnostic statistics: states recorded, changes detected, change type counts. |
| `to_dict` / `from_dict` | | Serialization. |

---

## Class: AttentionalFilter

Main entry point for priority classification. Uses `ChangeDetector` internally.

### Constructor

```python
AttentionalFilter(
    foreground_threshold: float = 0.35,
    subconscious_threshold: float = 0.10,
    subconscious_streak: int = 4,
    critical_keywords: Optional[Set[str]] = None,
    goal_drift_threshold: float = 0.5,
    adaptation_filter: Optional[Any] = None,
)
```

**Parameters**:

- `foreground_threshold` (float): Delta above this triggers FOREGROUND. Default: 0.35
- `subconscious_threshold` (float): Delta below this for N turns triggers SUBCONSCIOUS. Default: 0.10
- `subconscious_streak` (int): N consecutive low-delta turns for SUBCONSCIOUS. Default: 4
- `critical_keywords` (Optional[Set[str]]): Extra keywords that force CRITICAL priority. Merged with built-in keywords.
- `goal_drift_threshold` (float): Goal drift above this triggers CRITICAL. Default: 0.5
- `adaptation_filter` (Optional[Any]): Optional `AdaptationFilter` instance for habituation detection (SUPPRESSED classification). When the majority of tracked signals are habituated, messages are classified as SUPPRESSED.

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `classify` | `(message: str, context: Optional[Dict] = None) -> AttentionalPriority` | Classify a message. Call before LLM inference to determine processing budget. |
| `get_processing_budget` | `(priority: AttentionalPriority) -> Dict[str, Any]` | Get recommended budget: `max_tokens`, `model_tier`, `use_tools`, `use_memory_retrieval`, etc. |
| `should_use_cache` | `(message: str) -> bool` | Whether a cached response is acceptable. |
| `record_turn` | `(message, response, tools_used=None, quality=0.5) -> None` | Record a completed turn for future classification. |
| `get_attention_summary` | `() -> Dict[str, Any]` | Snapshot of current attentional state. |
| `get_stats` | `() -> Dict[str, Any]` | Diagnostic statistics: priority distribution, reason distribution, cache stats. |
| `to_dict` / `from_dict` | | Serialization. |

---

## Class: ContextDeltaCompressor

Optimizes context by highlighting changes and compressing stable parts.

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `compress` | `(current_context: Dict, previous_context: Optional[Dict]) -> Dict` | Delta-compress context. Stable keys get `[STABLE]` prefix, changed keys get `[CHANGED]` with delta info. |
| `highlight_changes` | `(context: Dict, changes: List[ChangeEvent]) -> str` | Human-readable change summary for system prompt injection. |
| `get_stats` | `() -> Dict[str, Any]` | Compression statistics: count, original/compressed chars, ratio. |
| `to_dict` / `from_dict` | | Serialization. |

---

## Class: AttentionalGate

Spotlight-based information flow control (Posner attention model). Fixed-capacity spotlight
(default 5 items). Spotlight = 1.0 multiplier, penumbra = 0.3, periphery = 0.1.

### Constructor

```python
AttentionalGate(
    capacity: int = 5,
    penumbra_multiplier: float = 0.3,
    periphery_multiplier: float = 0.1,
    decay_rate: float = 0.05,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `update_spotlight` | `(items: List[Tuple[str, float]]) -> None` | Shift spotlight. Highest relevance items enter; displaced items move to penumbra. |
| `is_in_spotlight` | `(item_id: str) -> bool` | Check if item is in spotlight. |
| `get_processing_multiplier` | `(item_id: str) -> float` | 1.0 (spotlight), 0.3 (penumbra), or 0.1 (periphery). |
| `get_spotlight_items` / `get_penumbra_items` | `() -> List[str]` | Current items in each zone. |
| `get_relevance` | `(item_id: str) -> float` | Current relevance score for an item. |
| `get_all_relevances` | `() -> Dict[str, float]` | All tracked items and their relevance scores. |
| `clear` | `() -> None` | Clear spotlight, penumbra, and all relevance scores. |
| `get_stats` | `() -> Dict[str, Any]` | Diagnostic statistics. |
| `to_dict` / `from_dict` | | Serialization. |

---

## Class: AttentionSystem

Unified facade wiring all attentional components into a single `process_turn` call.

### Constructor

```python
AttentionSystem(
    foreground_threshold: float = 0.35,
    subconscious_threshold: float = 0.10,
    subconscious_streak: int = 4,
    spotlight_capacity: int = 5,
    adaptation_filter: Optional[Any] = None,
)
```

**Parameters**:

- `foreground_threshold` (float): Passed to `AttentionalFilter`. Default: 0.35
- `subconscious_threshold` (float): Passed to `AttentionalFilter`. Default: 0.10
- `subconscious_streak` (int): Passed to `AttentionalFilter`. Default: 4
- `spotlight_capacity` (int): Passed to `AttentionalGate`. Default: 5
- `adaptation_filter` (Optional[Any]): Optional `AdaptationFilter` instance, passed to `AttentionalFilter` for habituation detection.

### Attributes

| Name | Type | Description |
|------|------|-------------|
| `filter` | `AttentionalFilter` | The attentional filter instance. |
| `compressor` | `ContextDeltaCompressor` | The context compressor instance. |
| `gate` | `AttentionalGate` | The spotlight gate instance. |

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `process_turn` | `(message, context, spotlight_items=None) -> Dict[str, Any]` | Full pipeline: classify, budget, compress, detect changes, update spotlight. Returns priority, budget, compressed context, change highlights. |
| `record_turn` | `(message, response, tools_used=None, quality=0.5) -> None` | Record completed turn. |
| `get_stats` | `() -> Dict[str, Any]` | Aggregate statistics from filter, compressor, and gate. |
| `to_dict` / `from_dict` | | Serialization. |

---

## Example

```python
from corteX.engine.attention import AttentionSystem

attention = AttentionSystem()

# Process a turn
result = attention.process_turn(
    message="There's a security vulnerability in the auth module!",
    context={"error_count": 0, "quality": 0.7},
)
print(result["priority"])  # "critical" (keyword "security" detected)
print(result["budget"])    # max_tokens=8192, model_tier="orchestrator"

# Process a routine follow-up
result2 = attention.process_turn(
    message="Thanks, that looks good",
    context={"error_count": 0, "quality": 0.8},
)
print(result2["priority"])  # "background" or "subconscious"
```
