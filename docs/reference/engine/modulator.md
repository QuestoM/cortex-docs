# Targeted Modulator API Reference

## Module: `corteX.engine.modulator`

Optogenetics-inspired precision control for AI agent behavior. Provides temporary, targeted overrides of specific tools, models, and behaviors without mutating underlying learned weights.

Neuroscience foundation (Prof. Idan Segev, Lecture 3): *"I can now decide that I want only ONE cell type to be electrically activated... and I have materials that can both activate AND silence."*

---

## Architecture

The modulator sits between the WeightEngine and the Orchestrator:

```
WeightEngine.get_weights()
      |
      v
TargetedModulator.apply_modulations(weights)
      |  <-- applies ACTIVATE, SILENCE, AMPLIFY, DAMPEN, CLAMP
      |  <-- evaluates enterprise policies
      |  <-- evaluates conditional modulations
      |  <-- enforces safety boundaries
      |  <-- resolves conflicts
      v
Orchestrator receives modulated weights
```

**Mathematical model**: Let `w(t)` be the current weight for target `t`. The effective weight is computed by the `ModulationConflictResolver` from all active modulations targeting `t`.

**Conflict resolution priority**: `CLAMP > enterprise_policy > max(priority) > most_recent`

---

## Enums

### `ModulationType(str, Enum)`

Modulation types inspired by optogenetic actuators.

| Value | Effect | Formula |
|-------|--------|---------|
| `ACTIVATE` | Force-enable (ChR2 blue light) | `w_eff = strength` (clamped to [0, 1]) |
| `SILENCE` | Force-suppress (NpHR yellow light) | `w_eff = 0.0` |
| `AMPLIFY` | Multiply up | `w_eff = w * strength` (strength >= 1.0) |
| `DAMPEN` | Multiply down | `w_eff = w * strength` (strength in [0, 1]) |
| `CLAMP` | Lock at fixed value | `w_eff = strength` (clamped to [-1, 1], ignores all others) |

### `ModulationScope(str, Enum)`

Temporal scope -- how long the "light" stays on.

| Value | Duration |
|-------|----------|
| `TURN` | Active for N turns (set via `scope_param`), then auto-expires |
| `GOAL` | Active until the current goal changes |
| `SESSION` | Active for the entire session |
| `PERMANENT` | Never expires until explicitly removed |
| `CONDITIONAL` | Active while a condition function evaluates to True |

---

## Data Classes

### `Modulation`

A single active modulation -- one "optogenetic probe" targeting a specific behavior, tool, or model.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `modulation_id` | `str` | UUID hex (12 chars) | Unique identifier |
| `target` | `str` | `""` | Entity being modulated (tool name, model id, behavior key) |
| `modulation_type` | `ModulationType` | `ACTIVATE` | How the weight is transformed |
| `scope` | `ModulationScope` | `SESSION` | Temporal extent |
| `scope_param` | `Any` | `None` | Scope-specific parameter (e.g., N for TURN, condition expr for CONDITIONAL) |
| `strength` | `float` | `1.0` | Intensity. ACTIVATE: forced weight. AMPLIFY/DAMPEN: multiplication factor. CLAMP: clamped value. SILENCE: ignored |
| `reason` | `str` | `""` | Human-readable explanation |
| `created_at` | `float` | `time.time()` | Creation timestamp |
| `expires_at` | `Optional[float]` | `None` | Absolute expiration timestamp |
| `priority` | `int` | `0` | Conflict resolution priority (higher wins). Enterprise policies use >= 100 |
| `source` | `str` | `"user"` | Creator (user, enterprise_policy, system) |
| `_active` | `bool` | `True` | Whether currently active (internal) |
| `_turns_remaining` | `Optional[int]` | `None` | Remaining turns for TURN scope (internal) |
| `_initial_goal` | `Optional[str]` | `None` | Goal ID at creation time for GOAL scope (internal) |

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `is_active` | `() -> bool` | Check if modulation is still active (respects time/turn expiration) |
| `remaining_scope` | `() -> str` | Human-readable description of remaining duration |
| `apply_to` | `(current_value: float) -> float` | Apply this modulation to a weight value |
| `tick` | `() -> None` | Advance one turn; decrements counter for TURN scope |
| `check_goal` | `(current_goal: Optional[str]) -> None` | Expire if goal changed (GOAL scope) |
| `deactivate` | `() -> None` | Manually deactivate this modulation |
| `to_dict` | `() -> Dict[str, Any]` | Serialize to dictionary for persistence/audit |

### `ConflictReport`

Report of conflicting modulations on the same target.

| Attribute | Type | Description |
|-----------|------|-------------|
| `target` | `str` | The entity with conflicting modulations |
| `conflicting_modulations` | `List[Modulation]` | The modulations in conflict |
| `resolution` | `str` | Description of how the conflict was resolved |
| `resolved_value` | `float` | The final resolved weight value |
| `timestamp` | `float` | When the conflict was detected |

### `Policy`

An enterprise-level modulation policy with tamper detection.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `policy_id` | `str` | `"pol_"` + UUID | Unique identifier |
| `name` | `str` | `""` | Human-readable name |
| `description` | `str` | `""` | What the policy does and why |
| `target_pattern` | `str` | `""` | Pattern matching targets (`"tool_x"`, `"tool_*"`, `"*"`) |
| `modulation_type` | `ModulationType` | `SILENCE` | What modulation to apply |
| `strength` | `float` | `0.0` | Modulation strength/factor/value |
| `priority` | `int` | `100` | Always >= 100 for enterprise policies (enforced) |
| `enabled` | `bool` | `True` | Whether the policy is active |
| `conditions` | `Dict[str, Any]` | `{}` | Conditions that must be met (supports `__gt`, `__lt`, `__gte`, `__lte`, `__ne`, `__in` suffixes) |
| `created_by` | `str` | `"admin"` | Admin who created this policy |
| `created_at` | `float` | `time.time()` | Creation timestamp |

#### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `verify_integrity` | `() -> bool` | SHA-256 tamper detection on critical fields |
| `matches_target` | `(target: str) -> bool` | Check if target matches pattern (supports `*` wildcard) |
| `evaluate_conditions` | `(context: Dict[str, Any]) -> bool` | Evaluate conditions against context (conjunction) |
| `to_dict` | `() -> Dict[str, Any]` | Serialize including integrity hash |

### `AuditEntry`

Immutable audit log entry for enterprise policy operations. Supports SOC 2/HIPAA compliance.

| Attribute | Type | Description |
|-----------|------|-------------|
| `entry_id` | `str` | Unique identifier |
| `timestamp` | `float` | When the event occurred |
| `event_type` | `str` | Event kind: `policy_activated`, `policy_evaluated`, `modulation_applied`, etc. |
| `policy_id` | `Optional[str]` | Related policy ID |
| `modulation_id` | `Optional[str]` | Related modulation ID |
| `target` | `str` | Target entity |
| `details` | `Dict[str, Any]` | Event-specific details |
| `actor` | `str` | Who triggered the event (user, system, admin) |

---

## Classes

### `TargetedModulator`

The main modulation engine. Manages active modulations, enterprise policies, conditional modulations, safety boundaries, conflict resolution, and audit trails.

#### Convenience Methods (Creating Modulations)

##### `activate(target, scope, strength, reason, priority, scope_param) -> Modulation`

Force-activate a target (ChR2 -- blue light ON). Sets the target's weight to `strength`.

```python
modulator.activate(
    target="preferred_model",
    strength=0.9,
    scope=ModulationScope.GOAL,
    reason="complex reasoning task"
)
```

##### `silence(target, scope, reason, priority, scope_param) -> Modulation`

Force-silence a target (NpHR -- yellow light ON). Sets the target's weight to 0.0.

```python
modulator.silence(
    "dangerous_tool",
    scope=ModulationScope.TURN,
    scope_param=3,
    reason="investigating safety issue"
)
```

##### `amplify(target, factor, scope, reason, priority, scope_param) -> Modulation`

Amplify a target's weight by multiplying it by `factor` (>= 1.0).

```python
modulator.amplify("claude-opus-4-6", factor=1.5, scope=ModulationScope.GOAL)
```

##### `dampen(target, factor, scope, reason, priority, scope_param) -> Modulation`

Dampen a target's weight by multiplying it by `factor` (0.0 to 1.0).

```python
modulator.dampen("expensive_model", factor=0.3, scope=ModulationScope.SESSION)
```

##### `clamp(target, value, scope, reason, priority, scope_param) -> Modulation`

Lock a target's weight to a fixed `value` ([-1.0, 1.0]). Strongest modulation type -- overrides all others. Default priority is 50.

```python
modulator.clamp("critical_tool", value=0.8, scope=ModulationScope.PERMANENT)
```

#### Core Methods

##### `apply_modulations(weights, context, safety_policy, safety_critical_tools) -> Dict[str, float]`

Apply all active modulations to a weight dictionary. This is the main interface called between the weight engine and the orchestrator.

**Steps performed**:

1. Evaluate conditional modulations against context
2. Collect all user/manual modulations
3. Evaluate enterprise policies for all targets
4. Enforce safety boundaries (blocks SILENCE/DAMPEN on safety-critical tools when safety policy is STRICT/LOCKED)
5. Resolve conflicts per target via `ModulationConflictResolver`
6. Log audit entry

**Parameters**:

| Parameter | Type | Description |
|-----------|------|-------------|
| `weights` | `Dict[str, float]` | Weight dictionary from weight engine (NOT mutated) |
| `context` | `Optional[Dict[str, Any]]` | Execution context for conditional/policy evaluation |
| `safety_policy` | `Optional[SafetyPolicy]` | When provided, prevents suppression of safety-critical tools |
| `safety_critical_tools` | `Optional[Set[str]]` | Tool names considered safety-critical |

**Returns**: New dictionary with modulated weights.

##### `tick(current_goal=None) -> None`

Advance the turn counter. Decrements TURN-scoped modulations, checks GOAL-scoped modulations for goal changes, and removes expired modulations.

##### `remove(modulation_id) -> bool`

Manually remove a modulation by ID. Returns `True` if found and removed.

##### `clear_all() -> None`

Remove all modulations and conditional registrations. Does NOT clear enterprise policies.

##### `set_goal(goal_id) -> None`

Update the current goal. GOAL-scoped modulations expire when this changes.

##### `detect_conflicts() -> List[ConflictReport]`

Detect and report all current modulation conflicts among active modulations.

#### Enterprise Policy Methods

##### `set_enterprise_policy(policy) -> None`

Register an enterprise policy (priority >= 100, overrides user modulations).

##### `remove_enterprise_policy(policy_id) -> bool`

Remove an enterprise policy by ID.

#### Conditional Modulation Methods

##### `register_condition(condition_expr, modulation) -> None`

Register a conditional modulation. Active whenever `condition_expr` (e.g., `"error_rate > 0.3"`) evaluates to True.

##### `update_condition_context(key, value) -> None`

Update a context variable for conditional evaluation (e.g., `error_rate`, `quality_score`).

#### Query & Audit Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `get_active_modulations()` | `List[Modulation]` | All currently active modulations |
| `get_modulations_for_target(target)` | `List[Modulation]` | Active modulations for a specific target |
| `get_stats()` | `Dict[str, Any]` | Comprehensive statistics (counts, by-type, by-scope, enterprise stats, etc.) |
| `get_audit_trail()` | `List[Dict[str, Any]]` | Full chronological audit trail (modulator + enterprise combined) |
| `to_dict()` | `Dict[str, Any]` | Serialize modulator state for persistence |

---

### `ModulationConflictResolver`

Resolves conflicts when multiple modulations target the same entity.

| Method | Signature | Description |
|--------|-----------|-------------|
| `resolve` | `(modulations: List[Modulation], current_value: float) -> float` | Resolve all active modulations for one target into a single value |
| `detect_conflicts` | `(modulations: List[Modulation]) -> List[ConflictReport]` | Detect ACTIVATE-vs-SILENCE, multiple CLAMPs, and AMPLIFY-vs-DAMPEN conflicts |
| `conflict_history` | `property -> List[ConflictReport]` | Full conflict history for audit |

### `EnterpriseModulationPolicy`

Enterprise-tier policy engine with tamper detection and audit logging.

| Method | Signature | Description |
|--------|-----------|-------------|
| `add_policy` | `(policy: Policy) -> None` | Register a policy (raises `ValueError` if integrity check fails) |
| `remove_policy` | `(policy_id: str) -> bool` | Remove a policy by ID |
| `evaluate` | `(target: str, context: Optional[Dict]) -> Optional[Modulation]` | Evaluate policies against a target; returns generated Modulation or None |
| `get_active_policies` | `() -> List[Policy]` | All enabled policies sorted by priority (highest first) |
| `audit_log` | `() -> List[AuditEntry]` | Complete audit trail |
| `get_stats` | `() -> Dict[str, Any]` | Policy engine statistics |

### `ConditionalModulator`

Handles CONDITIONAL-scope modulations -- closed-loop optogenetics. Conditions are re-evaluated each turn; modulations activate/deactivate dynamically.

Supported condition expressions: `"variable operator value"` where operator is `>`, `<`, `>=`, `<=`, `==`, `!=`.

| Method | Signature | Description |
|--------|-----------|-------------|
| `register_condition` | `(condition_expr: str, modulation: Modulation) -> None` | Register a condition-modulation pair |
| `evaluate_condition` | `(expr: str, context: Dict) -> bool` | Evaluate a single condition expression |
| `evaluate_all` | `(context: Dict) -> List[Modulation]` | Evaluate all conditions; return active modulations |
| `update_context` | `(key: str, value: Any) -> None` | Update a context variable |
| `get_context` | `() -> Dict[str, Any]` | Current context snapshot |
| `clear` | `() -> None` | Remove all registered conditions |

---

## Usage Example

```python
from corteX.engine.modulator import (
    TargetedModulator, ModulationType, ModulationScope, Modulation, Policy,
)

modulator = TargetedModulator()

# Silence a dangerous tool for 3 turns
modulator.silence(
    "rm_rf",
    scope=ModulationScope.TURN,
    scope_param=3,
    reason="investigating safety issue",
)

# Amplify a preferred model for the current goal
modulator.amplify(
    "claude-opus-4-6",
    factor=1.5,
    scope=ModulationScope.GOAL,
    reason="complex reasoning task",
)

# Clamp a critical tool at fixed weight
modulator.clamp("safety_checker", value=0.95, scope=ModulationScope.PERMANENT)

# Set an enterprise policy: block all tools matching "debug_*"
modulator.set_enterprise_policy(Policy(
    name="block_debug_tools",
    target_pattern="debug_*",
    modulation_type=ModulationType.SILENCE,
    created_by="security_admin",
))

# Register a conditional modulation: dampen risky tool when errors spike
modulator.register_condition(
    "error_rate > 0.3",
    Modulation(target="risky_tool", modulation_type=ModulationType.DAMPEN, strength=0.2),
)
modulator.update_condition_context("error_rate", 0.45)

# Apply modulations to weights from the weight engine
raw_weights = {"claude-opus-4-6": 0.6, "rm_rf": 0.8, "safety_checker": 0.5}
modulated = modulator.apply_modulations(raw_weights)
# modulated["claude-opus-4-6"] = 0.9  (0.6 * 1.5)
# modulated["rm_rf"] = 0.0          (SILENCE -> 0.0)
# modulated["safety_checker"] = 0.95 (CLAMP -> 0.95)

# Advance turn -- expire TURN-scoped modulations
modulator.tick()

# Remove a modulation manually
mod = modulator.activate("some_tool", reason="temporary boost")
modulator.remove(mod.modulation_id)

# Clear all modulations (does NOT clear enterprise policies)
modulator.clear_all()
```

---

## See Also

- [Weight Engine API](./weight-engine.md)
- [Cortical Map Reorganizer API](./reorganization.md)
