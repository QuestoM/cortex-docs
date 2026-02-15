# Policy Engine API Reference

## Module: `corteX.engine.policy_engine`

Enterprise-configurable policy engine with five guardrail types. Inspired by CUGA's policy system (IBM Research). Pure evaluation engine -- no LLM calls, no I/O. Pattern matching via exact, glob (fnmatch), and regex (`re:` prefix).

---

## Classes

### `PolicyType`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `INTENT_GUARD` | Block or gate intents before execution |
| `PLAYBOOK` | Standardized workflows for task types |
| `TOOL_APPROVAL` | Control which tools can be used |
| `TOOL_GUIDE` | Domain knowledge/guidance for tools |
| `OUTPUT_FORMATTER` | Enforce output constraints |

### `PolicyAction`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `ALLOW` | Permit the action |
| `BLOCK` | Block the action |
| `REQUIRE_APPROVAL` | Require user approval first |
| `MODIFY` | Allow with modifications |
| `LOG` | Allow but log for audit |

### `PolicyRule`

**Type**: `@dataclass`

A single policy rule evaluated against actions, outputs, or intents.

| Attribute | Type | Description |
|-----------|------|-------------|
| `rule_id` | `str` | Unique rule identifier |
| `policy_type` | `PolicyType` | Type of guardrail |
| `name` | `str` | Human-readable name |
| `description` | `str` | Rule description |
| `condition` | `str` | Pattern: exact, glob (fnmatch), or regex (`re:` prefix) |
| `action` | `PolicyAction` | Action to take on match |
| `priority` | `int` | Higher priority evaluated first (default: 0) |
| `enabled` | `bool` | Whether the rule is active (default: True) |
| `metadata` | `Dict[str, Any]` | Additional rule metadata |

### `PolicyEvaluation`

**Type**: `@dataclass`

Result of evaluating a single rule.

| Attribute | Type | Description |
|-----------|------|-------------|
| `rule_id` | `str` | Rule that was evaluated |
| `policy_type` | `PolicyType` | Type of the rule |
| `action` | `PolicyAction` | Action determined |
| `matched` | `bool` | Whether the condition matched |
| `reason` | `str` | Reason for the evaluation |
| `metadata` | `Dict[str, Any]` | Rule metadata |

### `PolicyResult`

**Type**: `@dataclass`

Aggregate result of evaluating all rules against an action.

| Attribute | Type | Description |
|-----------|------|-------------|
| `allowed` | `bool` | Whether the action is allowed |
| `evaluations` | `List[PolicyEvaluation]` | All individual evaluations |
| `blocked_by` | `Optional[str]` | Rule ID that blocked (if any) |
| `requires_approval` | `bool` | Whether approval is needed |
| `modifications` | `List[str]` | Required modifications |

**Properties**: `summary` -- returns `"allowed"`, `"requires_approval (rule: ...)"`, or `"blocked (rule: ...)"`.

---

### `PolicyEngine`

Enterprise-configurable policy engine with five guardrail types.

#### Constructor

```python
PolicyEngine()
```

No parameters. Rules are registered after construction.

#### Methods

##### `register_rule` / `remove_rule`

```python
def register_rule(self, rule: PolicyRule) -> None
def remove_rule(self, rule_id: str) -> bool
```

Register or remove policy rules. Rules are maintained in priority order (highest first).

##### `evaluate_tool_call`

```python
def evaluate_tool_call(
    self, tool_name: str, tool_args: Dict[str, Any],
    context: Optional[Dict[str, Any]] = None,
) -> PolicyResult
```

Evaluate a tool call against TOOL_APPROVAL, TOOL_GUIDE, and INTENT_GUARD policies. Supports approval requirements.

##### `evaluate_output`

```python
def evaluate_output(
    self, output: str, context: Optional[Dict[str, Any]] = None,
) -> PolicyResult
```

Evaluate agent output against OUTPUT_FORMATTER policies.

##### `evaluate_intent`

```python
def evaluate_intent(
    self, intent: str, context: Optional[Dict[str, Any]] = None,
) -> PolicyResult
```

Evaluate user/agent intent against INTENT_GUARD policies.

##### `get_tool_guide` / `get_playbook`

```python
def get_tool_guide(self, tool_name: str) -> Optional[str]
def get_playbook(self, task_type: str) -> Optional[str]
```

Retrieve domain knowledge for a tool or standardized workflow for a task type.

##### `get_active_rules` / `get_stats`

Get active rules (optionally filtered by type) and engine statistics.

#### Static Factory Methods

```python
PolicyEngine.create_blocked_tool_rule(tool_name: str, reason: str = "") -> PolicyRule
PolicyEngine.create_approval_required_rule(tool_pattern: str, reason: str = "") -> PolicyRule
PolicyEngine.create_blocked_topic_rule(topic_pattern: str) -> PolicyRule
```

Convenience methods for common rule patterns.

---

## Pattern Matching

The `condition` field supports three matching modes:

| Mode | Syntax | Example |
|------|--------|---------|
| Exact (substring) | plain string | `"file_delete"` |
| Glob | uses `*`, `?`, `[`, `]` | `"file_*"` |
| Regex | prefix `re:` | `"re:^(delete\|drop)_"` |

All matching is case-insensitive.

---

## Usage Example

```python
from corteX.engine.policy_engine import PolicyEngine, PolicyRule, PolicyType, PolicyAction

policy = PolicyEngine()

# Block dangerous tools
policy.register_rule(PolicyEngine.create_blocked_tool_rule("rm_rf", "Too dangerous"))

# Require approval for database operations
policy.register_rule(PolicyEngine.create_approval_required_rule("db_*", "Database ops need approval"))

# Add a playbook
policy.register_rule(PolicyRule(
    rule_id="playbook_deploy",
    policy_type=PolicyType.PLAYBOOK,
    name="Deployment Playbook",
    description="Standard deployment workflow",
    condition="deploy*",
    action=PolicyAction.MODIFY,
    metadata={"playbook": "1. Run tests 2. Build 3. Stage 4. Deploy"},
))

# Evaluate a tool call
result = policy.evaluate_tool_call("db_migrate", {"target": "production"})
print(result.summary)  # "requires_approval (rule: approve_tool_...)"

# Get deployment playbook
playbook = policy.get_playbook("deployment")
```

---

## See Also

- [Agent Loop API](./agent-loop.md)
- [Interaction Manager API](./interaction.md)
