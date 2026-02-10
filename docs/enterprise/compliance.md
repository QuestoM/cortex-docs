# Compliance

corteX provides built-in awareness of major compliance frameworks. This page describes how to configure corteX for each framework and what the SDK enforces automatically.

## Supported Frameworks

```python
from corteX.enterprise.config import ComplianceFramework

config.compliance = [
    ComplianceFramework.SOC2,
    ComplianceFramework.GDPR,
    ComplianceFramework.HIPAA,
    ComplianceFramework.ISO27001,
    ComplianceFramework.PCI_DSS,
    ComplianceFramework.CCPA,
    ComplianceFramework.FedRAMP,
]
```

## SOC 2

SOC 2 compliance focuses on security, availability, processing integrity, confidentiality, and privacy.

**corteX controls:**
- Enable audit logging (`AuditConfig.enabled=True`)
- Log all tool calls and model routing decisions
- Set `DataRetention.AUDIT_ONLY` to retain compliance evidence without storing user data
- Use `SafetyLevel.STRICT` or higher
- Configure `retention_days >= 365`

```python
config = TenantConfig(
    tenant_id="acme",
    compliance=[ComplianceFramework.SOC2],
    audit=AuditConfig(enabled=True, log_tool_calls=True, retention_days=365),
    data_retention=DataRetention.AUDIT_ONLY,
    safety=SafetyPolicy(level=SafetyLevel.STRICT),
)
```

## GDPR

The General Data Protection Regulation requires data minimization, right to erasure, and lawful processing.

**corteX controls:**
- Set `DataRetention.SESSION` or `DataRetention.NONE` to avoid persisting personal data
- Enable `pii_detection=True` for automatic PII masking
- Disable `log_messages=False` in audit config to avoid logging conversation content
- Implement erasure by clearing tenant data (file-based storage makes this straightforward)

## HIPAA

HIPAA governs protected health information (PHI) in healthcare contexts.

**corteX controls:**
- Set `SafetyLevel.LOCKED` to prevent any user overrides
- Enable full audit logging with `retention_days >= 2190` (6 years)
- Enable PII detection and content filtering
- Restrict tools to a vetted allowlist
- Use on-prem deployment with local models to keep PHI within the organization boundary

```python
config = TenantConfig(
    tenant_id="hospital",
    compliance=[ComplianceFramework.HIPAA],
    safety=SafetyPolicy(
        level=SafetyLevel.LOCKED,
        pii_detection=True,
        content_filtering=True,
    ),
    audit=AuditConfig(
        enabled=True,
        log_tool_calls=True,
        log_messages=True,  # Required for PHI access audit
        retention_days=2190,
    ),
    tools=ToolPolicy(
        allowed_tools=["medical_lookup", "patient_chart"],
        max_tool_calls_per_turn=5,
    ),
)
```

## ISO 27001

ISO 27001 focuses on information security management systems (ISMS).

**corteX controls:**
- Enable comprehensive audit logging
- Use enterprise modulation policies with integrity verification (SHA-256 tamper detection)
- Configure model and tool allowlists
- Implement access controls via per-seat licensing

## PCI DSS

PCI DSS governs payment card data handling.

**corteX controls:**
- Block patterns matching card numbers via `blocked_patterns`
- Enable PII detection
- Restrict data retention to `SESSION` or `NONE`
- Audit all tool calls involving data access

## Feature Requirements

Compliance features require the appropriate license plan:

| Feature | Starter | Professional | Enterprise | Unlimited |
|---------|:-------:|:----------:|:----------:|:---------:|
| Basic audit logging | | | x | x |
| Compliance framework enforcement | | | x | x |
| Custom safety policies | | x | x | x |
| Enterprise modulation policies | | | x | x |
