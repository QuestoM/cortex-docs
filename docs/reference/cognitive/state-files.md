# State File Manager API Reference

## Module: `corteX.engine.cognitive.state_files`

Three-layer externalized state surviving compactions. Layer 1 (Crystallized): Immutable goal/identity. Layer 2 (Fluid): Mutable working state. Layer 3 (Insights): Append-only wisdom. Atomic JSON persistence with per-tenant/session isolation.

## Classes

### `CrystallizedState`

**Type**: `@dataclass`

Layer 1: Immutable identity state set once at initialization.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `goal` | `str` | Original goal text |
| `success_criteria` | `List[str]` | Criteria for task success |
| `constraints` | `List[str]` | Hard constraints on execution |
| `user_identity` | `Dict[str, str]` | User identity attributes |
| `project_context` | `str` | Project-level context |
| `created_at` | `float` | Unix timestamp of creation |

---

### `FluidState`

**Type**: `@dataclass`

Layer 2: Mutable current-session working state.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `current_sub_goal` | `str` | Active sub-goal being worked on |
| `progress` | `float` | Overall progress (0.0-1.0) |
| `sub_goals` | `List[Dict[str, str]]` | Sub-goal list with `goal`, `status` keys |
| `active_entities` | `Dict[str, str]` | Currently relevant entities (name: description) |
| `open_questions` | `List[str]` | Unresolved questions |
| `brain_state_digest` | `Dict[str, Any]` | Snapshot of brain state |
| `step_count` | `int` | Current step number |
| `last_updated` | `float` | Unix timestamp of last update |

---

### `InsightState`

**Type**: `@dataclass`

Layer 3: Append-only accumulated wisdom.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `decision_log` | `List[Dict[str, str]]` | Key decisions with `step`, `decision`, `rationale`, `timestamp` |
| `error_journal` | `List[Dict[str, str]]` | Errors with `step`, `error`, `resolution`, `pattern`, `status` |
| `learned_constraints` | `List[str]` | Constraints discovered during execution |
| `entity_relationships` | `List[Dict[str, str]]` | Entity relationships with `from`, `relation`, `to` |
| `pattern_observations` | `List[str]` | Observed patterns |

---

### `StateFileManager`

Manages three-layer externalized state with atomic JSON persistence.

#### Constructor

```python
StateFileManager(
    base_path: str = ".cortex_state",
    tenant_id: str = "default",
    session_id: str = ""
)
```

**Parameters**:

- `base_path` (str, default=`".cortex_state"`): Root directory for state files.
- `tenant_id` (str, default=`"default"`): Tenant identifier for isolation.
- `session_id` (str): Session identifier. Auto-generated if empty.

**File layout**: `{base_path}/{tenant_id}/{session_id}/state.json`

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `tenant_id` | `str` | Current tenant ID |
| `session_id` | `str` | Current session ID |
| `is_initialized` | `bool` | Whether `initialize()` has been called |

#### Methods

##### `initialize`

```python
async def initialize(
    self, goal: str,
    constraints: Optional[List[str]] = None,
    success_criteria: Optional[List[str]] = None,
    user_identity: Optional[Dict[str, str]] = None,
    project_context: str = ""
) -> None
```

Set the crystallized (immutable) state and persist to disk.

##### `update_fluid`

```python
async def update_fluid(
    self, progress: Optional[float] = None,
    current_sub_goal: Optional[str] = None,
    sub_goals: Optional[List[Dict[str, str]]] = None,
    entities: Optional[Dict[str, str]] = None,
    questions: Optional[List[str]] = None,
    brain_digest: Optional[Dict[str, Any]] = None,
    step_count: Optional[int] = None
) -> None
```

Update the fluid (mutable) state. Only provided fields are updated. Progress is clamped to [0.0, 1.0].

##### `record_decision`

```python
async def record_decision(self, step: int, decision: str, rationale: str) -> None
```

Append a decision to the insight layer's decision log.

##### `record_error`

```python
async def record_error(
    self, step: int, error: str,
    resolution: str = "", pattern: str = ""
) -> None
```

Record an error. Status is `"resolved"` if resolution is provided, `"open"` otherwise.

##### `resolve_error`

```python
async def resolve_error(self, step: int, resolution: str) -> bool
```

Resolve the most recent open error at the given step. Returns `True` if an error was found and resolved.

##### `add_learned_constraint`

```python
async def add_learned_constraint(self, constraint: str) -> None
```

Append a learned constraint (deduplicated).

##### `add_entity_relationship`

```python
async def add_entity_relationship(
    self, entity_a: str, relation: str, entity_b: str
) -> None
```

Record a relationship between two entities (deduplicated).

##### `add_pattern_observation`

```python
async def add_pattern_observation(self, pattern: str) -> None
```

Record a pattern observation (deduplicated).

##### `build_persistent_context`

```python
def build_persistent_context(self, max_tokens: int = 2000) -> str
```

Build a markdown context string from all three layers for injection into the LLM prompt. Truncated to `max_tokens`.

**Output structure**:

```
## Goal
Build a REST API
## Constraints
- Must use PostgreSQL
## Success Criteria
- All endpoints return JSON
## Progress: 45%
## Current Focus
Implementing user endpoints
## Key Decisions
- Step 3: Chose FastAPI (lightweight, async support)
## Unresolved Errors
- Step 7: Connection refused on port 5432
```

##### `load`

```python
async def load(self) -> bool
```

Load state from disk. Returns `True` if state was loaded, `False` if no state file exists.

##### Getter Methods

- `get_crystallized() -> CrystallizedState`
- `get_fluid() -> FluidState`
- `get_insights() -> InsightState`
- `get_decision_log() -> List[Dict[str, str]]`
- `get_error_journal() -> List[Dict[str, str]]`
- `get_open_errors() -> List[Dict[str, str]]`

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: Dict with keys `initialized`, `tenant_id`, `session_id`, `progress`, `decisions_count`, `errors_total`, `errors_open`, `learned_constraints`, `entity_relationships`, `active_entities`, `pattern_observations`.

---

## Persistence

State is persisted as atomic JSON writes:

1. Write to `state.json.tmp`
2. Delete existing `state.json`
3. Rename `.tmp` to `state.json`

This ensures no partial writes corrupt the state file.

---

## Example

```python
from corteX.engine.cognitive.state_files import StateFileManager

manager = StateFileManager(
    base_path=".cortex_state",
    tenant_id="acme",
    session_id="sess_001",
)

# Initialize with goal
await manager.initialize(
    goal="Build a user management API",
    constraints=["Use PostgreSQL", "REST only"],
    success_criteria=["CRUD endpoints", "Auth middleware"],
)

# Update progress
await manager.update_fluid(
    progress=0.3,
    current_sub_goal="Create User model",
    entities={"User": "Main entity", "PostgreSQL": "Database"},
)

# Record decisions and errors
await manager.record_decision(1, "Use FastAPI", "Async support needed")
await manager.record_error(2, "Port 5432 refused", pattern="connection")
await manager.resolve_error(2, "Started PostgreSQL service")

# Build context for LLM injection
context = manager.build_persistent_context(max_tokens=2000)

# Reload in a new session
new_manager = StateFileManager(
    base_path=".cortex_state", tenant_id="acme", session_id="sess_001"
)
loaded = await new_manager.load()
```

---

## See Also

- [Cognitive Context Pipeline](./cognitive-pipeline.md) -- Injects state into persistent zone
- [Context Versioner](./versioner.md) -- Complementary per-step versioning
- [Tenant Manager](../tenancy/manager.md) -- Tenant isolation for state paths
