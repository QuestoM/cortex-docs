# Interaction Manager API Reference

## Module: `corteX.engine.interaction`

Agent-to-user interaction with five autonomy levels, smart timeouts, and auto-decision for minimal user interruption. Based on Sheridan & Verplank's autonomy taxonomy.

Brain analogy: Prefrontal cortex (autonomy), amygdala (risk assessment), hippocampus (auto-decide from memory), default mode (safe timeout defaults). Never calls LLM directly -- builds prompts and parses responses.

---

## Classes

### `AutonomyLevel`

**Type**: `str, Enum`

Five autonomy levels from Sheridan & Verplank's taxonomy.

| Value | Level | Description |
|-------|-------|-------------|
| `L1` | OPERATOR | User controls everything |
| `L2` | COLLABORATOR | Agent suggests, user confirms |
| `L3` | CONSULTANT | Agent acts, asks on key decisions |
| `L4` | APPROVER | Agent pre-selects, user rubber-stamps |
| `L5` | AUTONOMOUS | Fully autonomous, user monitors |

### `InteractionType`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `INFORMATION` | Informational notification |
| `CONFIRMATION` | Yes/no confirmation |
| `CHOICE` | Multiple-choice selection |
| `PERMISSION` | Permission request |

### `InteractionRequest`

**Type**: `@dataclass`

An interaction request from the agent to the user.

| Attribute | Type | Description |
|-----------|------|-------------|
| `interaction_type` | `InteractionType` | Type of interaction |
| `question` | `str` | Question text |
| `options` | `List[str]` | Available choices |
| `default_choice` | `Optional[str]` | Default if timeout |
| `timeout_seconds` | `float` | Time before auto-decide (default: 30.0) |
| `can_auto_decide` | `bool` | Whether auto-decision is allowed |
| `risk_level` | `float` | Risk assessment [0.0, 1.0] |
| `context` | `str` | Additional context |

### `InteractionResult`

**Type**: `@dataclass`

The resolved result of an interaction.

| Attribute | Type | Description |
|-----------|------|-------------|
| `choice` | `str` | The selected choice |
| `was_auto_decided` | `bool` | Whether auto-decided |
| `response_time_ms` | `float` | Response latency |
| `source` | `str` | `"user"`, `"auto"`, or `"timeout_default"` |

---

### `InteractionManager`

Manages agent-to-user interactions with autonomy levels and smart timeouts.

#### Constructor

```python
InteractionManager(
    default_autonomy: AutonomyLevel = AutonomyLevel.APPROVER,
    default_timeout: float = 30.0,
    risk_threshold: float = 0.7,
    confidence_threshold: float = 0.7,
    max_questions_per_task: int = 5,
)
```

#### Methods

##### `get_autonomy_level`

```python
def get_autonomy_level(
    self, risk_level: float, confidence: float,
    is_irreversible: bool = False, is_time_sensitive: bool = False,
) -> AutonomyLevel
```

Determine autonomy level from risk, confidence, and reversibility.

**Decision logic**:

| Condition | Result |
|-----------|--------|
| Irreversible action | L2 (COLLABORATOR) |
| High risk + low confidence | L3 (CONSULTANT) |
| Low risk + high confidence | L5 (AUTONOMOUS) |
| Time sensitive + low risk | L4 (APPROVER) |
| Default | Constructor default |

##### `should_ask_user`

```python
def should_ask_user(
    self, autonomy_level: AutonomyLevel, interaction_type: InteractionType,
) -> bool
```

Decide if user must be asked. Respects `max_questions_per_task`.

| Level | INFORMATION | CONFIRMATION | CHOICE | PERMISSION |
|-------|-------------|--------------|--------|------------|
| L1-L2 | Yes | Yes | Yes | Yes |
| L3 | No | Yes | No | Yes |
| L4 | No | No | No | Yes |
| L5 | No | No | No | No |

##### `create_request`

```python
def create_request(
    self, interaction_type: InteractionType, question: str,
    options: Optional[List[str]] = None, default_choice: Optional[str] = None,
    risk_level: float = 0.0, context: str = "",
) -> InteractionRequest
```

Create an interaction request with computed timeout. Higher risk = more time for user to respond.

##### `compute_timeout`

```python
def compute_timeout(self, risk_level: float, is_time_sensitive: bool = False) -> float
```

Smart timeout: `base * (1 + risk) * urgency`. Higher risk gives more time.

##### `build_auto_decide_prompt`

```python
def build_auto_decide_prompt(
    self, request: InteractionRequest,
    user_preferences: Dict[str, Any], context: str = "",
) -> str
```

Build prompt for LLM to predict user's choice based on preferences and recent decision history.

##### `parse_auto_decision`

```python
def parse_auto_decision(self, llm_response: str, request: InteractionRequest) -> str
```

Parse LLM auto-decision. Matches against options (case-insensitive), falls back to default or first option.

##### `resolve_with_default` / `resolve_with_user` / `resolve_with_auto`

Resolution methods for the three sources: timeout default, actual user input, and LLM auto-decision.

##### `record_decision`

```python
def record_decision(self, request: InteractionRequest, result: InteractionResult) -> None
```

Record a decision for learning and prompt building. Stores the question, interaction type, choice, source, risk level, and timestamp. Keeps the last 100 decisions. Called automatically by all `resolve_with_*` methods, but can also be called directly.

##### `set_user_callback` / `reset_task` / `get_stats`

Register user callback, reset per-task counters, and get interaction statistics.

---

## Usage Example

```python
from corteX.engine.interaction import InteractionManager, AutonomyLevel, InteractionType

manager = InteractionManager(default_autonomy=AutonomyLevel.APPROVER)

# Determine autonomy for this action
level = manager.get_autonomy_level(risk_level=0.8, confidence=0.4, is_irreversible=True)
# level = AutonomyLevel.COLLABORATOR (irreversible overrides)

if manager.should_ask_user(level, InteractionType.CONFIRMATION):
    request = manager.create_request(
        InteractionType.CONFIRMATION,
        "Delete all user data?",
        options=["yes", "no"],
        default_choice="no",
        risk_level=0.9,
    )
    # Present to user or auto-decide via LLM
    result = manager.resolve_with_user(request, "no")
```

---

## See Also

- [Agent Loop API](./agent-loop.md)
- [Policy Engine API](./policy-engine.md)
