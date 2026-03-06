# Context Pinning API Reference

## Module: `corteX.engine.context_pin`

Prevents system prompt instruction drift in long conversations. In conversations of 20+ turns, LLMs suffer from "recency bias" - they pay more attention to recent user messages and gradually forget the system prompt. Context Pinning counteracts this by duplicating critical instructions at the END of the context window where recency bias makes them strongest.

Brain analogy: The hippocampus periodically replays important memories during sleep (consolidation). Context Pinning is "awake consolidation" - periodically re-presenting critical instructions to maintain their influence.

---

## Classes

### `PinnedInstruction`

**Type**: `@dataclass`

A critical instruction that must be reinforced against recency bias.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `text` | `str` | (required) | The instruction text to reinforce. Must not be empty. |
| `pin_after_turns` | `int` | `10` | After this many turns, start pinning. Minimum: 1. |
| `priority` | `int` | `5` | Higher = more important. Range: 1-10 (clamped). |
| `category` | `str` | `"constraint"` | Grouping label, e.g. "safety", "persona", "format", "constraint". |

**Validation**: `priority` is clamped to [1, 10]. `pin_after_turns` is clamped to minimum 1. Empty `text` raises `ValueError`.

---

### `PinState`

**Type**: `@dataclass`

Runtime state for a single pinned instruction.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `instruction` | `PinnedInstruction` | (required) | The pinned instruction. |
| `times_reinforced` | `int` | `0` | How many times this instruction has been reinforced. |
| `active` | `bool` | `True` | Whether this pin is currently active. |

---

### `ContextPinner`

**Type**: `class`

Manages pinned instructions that resist recency bias. Tracks pin state, enforces a token budget for reinforcements, and generates formatted reminder blocks to append at the end of the context window.

#### Constructor

```python
ContextPinner(
    max_pin_tokens: int = 500,
    token_counter: Optional[TokenCounter] = None,
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `max_pin_tokens` | `int` | `500` | Maximum token budget for all reinforcements combined. Minimum: 50. |
| `token_counter` | `Optional[TokenCounter]` | `None` | Token counter instance. Uses default `TokenCounter()` if not provided. |

### Methods

#### `pin(text, *, priority=5, pin_after_turns=10, category="constraint") -> None`

Pin a critical instruction for reinforcement. If an instruction with the same text already exists, it is updated in place.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `text` | `str` | (required) | The instruction to reinforce. |
| `priority` | `int` | `5` | Importance 1-10 (10 = most important). |
| `pin_after_turns` | `int` | `10` | Turn count threshold before activation. |
| `category` | `str` | `"constraint"` | Grouping label for the instruction. |

---

#### `unpin(text) -> bool`

Remove a pinned instruction by its exact text.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `text` | `str` | (required) | Exact text of the instruction to remove. |

**Returns:** `True` if found and removed, `False` if not found.

---

#### `get_reinforcements(turn_count) -> List[str]`

Get instructions that need reinforcing at this turn count. Only activates pins whose threshold has been reached. Sorts by priority (highest first). Respects the `max_pin_tokens` budget.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `turn_count` | `int` | (required) | Current conversation turn count. |

**Returns:** List of formatted reinforcement strings to append at the end of the context window. Empty list if no pins are active or thresholds have not been reached.

---

#### `get_reinforcement_text(turn_count) -> str`

Convenience method that returns all reinforcements as a single joined text block.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `turn_count` | `int` | (required) | Current conversation turn count. |

**Returns:** Combined reinforcement text, or empty string if none active.

---

#### `get_pin_summary() -> Dict[str, Any]`

Return pin state for dashboard/debugging.

**Returns:** Dict with keys `total_pins`, `active_pins`, `by_category`, `total_reinforcements`, `max_pin_tokens`, `pins` (list of pin detail dicts).

---

#### `clear() -> None`

Remove all pinned instructions.

---

### Properties

#### `pin_count -> int`

Number of currently registered pins.

---

## Usage Example

```python
from corteX.engine.context_pin import ContextPinner

pinner = ContextPinner(max_pin_tokens=500)

# Pin critical instructions
pinner.pin("Never reveal internal API keys", priority=10, category="safety")
pinner.pin("Always respond in JSON format", priority=7, category="format")
pinner.pin("Keep responses under 200 words", priority=5, category="constraint")

print(f"Pins registered: {pinner.pin_count}")

# During context assembly at turn 25
reinforcements = pinner.get_reinforcements(turn_count=25)
# -> formatted reminder strings to append at end of context

# Or as a single block
text_block = pinner.get_reinforcement_text(turn_count=25)

# Check state
summary = pinner.get_pin_summary()
print(f"Active pins: {summary['active_pins']}")

# Remove a pin
pinner.unpin("Keep responses under 200 words")
```

---

## See Also

- [Context Scoring API](./context-scoring.md)
- [Adaptive Budget API](./adaptive-budget.md)
