# Compliance Engine API Reference

## Module: `corteX.security.compliance`

Compliance as code -- machine-readable compliance policy enforcement. Supports GDPR, HIPAA, SOC2, and ISO 27001 frameworks. Policies are evaluated before every action, producing a `ComplianceResult` that tells the runtime whether to proceed, log, or block.

## Classes

### `ComplianceFramework`

**Type**: `str, Enum`

Supported compliance frameworks.

| Value | Description |
|-------|-------------|
| `GDPR` | EU General Data Protection Regulation |
| `HIPAA` | US Health Insurance Portability and Accountability Act |
| `SOC2` | Service Organization Control 2 |
| `ISO27001` | ISO/IEC 27001 Information Security |

---

### `ActionType`

**Type**: `str, Enum`

Types of compliance-mandated actions.

| Value | Description |
|-------|-------------|
| `LOG` | Action must be logged |
| `BLOCK` | Action is blocked |
| `WARN` | Warning issued but not blocked |
| `ENCRYPT` | Data must be encrypted |
| `ANONYMIZE` | Data must be anonymized |
| `APPROVAL_REQUIRED` | Human approval needed |
| `RETENTION_CHECK` | Data retention limits apply |

---

### `ComplianceAction`

**Type**: `@dataclass`

A single compliance-mandated action.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `framework` | `ComplianceFramework` | Which framework mandates this action |
| `action_type` | `ActionType` | Type of action required |
| `description` | `str` | Human-readable description with regulatory reference |
| `required` | `bool` | Whether this action is mandatory (default `True`) |

---

### `ComplianceResult`

**Type**: `@dataclass`

Outcome of a compliance pre-check.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `allowed` | `bool` | Whether the action may proceed |
| `violations` | `List[str]` | List of violation descriptions (blocking actions) |
| `actions_required` | `List[ComplianceAction]` | All compliance actions that apply |

---

### `ComplianceEngine`

Evaluate actions against active compliance frameworks.

#### Constructor

```python
ComplianceEngine(
    frameworks: Optional[List[ComplianceFramework]] = None
)
```

**Parameters**:

- `frameworks` (`Optional[List[ComplianceFramework]]`): Active frameworks. Empty list means no compliance checks.

#### Methods

##### `pre_check`

```python
def pre_check(
    self, action: str, data_level: DataLevel,
    context: Optional[Dict[str, Any]] = None
) -> ComplianceResult
```

Check an action against all active compliance frameworks.

**Parameters**:

- `action` (`str`): The action being performed.
- `data_level` (`DataLevel`): Classification level of the data.
- `context` (`Optional[Dict[str, Any]]`): Context dict with framework-specific keys.

**Returns**: `ComplianceResult` -- `allowed=True` if no blocking violations.

##### `enforce_gdpr`

```python
def enforce_gdpr(self, context: Dict[str, Any]) -> List[ComplianceAction]
```

GDPR enforcement: data minimization, retention limits, erasure support.

**Context keys**:

| Key | Type | Default | GDPR Article |
|-----|------|---------|-------------|
| `has_consent` | `bool` | `True` | Art. 6 -- Lawful basis |
| `retention_days` | `int` | `90` | Art. 5(1)(e) -- Storage limitation |

**Actions generated**:

- **BLOCK** if `has_consent=False` (Art. 6)
- **ANONYMIZE** (optional) for data minimization (Art. 5(1)(c))
- **RETENTION_CHECK** if retention > 365 days (Art. 5(1)(e))
- **LOG** for right to erasure support (Art. 17)

##### `enforce_hipaa`

```python
def enforce_hipaa(self, context: Dict[str, Any]) -> List[ComplianceAction]
```

HIPAA enforcement: PHI isolation, access logging, encryption.

**Context keys**:

| Key | Type | Default | HIPAA Section |
|-----|------|---------|-------------|
| `encrypted` | `bool` | `True` | 164.312(a)(2)(iv) |
| `is_local` | `bool` | `True` | 164.314 |
| `has_baa` | `bool` | `False` | 164.314 -- Business Associate Agreement |

##### `enforce_soc2`

```python
def enforce_soc2(self, context: Dict[str, Any]) -> List[ComplianceAction]
```

SOC2 enforcement: access controls, monitoring, change approval.

**Context keys**:

| Key | Type | Default | SOC2 Section |
|-----|------|---------|-------------|
| `has_access_control` | `bool` | `True` | CC6.1 |
| `is_config_change` | `bool` | `False` | CC8.1 |

##### `get_active_frameworks`

```python
def get_active_frameworks(self) -> List[ComplianceFramework]
```

Return the list of currently active compliance frameworks.

---

## Cross-Framework Data Level Enforcement

In addition to framework-specific rules, data level rules apply across all frameworks:

| Data Level | Action | Condition |
|------------|--------|-----------|
| `CONFIDENTIAL`+ | LOG | Always (audit required) |
| `RESTRICTED` | BLOCK | If `is_local=False` (no cloud processing) |

---

## Example

```python
from corteX.security.compliance import ComplianceEngine, ComplianceFramework
from corteX.security.classification import DataLevel

engine = ComplianceEngine(
    frameworks=[ComplianceFramework.GDPR, ComplianceFramework.SOC2]
)

# Pre-check an action
result = engine.pre_check(
    action="send_email",
    data_level=DataLevel.CONFIDENTIAL,
    context={
        "has_consent": True,
        "retention_days": 90,
        "has_access_control": True,
    },
)

if result.allowed:
    print("Action allowed")
    for action in result.actions_required:
        print(f"  {action.framework.value}: {action.action_type.value} -- {action.description}")
else:
    print(f"BLOCKED: {result.violations}")

# GDPR without consent
result = engine.pre_check(
    action="process_user_data",
    data_level=DataLevel.CONFIDENTIAL,
    context={"has_consent": False},
)
assert not result.allowed
assert "GDPR Art. 6" in result.violations[0]

# HIPAA with cloud model
hipaa_engine = ComplianceEngine([ComplianceFramework.HIPAA])
result = hipaa_engine.pre_check(
    action="analyze_records",
    data_level=DataLevel.RESTRICTED,
    context={"is_local": False, "has_baa": False},
)
assert not result.allowed  # PHI cannot go to cloud without BAA
```

---

## See Also

- [Data Classifier](./classification.md) -- Data level classification
- [Capability Set](./capabilities.md) -- Permission enforcement
- [Risk Attenuator](./attenuation.md) -- Risk-based capability reduction
- [Audit Stream](../observability/audit-stream.md) -- Tamper-evident audit logging
