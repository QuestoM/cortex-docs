# Targeted Modulator API Reference

## Module: `corteX.engine.modulator`

Optogenetics-inspired precision control for AI agent behavior. Provides temporary, targeted overrides of specific tools, models, and behaviors without mutating underlying learned weights.

Neuroscience foundation (Prof. Idan Segev, Lecture 3): "I can now decide that I want only ONE cell type to be electrically activated... and I have materials that can both activate AND silence."

---

## Core Concepts

The modulator sits between the WeightEngine and the Orchestrator:

```
WeightEngine -> [TargetedModulator] -> Orchestrator
```

Modulations are temporary overrides applied on top of learned weights, not replacing them.

**Mathematical model**: Let `w(t)` be the current weight for target `t`. The effective weight is computed by the ModulationConflictResolver from active modulations.

---

## Classes

### `ModulationType`

**Type**: `str, Enum`

Inspired by optogenetic actuators (channelrhodopsin, halorhodopsin).

| Value | Effect | Formula |
|-------|--------|---------|
| `ACTIVATE` | Force-enable | `w_eff = strength` |
| `SILENCE` | Force-suppress | `w_eff = 0.0` |
| `AMPLIFY` | Multiply up | `w_eff = w * strength` (strength >= 1.0) |
| `DAMPEN` | Multiply down | `w_eff = w * strength` (strength 0.0 to 1.0) |
| `CLAMP` | Lock at value | `w_eff = strength` (ignores all other) |

### `ModulationScope`

**Type**: `str, Enum`

Temporal scope - how long the "light" stays on.

| Value | Duration |
|-------|----------|
| `TURN` | Active for N turns (set via `scope_param`), then auto-expires |
| `GOAL` | Active until the current goal changes |
| `SESSION` | Active for the entire session |
| `PERMANENT` | Never expires until explicitly removed |
| `CONDITIONAL` | Active while a condition evaluates to True |

### `Modulation`

**Type**: `@dataclass`

A single modulation applied to a target. The `strength` field is reused for all modulation types: for ACTIVATE it is the forced weight, for AMPLIFY/DAMPEN it is the multiplication factor, for CLAMP it is the clamped value.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `modulation_id` | `str` | auto-generated | Unique identifier (uuid hex, 12 chars) |
| `target` | `str` | `""` | Target entity (tool name, model id, behavior key) |
| `modulation_type` | `ModulationType` | `ACTIVATE` | Type of modulation |
| `scope` | `ModulationScope` | `SESSION` | Temporal scope |
| `scope_param` | `Any` | `None` | Scope-specific parameter (e.g., turn count for TURN scope) |
| `strength` | `float` | `1.0` | Modulation intensity - used for all types (see above) |
| `reason` | `str` | `""` | Human-readable explanation |
| `created_at` | `float` | `time.time()` | Creation timestamp |
| `expires_at` | `Optional[float]` | `None` | Optional absolute expiration timestamp |
| `priority` | `int` | `0` | Conflict resolution priority (higher wins) |
| `source` | `str` | `"user"` | Who created it (user, enterprise_policy, system) |
| `_active` | `bool` | `True` | Whether this modulation is currently active |
| `_turns_remaining` | `Optional[int]` | `None` | For TURN scope, turns left before expiry |
| `_initial_goal` | `Optional[str]` | `None` | For GOAL scope, the goal ID at creation time |

#### Modulation Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `is_active` | `() -> bool` | Check if this modulation is still active |
| `apply_to` | `(current_value: float) -> float` | Apply this modulation to a weight value |
| `tick` | `() -> None` | Advance one turn; decrements TURN-scoped counter |
| `check_goal` | `(current_goal: Optional[str]) -> None` | Expire if goal changed (for GOAL scope) |
| `deactivate` | `() -> None` | Manually deactivate this modulation |
| `remaining_scope` | `() -> str` | Human-readable remaining scope description |
| `to_dict` | `() -> Dict[str, Any]` | Serialize to dictionary for persistence |

### `ConflictReport`

**Type**: `@dataclass`

Report of conflicting modulations on the same target.

| Attribute | Type | Description |
|-----------|------|-------------|
| `target` | `str` | The entity with conflicting modulations |
| `conflicting_modulations` | `List[Modulation]` | The conflicting modulation objects |
| `resolution` | `str` | How the conflict was resolved |
| `resolved_value` | `float` | The final value after resolution |
| `timestamp` | `float` | When the conflict was detected |

### `Policy`

**Type**: `@dataclass`

Enterprise-level modulation policy with tamper detection. Policies generate modulations automatically and have priority >= 100.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `policy_id` | `str` | auto-generated | Unique identifier |
| `name` | `str` | `""` | Human-readable policy name |
| `description` | `str` | `""` | What this policy does and why |
| `target_pattern` | `str` | `""` | Pattern matching targets (exact, `"tool_*"`, or `"*"`) |
| `modulation_type` | `ModulationType` | `SILENCE` | What kind of modulation to apply |
| `strength` | `float` | `0.0` | Modulation strength/factor/clamped value |
| `priority` | `int` | `100` | Always >= 100 for enterprise policies |
| `enabled` | `bool` | `True` | Whether this policy is active |
| `conditions` | `Dict[str, Any]` | `{}` | Optional conditions that must be met |
| `created_by` | `str` | `"admin"` | Admin who created this policy |
| `created_at` | `float` | `time.time()` | Timestamp of creation |

#### Policy Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `verify_integrity` | `() -> bool` | Check that the policy has not been tampered with |
| `matches_target` | `(target: str) -> bool` | Check if a target matches the policy pattern |
| `evaluate_conditions` | `(context: Dict[str, Any]) -> bool` | Evaluate conditions against context |
| `to_dict` | `() -> Dict[str, Any]` | Serialize to dictionary |

### `AuditEntry`

**Type**: `@dataclass`

Immutable audit log entry for enterprise policy activations. Supports SOC2/HIPAA compliance.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `entry_id` | `str` | auto-generated | Unique identifier |
| `timestamp` | `float` | `time.time()` | When the event occurred |
| `event_type` | `str` | `""` | Event type (policy_activated, etc.) |
| `policy_id` | `Optional[str]` | `None` | Related policy ID |
| `modulation_id` | `Optional[str]` | `None` | Related modulation ID |
| `target` | `str` | `""` | Target entity |
| `details` | `Dict[str, Any]` | `{}` | Additional event details |
| `actor` | `str` | `""` | Who triggered this (user, system, admin) |

---

### `ModulationConflictResolver`

Resolves conflicts when multiple modulations target the same entity.

**Resolution priority**: `CLAMP > enterprise_policy > max(priority) > most_recent`

| Method | Signature | Description |
|--------|-----------|-------------|
| `resolve` | `(modulations: List[Modulation], current_value: float) -> float` | Resolve all active modulations for one target |
| `detect_conflicts` | `(modulations: List[Modulation]) -> List[ConflictReport]` | Detect and report conflicts |

---

### `TargetedModulator`

Main modulation engine. Manages active modulations, applies them to weights, handles expiration, scope transitions, enterprise policies, conditional modulations, and safety boundary enforcement.

#### Constructor

```python
TargetedModulator()
```

No constructor parameters. Initializes internal conflict resolver, enterprise policy engine, conditional modulator, and audit trail.

#### Convenience Methods (Creating Modulations)

##### `activate`

```python
activate(
    target: str,
    scope: ModulationScope = ModulationScope.SESSION,
    strength: float = 1.0,
    reason: str = "",
    priority: int = 0,
    scope_param: Any = None,
) -> Modulation
```

Force-activate a target (ChR2 - blue light ON). Sets the target's weight to `strength`, overriding the learned weight.

##### `silence`

```python
silence(
    target: str,
    scope: ModulationScope = ModulationScope.SESSION,
    reason: str = "",
    priority: int = 0,
    scope_param: Any = None,
) -> Modulation
```

Force-silence a target (NpHR - yellow light ON). Sets the target's weight to 0.0.

##### `amplify`

```python
amplify(
    target: str,
    factor: float = 1.5,
    scope: ModulationScope = ModulationScope.SESSION,
    reason: str = "",
    priority: int = 0,
    scope_param: Any = None,
) -> Modulation
```

Amplify a target's weight. Multiplies the current weight by `factor` (clamped to >= 1.0). The factor is stored in the `strength` field of the resulting Modulation.

##### `dampen`

```python
dampen(
    target: str,
    factor: float = 0.5,
    scope: ModulationScope = ModulationScope.SESSION,
    reason: str = "",
    priority: int = 0,
    scope_param: Any = None,
) -> Modulation
```

Dampen a target's weight. Multiplies the current weight by `factor` (clamped to 0.0-1.0). The factor is stored in the `strength` field.

##### `clamp`

```python
clamp(
    target: str,
    value: float,
    scope: ModulationScope = ModulationScope.SESSION,
    reason: str = "",
    priority: int = 50,
    scope_param: Any = None,
) -> Modulation
```

Clamp a target's weight to a fixed value (voltage clamp). Locks at `value` (clamped to -1.0 to 1.0), ignoring all other modulations. CLAMP implicitly wins conflict resolution regardless of priority.

#### Core Methods

##### `apply_modulations`

```python
apply_modulations(
    weights: Dict[str, float],
    context: Optional[Dict[str, Any]] = None,
    safety_policy: Optional[Any] = None,
    safety_critical_tools: Optional[Set[str]] = None,
) -> Dict[str, float]
```

Apply all active modulations to a weight dictionary. This is the main interface called between the weight engine and the orchestrator. The input `weights` dict is NOT mutated; a new dict is returned.

Steps: (1) Evaluate conditional modulations, (2) Collect all modulation sources, (3) Evaluate enterprise policies, (4) Enforce safety boundaries if `safety_policy` provided, (5) Resolve conflicts per target, (6) Log audit entry.

**Safety boundary enforcement**: When `safety_policy` is provided and `safety_critical_tools` lists tool names, any modulation that would SILENCE or DAMPEN a safety-critical tool is blocked when the safety policy level is STRICT or LOCKED, or when injection_protection or content_filtering is enabled.

##### `tick`

```python
tick(current_goal: Optional[str] = None) -> None
```

Advance the turn counter and expire time-based modulations. Call once per agent turn. Decrements TURN-scoped modulations, checks GOAL-scoped for goal changes, and removes expired modulations.

##### `remove`

```python
remove(modulation_id: str) -> bool
```

Manually remove a modulation by ID. Returns True if found and removed, False if not found.

##### `get_active_modulations`

```python
get_active_modulations() -> List[Modulation]
```

Return all currently active modulations.

##### `get_modulations_for_target`

```python
get_modulations_for_target(target: str) -> List[Modulation]
```

Return all active modulations targeting a specific entity.

#### Enterprise Policy Methods

##### `set_enterprise_policy`

```python
set_enterprise_policy(policy: Policy) -> None
```

Set an enterprise modulation policy. Enterprise policies have priority >= 100 and override user modulations. They are checked during `apply_modulations()` for every target.

##### `remove_enterprise_policy`

```python
remove_enterprise_policy(policy_id: str) -> bool
```

Remove an enterprise policy by ID.

#### Conditional Modulation Methods

##### `register_condition`

```python
register_condition(condition_expr: str, modulation: Modulation) -> None
```

Register a conditional modulation (closed-loop control). The modulation will be active whenever `condition_expr` evaluates to True against the current context. Example: `"error_rate > 0.3"`.

##### `update_condition_context`

```python
update_condition_context(key: str, value: Any) -> None
```

Update a context variable for conditional modulation evaluation. Supported keys include: `error_rate`, `quality_score`, `turn_count`, `goal_drift`, `surprise_level`, and any custom metric.

#### Goal Management

##### `set_goal`

```python
set_goal(goal_id: str) -> None
```

Update the current goal. GOAL-scoped modulations will expire when this changes from their creation-time goal.

#### Conflict Detection

##### `detect_conflicts`

```python
detect_conflicts() -> List[ConflictReport]
```

Detect and report all current modulation conflicts among active modulations.

#### Statistics, Audit, and Serialization

##### `get_stats`

```python
get_stats() -> Dict[str, Any]
```

Return comprehensive statistics: `turn_count`, `current_goal`, `active_modulations`, `total_modulations_created`, `total_modulations_expired`, `total_apply_calls`, `modulations_by_type`, `modulations_by_scope`, `modulated_targets`, `enterprise_policy_stats`, `conditional_stats`, `conflict_history_size`, `audit_trail_size`.

##### `get_audit_trail`

```python
get_audit_trail() -> List[Dict[str, Any]]
```

Return the full audit trail of all modulation operations, combined with the enterprise policy engine's audit log, sorted chronologically.

##### `to_dict`

```python
to_dict() -> Dict[str, Any]
```

Serialize the modulator state for persistence.

##### `clear_all`

```python
clear_all() -> None
```

Remove all user and conditional modulations. Does NOT clear enterprise policies - those require explicit removal via `remove_enterprise_policy()`.

---

## Usage Example

```python
from corteX.engine.modulator import (
    TargetedModulator, ModulationType, ModulationScope, Policy
)

modulator = TargetedModulator()

# Silence a dangerous tool for this goal
modulator.silence(
    target="rm_rf",
    scope=ModulationScope.GOAL,
    priority=100,
    reason="enterprise safety policy",
)

# Amplify a preferred tool
modulator.amplify(
    target="code_writer",
    factor=1.5,
    scope=ModulationScope.SESSION,
    priority=50,
    reason="complex coding task",
)

# Clamp a tool to exact value
modulator.clamp(
    target="search_engine",
    value=0.8,
    scope=ModulationScope.SESSION,
    reason="fixed priority for search",
)

# Apply modulations to weight dictionary from weight engine
raw_weights = {"code_writer": 0.6, "rm_rf": 0.8, "search_engine": 0.5}
modulated = modulator.apply_modulations(raw_weights)
# modulated = {"code_writer": 0.9, "rm_rf": 0.0, "search_engine": 0.8}

# Register a conditional modulation
from corteX.engine.modulator import Modulation
error_mod = Modulation(
    target="fallback_model",
    modulation_type=ModulationType.ACTIVATE,
    scope=ModulationScope.CONDITIONAL,
    strength=1.0,
    reason="activate fallback when errors are high",
)
modulator.register_condition("error_rate > 0.3", error_mod)

# Set enterprise policy
policy = Policy(
    name="no_shell_access",
    target_pattern="shell_*",
    modulation_type=ModulationType.SILENCE,
    strength=0.0,
    priority=200,
    description="Block all shell tools for security",
)
modulator.set_enterprise_policy(policy)

# Advance turn - expire TURN-scoped modulations
modulator.tick()

# Check statistics
stats = modulator.get_stats()
```

---

## See Also

- [Weight Engine API](./weights.md)
- [Cortical Map Reorganizer API](./reorganization.md)
- [Policy Engine API](./policy-engine.md)
