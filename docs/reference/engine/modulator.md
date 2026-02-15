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
| `AMPLIFY` | Multiply up | `w_eff = w * factor` (factor > 1.0) |
| `DAMPEN` | Multiply down | `w_eff = w * factor` (factor < 1.0) |
| `CLAMP` | Lock at value | `w_eff = clamped_value` (ignores all other) |

### `ModulationScope`

**Type**: `str, Enum`

Temporal scope -- how long the "light" stays on.

| Value | Duration |
|-------|----------|
| `TURN` | One turn only |
| `GOAL` | Duration of current goal |
| `SESSION` | Entire session |
| `PERMANENT` | Until explicitly removed |
| `CONDITIONAL` | Until condition is met |

### `Modulation`

**Type**: `@dataclass`

A single modulation applied to a target.

| Attribute | Type | Description |
|-----------|------|-------------|
| `modulation_id` | `str` | Unique identifier |
| `target` | `str` | Target entity (tool, model, behavior) |
| `modulation_type` | `ModulationType` | Type of modulation |
| `scope` | `ModulationScope` | Temporal scope |
| `strength` | `float` | Force value (ACTIVATE/SILENCE) |
| `factor` | `float` | Multiplier (AMPLIFY/DAMPEN) |
| `clamped_value` | `float` | Fixed value (CLAMP) |
| `priority` | `int` | Higher = takes precedence |
| `source` | `str` | Who created it (user, enterprise, system) |
| `created_at` | `float` | Creation timestamp |
| `expires_at` | `Optional[float]` | Expiration timestamp |
| `condition` | `Optional[str]` | Condition for CONDITIONAL scope |

---

### `TargetedModulator`

Main modulation engine. Manages active modulations, applies them to weights, handles expiration and scope transitions.

#### Key Methods

##### `add_modulation`

Add a new modulation for a target entity. Returns the modulation ID.

##### `remove_modulation`

Remove a specific modulation by ID.

##### `apply`

Apply all active modulations to a weight value for a given target. Uses `ModulationConflictResolver` for precedence.

##### `tick`

Advance the modulation clock. Expires TURN-scoped modulations, checks CONDITIONAL scopes.

##### `get_active_modulations`

Get all currently active modulations, optionally filtered by target.

##### `clear_scope`

Remove all modulations of a given scope (e.g., clear all GOAL-scoped on goal change).

---

### `ModulationConflictResolver`

Resolves conflicts when multiple modulations target the same entity.

**Resolution priority**: `CLAMP > enterprise_policy > max(priority) > most_recent`

### `EnterpriseModulationPolicy`

Institutional-level modulation that overrides user control. Analogous to pharmacological interventions affecting entire neural circuits.

### `ConditionalModulator`

Manages CONDITIONAL scope modulations with programmable activation/deactivation conditions.

---

## Usage Example

```python
from corteX.engine.modulator import TargetedModulator, ModulationType, ModulationScope

modulator = TargetedModulator()

# Silence a dangerous tool for this goal
modulator.add_modulation(
    target="rm_rf",
    modulation_type=ModulationType.SILENCE,
    scope=ModulationScope.GOAL,
    priority=100,
    source="enterprise_policy",
)

# Amplify a preferred tool
modulator.add_modulation(
    target="code_writer",
    modulation_type=ModulationType.AMPLIFY,
    factor=1.5,
    scope=ModulationScope.SESSION,
    priority=50,
    source="user",
)

# Apply modulations to weight values
base_weight = 0.6
effective = modulator.apply("code_writer", base_weight)
# effective = 0.9 (0.6 * 1.5)

silenced = modulator.apply("rm_rf", 0.8)
# silenced = 0.0 (SILENCE overrides to 0)

# Advance turn -- expire TURN-scoped modulations
modulator.tick()
```

---

## See Also

- [Weight Engine API](./weight-engine.md)
- [Cortical Map Reorganizer API](./reorganization.md)
- [Policy Engine API](./policy-engine.md)
