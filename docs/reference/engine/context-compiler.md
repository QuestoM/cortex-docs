# Context Compiler API Reference

## Module: `corteX.engine.context_compiler`

4-Zone Context Window Assembly for KV-cache reuse. Zones: SYSTEM (12%), PERSISTENT (8%), WORKING (40%), RECENT (40%). Compaction triggers at 80/90/95% utilization. Goal text appears at both extremes of the context window.

---

## Classes

### `ContextItem`

**Type**: `@dataclass`

Single context window item with importance metadata.

| Attribute | Type | Description |
|-----------|------|-------------|
| `role` | `str` | Message role (`"system"`, `"user"`, `"assistant"`, `"tool"`) |
| `content` | `str` | Message content |
| `token_estimate` | `int` | Estimated token count |
| `importance` | `float` | Importance score [0.0, 1.0] |
| `timestamp` | `float` | Creation timestamp |
| `zone` | `str` | Which zone this item belongs to |
| `metadata` | `Dict[str, Any]` | Additional metadata |

### `ContextZone`

**Type**: `@dataclass`

Named zone in the 4-zone architecture. Each zone has a budget ratio (fraction of the total context window) and tracks its own items and token count.

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Zone name (`"system"`, `"persistent"`, `"working"`, `"recent"`) |
| `budget_ratio` | `float` | Fraction of total context window allocated to this zone |
| `items` | `List[ContextItem]` | Items stored in this zone |
| `token_count` | `int` | Current total token count for this zone |

#### Methods

##### `recalculate_tokens`

```python
def recalculate_tokens(self) -> int
```

Recalculate the zone's token count by summing all item token estimates. Returns the updated token count.

### `CompiledContext`

**Type**: `@dataclass`

Assembled context window ready for the LLM API.

| Attribute | Type | Description |
|-----------|------|-------------|
| `messages` | `List[Dict[str, str]]` | Ordered messages for the LLM |
| `total_tokens` | `int` | Total estimated tokens |
| `zone_usage` | `Dict[str, int]` | Token usage per zone |
| `compaction_needed` | `bool` | Whether compaction is needed |
| `compaction_level` | `str` | `"none"`, `"light"`, `"full"`, or `"emergency"` |

---

### `ContextCompiler`

Assembles optimal context windows for LLM calls using the 4-zone architecture.

#### Constructor

```python
ContextCompiler(
    max_context_tokens: int = 128_000,
    zone_budgets: Optional[Dict[str, float]] = None,
)
```

**Parameters**:

- `max_context_tokens` (int): Maximum context window size. Default: 128,000
- `zone_budgets` (Optional[Dict]): Custom zone budget ratios. Default: `{"system": 0.12, "persistent": 0.08, "working": 0.40, "recent": 0.40}`

#### Methods

##### `set_system_prompt`

```python
def set_system_prompt(self, prompt: str) -> None
```

Set agent identity / base system prompt. Stable content cached in SYSTEM zone.

##### `set_user_preferences`

```python
def set_user_preferences(self, prefs: Dict[str, Any]) -> None
```

Set user preferences that are included in the SYSTEM zone. Preferences are serialized as JSON and appended under a `## User Preferences` heading in the system content.

**Parameters**:

- `prefs` (`Dict[str, Any]`): Arbitrary user preference key-value pairs.

##### `set_policies`

```python
def set_policies(self, policies: List[str]) -> None
```

Set active policies that are included in the SYSTEM zone. Policies are rendered as a bullet list under a `## Policies` heading in the system content.

**Parameters**:

- `policies` (`List[str]`): List of policy descriptions.

##### `set_tool_definitions`

```python
def set_tool_definitions(self, tools: List[Dict[str, Any]]) -> None
```

Set available tool definitions that are included in the SYSTEM zone. Tool names are extracted from each dict's `"name"` key and listed under a `## Available Tools` heading.

**Parameters**:

- `tools` (`List[Dict[str, Any]]`): List of tool definition dicts (each must have a `"name"` key).

##### `set_goal`

```python
def set_goal(self, goal: str) -> None
```

Set current goal. Appears at the start (PERSISTENT zone) AND end (RECENT zone) of context.

##### `set_brain_digest`

```python
def set_brain_digest(self, digest: Dict[str, Any]) -> None
```

Set the brain state digest that is included in the PERSISTENT zone. The digest is serialized as JSON under a `## Brain State` heading alongside the goal.

**Parameters**:

- `digest` (`Dict[str, Any]`): Brain state summary (e.g., current weights, active concepts).

##### `set_task_state`

```python
def set_task_state(self, state: str) -> None
```

Set current task state / next-step description. Rendered under a `## Current Task` heading at the start of the RECENT zone.

**Parameters**:

- `state` (`str`): Free-text description of the current task state or next step.

##### `append_message`

```python
def append_message(self, role: str, content: str, importance: float = 0.5) -> None
```

Append a conversation message to working memory.

##### `append_tool_result`

```python
def append_tool_result(self, tool_name: str, result: str, success: bool = True) -> None
```

Append a tool result. Failed results get higher importance (0.8) than successful ones (0.4).

##### `add_recent_message`

```python
def add_recent_message(self, role: str, content: str, importance: float = 0.7) -> None
```

Add a message to the recent (hot) zone. Recent zone items are the last messages in the context window, closest to the LLM's "attention focus". Default importance is 0.7 (higher than working memory's 0.5).

**Parameters**:

- `role` (`str`): Message role (`"user"`, `"assistant"`, `"system"`, `"tool"`).
- `content` (`str`): Message content.
- `importance` (`float`): Importance score [0.0, 1.0]. Clamped to valid range. Default: 0.7.

##### `write_scratchpad`

```python
def write_scratchpad(self, note: str) -> None
```

Agent-writable scratchpad note stored in working memory.

##### `compile`

```python
def compile(self) -> CompiledContext
```

Assemble the optimal context window from all four zones. Returns `CompiledContext` with ordered messages and utilization metrics.

##### `get_compaction_level`

```python
def get_compaction_level(self) -> str
```

Check current token utilization and return the compaction level without performing a full compile. Useful for monitoring context pressure and deciding when to trigger compaction.

**Returns**: One of `"none"`, `"light"`, `"full"`, or `"emergency"`.

| Return Value | Utilization |
|--------------|-------------|
| `"none"` | < 80% |
| `"light"` | >= 80% |
| `"full"` | >= 90% |
| `"emergency"` | >= 95% |

##### `compact_working_memory`

```python
def compact_working_memory(self, summary: str) -> None
```

Replace all working-memory items with a single summary. Used for light/full compaction.

##### `emergency_compact`

```python
def emergency_compact(self, summary: str) -> None
```

Keep only goal + brief summary + last 2 recent items. Used when utilization exceeds 95%.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Return token usage and item counts per zone. Provides a comprehensive snapshot of context window utilization without performing a full compile.

**Returns**: Dict with the following keys:

| Key | Type | Description |
|-----|------|-------------|
| `max_context_tokens` | `int` | Configured maximum context window size |
| `system_tokens` | `int` | Token count in the SYSTEM zone |
| `persistent_tokens` | `int` | Token count in the PERSISTENT zone |
| `working_tokens` | `int` | Token count in the WORKING zone (includes scratchpad) |
| `working_items` | `int` | Number of items in working memory |
| `scratchpad_notes` | `int` | Number of scratchpad notes |
| `recent_tokens` | `int` | Token count in the RECENT zone (includes task state and goal restatement) |
| `recent_items` | `int` | Number of items in recent memory |
| `total_tokens_estimate` | `int` | Total estimated tokens across all zones |
| `utilization` | `float` | Ratio of total tokens to max context tokens [0.0, 1.0] |
| `compaction_level` | `str` | Current compaction level (`"none"`, `"light"`, `"full"`, `"emergency"`) |
| `goal_set` | `bool` | Whether a goal has been set |

---

## Zone Architecture

| Zone | Budget | Content | Stability |
|------|--------|---------|-----------|
| SYSTEM | 12% | System prompt, policies, tools, user prefs | Stable (cached) |
| PERSISTENT | 8% | Goal, brain state digest | Semi-stable |
| WORKING | 40% | Conversation history, tool results, scratchpad | Dynamic |
| RECENT | 40% | Recent messages, task state, goal restatement | Hot |

## Compaction Levels

| Level | Trigger | Action |
|-------|---------|--------|
| `none` | < 80% utilization | No action |
| `light` | >= 80% | Summarize working memory |
| `full` | >= 90% | Full compaction of working memory |
| `emergency` | >= 95% | Keep only goal + summary + last 2 items |

---

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `DEFAULT_ZONE_BUDGETS` | `{"system": 0.12, "persistent": 0.08, "working": 0.40, "recent": 0.40}` | Default budget ratios |
| `COMPACT_LIGHT` | `0.80` | Light compaction threshold |
| `COMPACT_FULL` | `0.90` | Full compaction threshold |
| `COMPACT_EMERGENCY` | `0.95` | Emergency compaction threshold |

---

## Usage Example

```python
from corteX.engine.context_compiler import ContextCompiler

compiler = ContextCompiler(max_context_tokens=128_000)
compiler.set_system_prompt("You are a helpful coding assistant.")
compiler.set_goal("Build a REST API for user management")

# Configure zones
compiler.set_user_preferences({"language": "en", "verbosity": "concise"})
compiler.set_policies(["Never expose PII", "Always validate input"])
compiler.set_tool_definitions([{"name": "code_writer"}, {"name": "bash"}])
compiler.set_brain_digest({"active_concept": "rest_api", "confidence": 0.85})
compiler.set_task_state("Creating the User model")

# Add conversation messages
compiler.append_message("user", "Create the user model")
compiler.append_message("assistant", "Here is the User model...")
compiler.append_tool_result("code_writer", "file created: models/user.py", success=True)

# Add recent hot messages
compiler.add_recent_message("user", "Now add validation", importance=0.9)

# Check utilization before compile
stats = compiler.get_stats()
print(f"Utilization: {stats['utilization']:.1%}")
print(f"Compaction level: {compiler.get_compaction_level()}")

# Compile the context window
ctx = compiler.compile()
print(f"Total tokens: {ctx.total_tokens}")
print(f"Compaction needed: {ctx.compaction_needed}")
# ctx.messages is ready for the LLM API
```

---

## See Also

- [Context Engine Concept](../../concepts/context/context-engine.md)
- [Agent Loop API](./agent-loop.md)
