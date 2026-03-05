# Compliance Engine

The Compliance Engine evaluates every agent action against active
regulatory frameworks before execution. It supports GDPR, HIPAA,
SOC 2, and ISO 27001, producing a `ComplianceResult` that tells the
runtime whether to proceed, log, or block the action.

## How It Works

The `ComplianceEngine` is initialized with one or more
`ComplianceFramework` values. Before every action, the orchestrator
calls `pre_check()` with the action name, data classification level,
and optional context. The engine dispatches to framework-specific
handlers and aggregates the results.

Each handler returns a list of `ComplianceAction` objects with a type:

| Action Type | Meaning |
|-------------|---------|
| `LOG` | Action must be logged for audit trail |
| `BLOCK` | Action is prohibited - stops execution |
| `WARN` | Advisory warning, non-blocking |
| `ENCRYPT` | Data must be encrypted before proceeding |
| `ANONYMIZE` | Data must be anonymized/minimized |
| `APPROVAL_REQUIRED` | Human approval needed before proceeding |
| `RETENTION_CHECK` | Data retention period exceeds limits |

If any `BLOCK` action has `required=True`, the overall result is
`allowed=False` and the violations list contains the specific reasons.

## Key Features

- **Four frameworks** - GDPR, HIPAA, SOC 2, ISO 27001 with framework-specific rules
- **Pre-execution checks** - every action is validated before it runs
- **Consent auto-resolution** - when GDPR is active with a `ConsentManager`, consent status is looked up automatically from tenant/user/purpose context
- **Data-level enforcement** - CONFIDENTIAL and RESTRICTED data trigger mandatory audit and cloud-blocking rules
- **GDPR specifics** - consent checks (Art. 6), data minimization (Art. 5), retention limits (Art. 5), right to erasure (Art. 17)
- **HIPAA specifics** - PHI encryption (164.312), access logging (164.312), BAA requirement for cloud (164.314)
- **SOC 2 specifics** - access controls (CC6.1), security event logging (CC7.2), config change approval (CC8.1)
- **ISO 27001 specifics** - activity logging (A.12.4), risk assessment warnings (6.1.2)

## Integration

The Compliance Engine works alongside the Data Classifier, which
provides the `DataLevel` input for compliance checks. When GDPR is
among the active frameworks and no `ConsentManager` is provided, the
engine auto-creates one from the enterprise consent module (if
available).

The result feeds into the orchestrator's decision loop: if `allowed`
is `False`, the action is blocked and the violations are reported.
Required actions like `LOG` and `ENCRYPT` are enforced by the runtime
before proceeding.

## Usage Example

```python
from corteX.security.compliance import (
    ComplianceEngine, ComplianceFramework, ActionType,
)
from corteX.security.classification import DataLevel

# Initialize with active frameworks
engine = ComplianceEngine(
    frameworks=[
        ComplianceFramework.GDPR,
        ComplianceFramework.SOC2,
    ],
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
    # Proceed, but fulfill required actions first
    for action in result.actions_required:
        if action.action_type == ActionType.LOG:
            print(f"Must log: {action.description}")
        elif action.action_type == ActionType.ENCRYPT:
            print(f"Must encrypt: {action.description}")
else:
    # Action blocked
    for violation in result.violations:
        print(f"Violation: {violation}")

# HIPAA with cloud model check
hipaa_engine = ComplianceEngine(
    frameworks=[ComplianceFramework.HIPAA],
)
result = hipaa_engine.pre_check(
    action="analyze_records",
    data_level=DataLevel.RESTRICTED,
    context={"is_local": False, "has_baa": False},
)
assert not result.allowed  # PHI cannot go to cloud without BAA
```

## See Also

- [Data Classification](data-classification.md) - provides data sensitivity levels
- [Capability-Based Security](capabilities.md) - permission model enforced alongside compliance
- [Risk-Based Attenuation](risk-attenuation.md) - dynamic permission reduction
- [Tenant Audit Stream](../observability/audit-stream.md) - records compliance events
