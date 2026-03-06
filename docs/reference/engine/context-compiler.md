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

### `ContextZone`

**Type**: `@dataclass`

Named zone in the 4-zone architecture (internal).

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Zone name (system, persistent, working, recent) |
| `budget_ratio` | `float` | Budget ratio for this zone |
| `items` | `List[ContextItem]` | Items in this zone |
| `token_count` | `int` | Current token count |

**Methods**: `recalculate_tokens() -> int`

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
def set_user_preferences(prefs: Dict[str, Any]) -> None
```

Set user preferences for inclusion in the SYSTEM zone.

##### `set_policies`

```python
def set_policies(policies: List[str]) -> None
```

Set active policies for inclusion in the SYSTEM zone.

##### `set_tool_definitions`

```python
def set_tool_definitions(tools: List[Dict[str, Any]]) -> None
```

Set tool definitions for inclusion in the SYSTEM zone.

##### `set_goal`

```python
def set_goal(self, goal: str) -> None
```

Set current goal. Appears at the start (PERSISTENT zone) AND end (RECENT zone) of context.

##### `set_brain_digest`

```python
def set_brain_digest(digest: Dict[str, Any]) -> None
```

Set brain state digest for inclusion in the PERSISTENT zone.

##### `set_task_state`

```python
def set_task_state(state: str) -> None
```

Set current task state / next-step description for the RECENT zone.

##### `add_recent_message`

```python
def add_recent_message(role: str, content: str, importance: float = 0.7) -> None
```

Add a message directly to the RECENT (hot) zone.

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

##### `get_compaction_level`

```python
def get_compaction_level() -> str
```

Check current utilization and return the compaction level: `"none"`, `"light"`, `"full"`, or `"emergency"`.

##### `get_stats`

```python
def get_stats() -> Dict[str, Any]
```

Return token usage and item counts per zone. Includes total tokens, per-zone breakdowns, and utilization percentage.

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

## Usage Example

```python
from corteX.engine.context_compiler import ContextCompiler

compiler = ContextCompiler(max_context_tokens=128_000)
compiler.set_system_prompt("You are a helpful coding assistant.")
compiler.set_goal("Build a REST API for user management")

compiler.append_message("user", "Create the user model")
compiler.append_message("assistant", "Here is the User model...")
compiler.append_tool_result("code_writer", "file created: models/user.py", success=True)

ctx = compiler.compile()
print(f"Total tokens: {ctx.total_tokens}")
print(f"Compaction needed: {ctx.compaction_needed}")
# ctx.messages is ready for the LLM API
```

---

## See Also

- [Context Engine Concept](../../concepts/context/context-engine.md)
- [Agent Loop API](./agent-loop.md)
