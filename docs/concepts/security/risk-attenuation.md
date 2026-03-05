# Risk-Based Attenuation

As risk increases, agent capabilities automatically decrease. The
Risk Attenuator dynamically narrows what an agent can do based on a
computed risk score, without requiring any developer intervention.
This is a novel mechanism that connects risk assessment directly to
the capability system.

## How It Works

The `RiskAttenuator` computes a composite risk score (0.0 to 1.0)
from multiple factors, then maps it to one of four risk levels that
determine which actions are stripped:

| Risk Level | Score Range | Actions Allowed |
|------------|-------------|-----------------|
| LOW        | 0.0 - 0.3   | Full capabilities (no reduction) |
| MEDIUM     | 0.3 - 0.6   | Read + Execute (write/delete/admin removed) |
| HIGH       | 0.6 - 0.8   | Read only (execute also removed) |
| CRITICAL   | 0.8 - 1.0   | Read only + human approval required |

The composite risk score is a weighted blend of four factors:

- **Tool type risk (35%)** - inherent risk of the tool (e.g., shell=0.9, search=0.1)
- **Data classification (25%)** - sensitivity level of the data involved
- **Agent confidence (20%)** - lower LLM confidence increases risk
- **Goal drift (20%)** - higher drift from original goal increases risk

## Key Features

- **Automatic capability reduction** - no developer code needed; risk maps directly to permissions
- **Four-factor risk computation** - tool type, data level, confidence, and drift
- **Built-in tool risk catalog** - pre-configured risk scores for common tools (shell, browser, database, etc.)
- **Human-in-the-loop** - CRITICAL risk level sets `requires_approval=True`
- **Detailed result tracking** - `AttenuationResult` lists every removed action
- **Drift-aware** - goal drift above 0.5 exponentially increases risk contribution

## Integration

The Risk Attenuator bridges the brain engine and the capability system.
During each agent turn, the orchestrator calls `compute_risk()` with
the current action and context (data level, confidence, drift score),
then passes the result to `attenuate_by_risk()` which returns a
narrowed `CapabilitySet`. The agent proceeds with the reduced
permissions for that turn.

The Data Classifier feeds data sensitivity levels into the risk
computation. The Goal Tracker provides drift scores. The brain
engine supplies confidence values. All four signals converge in the
attenuator to produce a single, actionable risk assessment.

## Usage Example

```python
from corteX.security.attenuation import RiskAttenuator, RiskLevel
from corteX.security.capabilities import Capability, CapabilitySet

attenuator = RiskAttenuator()

# Compute risk from context
risk_score = attenuator.compute_risk(
    action="shell",
    context={
        "data_level": "confidential",
        "confidence": 0.6,
        "drift_score": 0.3,
    },
)
# risk_score will be high due to shell + confidential data

# Attenuate capabilities based on risk
caps = CapabilitySet([
    Capability("tool:shell", frozenset({"read", "write", "execute"})),
    Capability("memory:long_term", frozenset({"read", "write"})),
])

result = attenuator.attenuate_by_risk(caps, risk=risk_score)

print(result.risk_level)        # RiskLevel.HIGH
print(result.removed_actions)   # ["tool:shell:execute", "tool:shell:write", ...]
print(result.requires_approval) # True if CRITICAL

# Use the attenuated capabilities for this turn
safe_caps = result.attenuated_caps
```

## See Also

- [Capability-Based Security](capabilities.md) - the permission model that gets attenuated
- [Data Classification](data-classification.md) - feeds data sensitivity into risk computation
- [Compliance Engine](compliance-engine.md) - regulatory constraints that interact with risk
