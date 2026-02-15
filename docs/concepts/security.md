# Security Framework

corteX provides a defense-in-depth security framework that protects API keys, enforces least-privilege access, classifies data sensitivity, and ensures compliance with enterprise regulations. All security modules run entirely in-process with zero external dependencies.

## What It Does

The security framework provides five layered capabilities:

1. **KeyVault**: Per-tenant in-memory API key management with obfuscation
2. **CapabilitySet**: Immutable, attenuatable permission tokens for fine-grained access control
3. **RiskAttenuator**: Automatic capability reduction as risk increases
4. **DataClassifier**: Real-time PII detection and data sensitivity enforcement
5. **ComplianceEngine**: Machine-readable policy enforcement for GDPR, HIPAA, SOC2, and ISO 27001

Together, these components enforce security at every layer of agent execution -- from key storage to data flow to regulatory compliance.

---

## KeyVault

**Per-tenant secret management that keeps API keys out of plaintext memory.**

The KeyVault stores LLM provider keys in memory using obfuscation, ensuring they never appear as plaintext strings in process memory, disk, or logs. Each tenant has an isolated vault with atomic key rotation.

### Why It Matters

API keys are the most common source of security incidents in AI applications. Developers accidentally log keys, leave them in environment variables, or share them across tenants. KeyVault enforces strict discipline:

- Keys never touch disk or appear in logs
- Each tenant's keys are isolated -- no cross-tenant access
- Key rotation is atomic: old keys are zeroed before new keys are stored
- No environment variable fallback -- missing keys raise explicit errors

### How to Use It

```python
import cortex

engine = cortex.Engine()

# Store keys per tenant
engine.security.vault.store_key(
    tenant_id="acme_corp",
    provider="openai",
    api_key="sk-..."
)

# Keys are retrieved internally during LLM calls
# You never need to handle raw keys in application code

# Rotate a key atomically
engine.security.vault.rotate_key(
    tenant_id="acme_corp",
    provider="openai",
    new_key="sk-new-..."
)
# Old key is zeroed before new key is stored
```

---

## Capability-Based Security

**Immutable permission tokens that can only shrink, never grow.**

Instead of traditional role-based access (admin, user, viewer), corteX uses capability sets -- frozen permission tokens that define exactly what an agent can do. Capabilities can be attenuated (reduced) but never escalated.

### Why It Matters

Role-based access is too coarse for AI agents. An agent that needs to read a database should not automatically get permission to write to it. Capability sets enforce the **principle of least privilege** at a granular level:

- An agent created with `read + write + execute` capabilities
- ...can be attenuated to `read + execute` for a sub-task
- ...and further attenuated to `read` only when risk is high
- But it can **never** gain capabilities it was not created with

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="data-analyst",
    system_prompt="You analyze customer data.",
    capabilities=["read_database", "write_reports", "execute_queries"]
)

session = agent.start_session(user_id="analyst_1")

# When the agent spawns a sub-agent, capabilities are automatically
# attenuated -- the sub-agent receives fewer permissions
sub_result = await session.run_sub_agent(
    task="Summarize Q4 revenue",
    # Sub-agent automatically gets read-only capabilities
)
```

### Capability Attenuation

Sub-agents always receive attenuated capability sets:

| Parent Capability | Sub-Agent Receives | Rationale |
|---|---|---|
| `read + write + execute` | `read + execute` | Write stripped by default |
| `read + execute` | `read` | Execute stripped for deeper nesting |
| `read` | `read` | Cannot attenuate further |

---

## Risk-Based Attenuation

**As risk increases, agent capabilities automatically decrease.**

The RiskAttenuator monitors the current risk level of agent actions and dynamically reduces capabilities. When an agent encounters sensitive data, makes expensive API calls, or operates in an unfamiliar domain, its permissions shrink automatically.

### Risk Levels

| Risk Level | Range | Effect |
|---|---|---|
| **Low** | 0.0 -- 0.3 | Full capabilities |
| **Medium** | 0.3 -- 0.6 | Write access removed |
| **High** | 0.6 -- 0.8 | Execute access removed, read only |
| **Critical** | 0.8 -- 1.0 | Read only + human approval required |

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="support",
    system_prompt="Help customers with their accounts.",
    risk_attenuation=True  # Enable automatic risk-based attenuation
)

session = agent.start_session(user_id="support_1")

# Normal request -- low risk, full capabilities
response = await session.run("Show order history for customer 123")

# Sensitive request -- risk increases, capabilities shrink
response = await session.run("Delete customer 123's account")
# Agent's write capability is automatically removed
# Response: "I need human approval before deleting accounts."
```

### How Risk Is Calculated

Risk scores combine multiple signals:

- **Data sensitivity**: PII or confidential data increases risk
- **Action severity**: Delete/modify operations score higher than reads
- **Domain familiarity**: Novel task types increase risk
- **Error rate**: Recent failures increase risk

The agent does not need to be told to "be careful" -- risk attenuation is automatic and continuous.

---

## Data Classification

**Automatic PII detection and data-level enforcement throughout the pipeline.**

The DataClassifier scans every piece of data flowing through corteX and assigns it one of four sensitivity levels. Classification can only escalate (become more sensitive), never downgrade.

### Four Data Levels

| Level | Description | Allowed Destinations |
|---|---|---|
| **Public** | No sensitive content | Any model (cloud or local) |
| **Internal** | Business data, no PII | On-prem models only |
| **Confidential** | Contains PII or financial data | On-prem models + audit required |
| **Restricted** | Highly sensitive (medical, legal) | On-prem + approval + full audit trail |

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="hr-assistant",
    system_prompt="Help HR team with employee queries.",
    data_classification=True  # Enable automatic classification
)

session = agent.start_session(user_id="hr_1")

# Public data -- can use any model
response = await session.run("What are our company holidays?")

# Confidential data detected automatically
response = await session.run(
    "Look up salary for employee John Smith, SSN 123-45-6789"
)
# DataClassifier detects SSN pattern -> escalates to CONFIDENTIAL
# Request is automatically routed to on-prem model
# Audit log entry is created
```

### PII Detection

The classifier detects common PII patterns without sending data to external services:

- Social Security Numbers (SSN)
- Credit card numbers
- Email addresses
- Phone numbers
- Medical record identifiers
- Financial account numbers

All detection runs locally using pattern matching -- no data leaves your infrastructure.

---

## Compliance Engine

**Machine-readable compliance policies evaluated before every agent action.**

The ComplianceEngine evaluates regulatory policies before the agent executes any action, producing a clear proceed/log/block decision. It supports four major compliance frameworks out of the box.

### Supported Frameworks

| Framework | Focus Area | Key Requirements |
|---|---|---|
| **GDPR** | Data protection (EU) | Consent tracking, data minimization, right to erasure |
| **HIPAA** | Healthcare (US) | PHI protection, access controls, audit trails |
| **SOC2** | Service organizations | Security controls, availability, processing integrity |
| **ISO 27001** | Information security | Risk management, access control, incident response |

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="medical-assistant",
    system_prompt="Help clinicians with patient queries.",
    compliance_frameworks=["hipaa", "soc2"]
)

session = agent.start_session(
    user_id="dr_smith",
    tenant_id="hospital_a"
)

# Every action is checked against compliance policies
response = await session.run("Show patient records for room 302")
# ComplianceEngine checks:
#   1. HIPAA: Is the user authorized to access PHI?
#   2. SOC2: Is the audit trail enabled?
#   3. Data classification: Is the response marked RESTRICTED?
# If all checks pass -> proceed with full audit logging
# If any check fails -> block with explanation
```

### Policy Evaluation Flow

For every agent action, the compliance engine:

1. Identifies the data classification level of inputs and outputs
2. Checks the action against all enabled framework policies
3. Produces a `ComplianceResult` with one of three decisions:
   - **Proceed**: Action is compliant, execute normally
   - **Log**: Action is compliant but requires audit entry
   - **Block**: Action violates a policy, do not execute

---

## Integration

All security modules integrate automatically when configured:

```python
import cortex

engine = cortex.Engine(
    security={
        "key_vault": True,
        "capabilities": True,
        "risk_attenuation": True,
        "data_classification": True,
        "compliance_frameworks": ["gdpr", "soc2"]
    }
)

# Security is now enforced at every layer:
# 1. KeyVault manages all provider keys
# 2. CapabilitySet controls agent permissions
# 3. RiskAttenuator adjusts permissions based on context
# 4. DataClassifier scans all data flow
# 5. ComplianceEngine validates every action
```

---

## See Also

- [Security & Isolation](../enterprise/security.md) -- Zero-trust network model and tenant isolation
- [Audit Logging](../enterprise/audit.md) -- Enterprise audit trail configuration
- [Compliance](../enterprise/compliance.md) -- Detailed compliance framework setup
- [Observability](observability.md) -- Monitoring and tracing for security events
