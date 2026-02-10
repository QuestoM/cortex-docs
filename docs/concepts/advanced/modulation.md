# Targeted Modulation

The Targeted Modulator provides optogenetics-inspired precision control over specific tools, models, and behaviors. It can activate, silence, amplify, dampen, or clamp individual components without modifying the underlying learned weights.

## What It Does

The modulator sits between the Weight Engine and the Orchestrator, applying temporary overrides:

```
WeightEngine (learned weights)
       |
       v
[TargetedModulator] -- applies temporary overrides
       |
       v
Orchestrator (effective weights)
```

Five modulation types, each inspired by optogenetic actuators:

| Type | Effect | Analogy |
|------|--------|---------|
| **ACTIVATE** | Force-enable (override weight to strength value) | ChR2 (blue light) |
| **SILENCE** | Force-suppress (override weight to 0.0) | NpHR / halorhodopsin (yellow light) |
| **AMPLIFY** | Multiply weight by factor > 1.0 | Varying light intensity (increase) |
| **DAMPEN** | Multiply weight by factor < 1.0 | Varying light intensity (decrease) |
| **CLAMP** | Lock at fixed value, ignore all updates | Sustained stimulation |

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Optogenetics"
    "I can now decide that I want only ONE cell type to be electrically activated... and I have materials that can both activate AND silence. In short, this gives me better control -- I can do up and down, both activate and silence." -- Prof. Idan Segev

    Optogenetics (Boyden et al., 2005; Deisseroth, 2011) introduces light-sensitive proteins into specific neuron types, allowing researchers to:
    1. **ACTIVATE** specific neurons with blue light (ChR2 / channelrhodopsin)
    2. **SILENCE** specific neurons with yellow light (NpHR / halorhodopsin)
    3. Control is **TEMPORARY**: light on = modulation active, light off = normal
    4. Control is **TARGETED**: only neurons expressing the opsin are affected
    5. This operates **OVER** the normal synaptic weight system, not replacing it

    The Targeted Modulator translates these principles: modulations are temporary, targeted, and sit on top of learned weights.

## How It Works

### Creating Modulations

```python
from corteX.engine.modulator import (
    TargetedModulator,
    ModulationType,
    ModulationScope,
)

modulator = TargetedModulator()

# Silence a dangerous tool for this goal
modulator.silence(
    "dangerous_tool",
    reason="Investigating safety issue",
    scope=ModulationScope.GOAL,
)

# Amplify a successful tool for this session
modulator.amplify(
    "code_interpreter",
    factor=1.5,
    reason="Working well for current project",
    scope=ModulationScope.SESSION,
)

# Clamp safety strictness at maximum
modulator.clamp(
    "safety_strictness",
    value=1.0,
    reason="HIPAA compliance requirement",
    scope=ModulationScope.PERMANENT,
)
```

### Modulation Scopes

| Scope | Duration | Use Case |
|-------|----------|----------|
| **TURN** | One turn only | Brief pulse for immediate effect |
| **GOAL** | Until goal changes | Suppress tool during a specific task |
| **SESSION** | Until session ends | Amplify preferred model for this session |
| **PERMANENT** | Until explicitly removed | Enterprise compliance requirements |
| **CONDITIONAL** | Until condition is met | Conditional closed-loop control |

### Conflict Resolution

When multiple modulations target the same component, the conflict resolver applies strict priority:

```
CLAMP > enterprise_policy > max(priority) > most_recent
```

```python
# CLAMP always wins
modulator.clamp("tool_x", value=0.5)
modulator.amplify("tool_x", factor=2.0)
# Effective value: 0.5 (CLAMP overrides AMPLIFY)
```

### Enterprise Modulation Policies

Enterprise policies use SHA-256 integrity hashes to prevent tampering:

```python
from corteX.engine.modulator import EnterpriseModulationPolicy

policy = EnterpriseModulationPolicy(
    name="hipaa_safety",
    targets=["safety_strictness", "pii_detection"],
    modulation_type=ModulationType.CLAMP,
    value=1.0,
    reason="HIPAA compliance",
)

modulator.add_policy(policy)

# Policies are integrity-verified on every evaluation
# Tampered policies are rejected and logged
```

### Audit Trail

Every modulation operation is logged with full audit trail:

```python
trail = modulator.get_audit_trail()
# Each entry: event type, timestamp, turn number,
#             modulation_id, target, details, actor
```

Event types logged:
- `policy_added` / `policy_removed`
- `policy_activated` / `policy_evaluated`
- `policy_integrity_violation` / `policy_rejected_tampered`

### Conditional Modulation

The `ConditionalModulator` implements closed-loop control -- modulations that activate or deactivate based on runtime conditions:

```python
from corteX.engine.modulator import ConditionalModulator

conditional = ConditionalModulator()

# Dampen verbosity when frustration exceeds threshold
conditional.add_rule(
    target="verbosity",
    condition=lambda ctx: ctx.get("frustration_level", 0) > 0.7,
    modulation_type=ModulationType.DAMPEN,
    factor=0.5,
)
```

## When It Activates

- **Before every decision**: Active modulations are applied to weight lookups
- **On policy evaluation**: Enterprise policies are checked (with integrity verification)
- **On scope expiry**: TURN-scoped modulations expire after one turn; GOAL-scoped on goal change
- **On admin action**: Enterprise policies can be added/removed at runtime

## API Reference

```python
from corteX.engine.modulator import (
    TargetedModulator,
    Modulation,
    ModulationType,
    ModulationScope,
    EnterpriseModulationPolicy,
    ModulationConflictResolver,
    ConditionalModulator,
)
```
