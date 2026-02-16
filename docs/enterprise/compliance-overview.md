# Compliance Overview

corteX provides comprehensive, code-level compliance enforcement for SOC 2 and GDPR -- the two frameworks most commonly required by enterprise SaaS customers. Rather than treating compliance as a checklist, corteX embeds compliance controls directly into the agent runtime so violations are impossible by construction.

## Architecture

```
                    +-------------------+
                    |   Application     |
                    +--------+----------+
                             |
                    +--------v----------+
                    |   corteX Session   |
                    +--------+----------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v-----+  +----v-------+  +---v---------+
     | Consent Mgr  |  | Compliance |  | Data        |
     | (Art. 7)     |  | Engine     |  | Residency   |
     +--------------+  +------------+  | (Art. 44)   |
              |              |         +-------------+
     +--------v-----+  +----v-------+
     | Profiling    |  | Retention  |
     | Opt-Out      |  | Enforcer   |
     | (Art. 22)    |  | (TTL/Purge)|
     +--------------+  +------------+
              |              |
     +--------v-----+  +----v-------+
     | GDPR Manager |  | Audit      |
     | (DSAR Arts   |  | Logger     |
     |  13-22)      |  | (SOC 2)    |
     +--------------+  +------------+
              |              |
     +--------v-----+  +----v-------+
     | Explainability| | PII        |
     | Engine       |  | Tokenizer  |
     | (Art. 15/22) |  +------------+
     +--------------+       |
                        +----v-------+
                        | Tenant Key |
                        | Manager    |
                        | (Art. 32)  |
                        +------------+
```

## SOC 2 Compliance

SOC 2 (Service Organization Control 2) focuses on five trust service criteria: security, availability, processing integrity, confidentiality, and privacy. corteX addresses these through several integrated modules.

### Tamper-Evident Audit Logging

The `AuditLogger` provides hash-chained (SHA-256) audit entries that satisfy SOC 2 CC4.1/CC4.2 (monitoring activities) and CC7.1/CC7.2 (system operations monitoring). Every LLM call, tool execution, weight change, security block, and policy decision is logged with a tamper-evident chain.

```python
from corteX.enterprise.config import AuditConfig
from corteX.security.audit_logger import AuditLogger

logger = AuditLogger(
    config=AuditConfig(
        enabled=True,
        log_tool_calls=True,
        log_model_routing=True,
        log_weight_changes=True,
        log_path="/var/log/cortex/audit",
        retention_days=365,  # SOC 2 requires 1 year
    ),
    tenant_id="acme",
)

# Verify log integrity at any time
if not logger.verify_integrity():
    report = logger.verify_integrity_detailed()
    raise SecurityError(f"Audit tampering at entry {report['break_index']}")
```

### Data Retention Enforcement

The `RetentionEnforcer` automatically purges data past its TTL based on compliance framework presets. SOC 2 preset requires 365-day retention for audit logs.

```python
from corteX.enterprise.retention import RetentionEnforcer

enforcer = RetentionEnforcer(config=tenant_config)
report = await enforcer.enforce()
```

### Per-Tenant Encryption

The `TenantKeyManager` derives unique AES-256 encryption keys per tenant using HKDF-SHA256, satisfying SOC 2 CC6.1 (logical and physical access controls).

```python
from corteX.security.tenant_encryption import TenantKeyManager

km = TenantKeyManager(master_key=master_key)
ciphertext = km.encrypt("acme", sensitive_data)
```

---

## GDPR Compliance

The General Data Protection Regulation (GDPR) requires specific data subject rights and processing controls. corteX implements these as first-class APIs.

### Consent Management (Art. 7)

The `ConsentManager` provides granular, per-purpose consent tracking with full audit trail. Processing is automatically blocked when consent is missing.

```python
from corteX.enterprise.consent import ConsentManager

consent = ConsentManager()

# Record granular consent
consent.record_consent(
    tenant_id="acme", user_id="user_42",
    purpose="personalization",
    mechanism="explicit_opt_in",
    policy_version="2.1",
)

# Check before any processing
if not consent.check_consent("acme", "user_42", "personalization"):
    raise ConsentRequired("No consent for personalization")
```

### Data Subject Access Requests (Arts. 13-22)

The `GDPRManager` handles the full DSAR lifecycle:

| Right | Article | Method |
|-------|---------|--------|
| Right of Access | Art. 15 | `export_user_data()` |
| Right to Rectification | Art. 16 | `rectify_user_data()` |
| Right to Erasure | Art. 17 | `erase_user_data()` |
| Right to Restriction | Art. 18 | `restrict_processing()` |
| Right to Portability | Art. 20 | `export_portable_data()` |
| Right to Object | Art. 21 | `register_objection()` |
| Right to Transparency | Arts. 13-14 | `get_processing_info()` |

```python
from corteX.enterprise.gdpr import GDPRManager

gdpr = GDPRManager(memory=fabric, weights=engine, audit=logger)

# Art. 17 -- Right to Erasure (cascade across all stores)
result = await gdpr.erase_user_data("acme", "user_42", scope="all")
# Deletes from: conversations, all memory tiers, cold storage,
# weight deltas, preferences, audit logs
```

### Profiling Opt-Out (Art. 22)

The `ProfilingManager` lets users opt out of automated profiling. When opted out, synaptic weight personalization and behavioral pattern learning are automatically disabled.

```python
from corteX.enterprise.profiling import ProfilingManager

profiling = ProfilingManager()
profiling.opt_out("acme", "user_42", reason="User preference")

# Guard in agent loop
if not profiling.is_profiling_allowed("acme", "user_42"):
    # Skip weight personalization, use generic weights
    pass
```

### Decision Explainability (Arts. 13-15, 22)

The `ExplainabilityEngine` generates human-readable explanations for every agent decision, satisfying the GDPR requirement to explain automated decision-making.

```python
from corteX.enterprise.explainability import ExplainabilityEngine

engine = ExplainabilityEngine(tracer=tracer)
explanation = engine.explain_decision("sess_abc", step_index=3)
print(explanation.human_readable)
print(explanation.gdpr_disclosure)
```

### Data Residency (Arts. 44-49)

The `DataResidencyManager` ensures personal data stays within configured geographic regions by validating LLM provider endpoints.

```python
from corteX.enterprise.data_residency import DataResidencyManager

drm = DataResidencyManager()
drm.set_allowed_regions("acme_eu", ["eu", "uk"])

# Blocks US-based providers for EU tenants
check = drm.validate_provider("acme_eu", "https://api.openai.com/v1")
assert not check.allowed  # OpenAI is US-based
```

### PII Protection

The `PIITokenizer` replaces PII with reversible tokens before sending text to LLMs, ensuring personal data never leaves the enterprise boundary.

```python
from corteX.security.pii_tokenizer import PIITokenizer

tokenizer = PIITokenizer()
result = tokenizer.tokenize("Contact user@test.com", tenant_id="acme")
# result.text == "Contact [PII_EMAIL_001]"
# Send to LLM, then restore:
restored = tokenizer.detokenize(llm_response, tenant_id="acme")
```

---

## Putting It All Together

A fully compliant enterprise configuration combines all modules:

```python
from corteX.enterprise.config import (
    TenantConfig, ComplianceFramework, DataRetention,
    AuditConfig, SafetyPolicy, SafetyLevel,
)
from corteX.enterprise.consent import ConsentManager
from corteX.enterprise.gdpr import GDPRManager
from corteX.enterprise.profiling import ProfilingManager
from corteX.enterprise.retention import RetentionEnforcer
from corteX.enterprise.explainability import ExplainabilityEngine
from corteX.enterprise.data_residency import DataResidencyManager
from corteX.security.audit_logger import AuditLogger
from corteX.security.pii_tokenizer import PIITokenizer
from corteX.security.tenant_encryption import TenantKeyManager

# 1. Tenant configuration
config = TenantConfig(
    tenant_id="acme_eu",
    compliance=[ComplianceFramework.GDPR, ComplianceFramework.SOC2],
    data_retention=DataRetention.PERSISTENT,
    audit=AuditConfig(
        enabled=True,
        log_tool_calls=True,
        log_model_routing=True,
        retention_days=365,
    ),
    safety=SafetyPolicy(level=SafetyLevel.STRICT),
)

# 2. Initialize compliance stack
audit = AuditLogger(config=config.audit, tenant_id="acme_eu")
consent = ConsentManager()
gdpr = GDPRManager(audit=audit)
profiling = ProfilingManager()
retention = RetentionEnforcer(config=config, audit=audit)
explainability = ExplainabilityEngine()
residency = DataResidencyManager()
pii = PIITokenizer()
encryption = TenantKeyManager(master_key=master_key)

# 3. Configure data residency (EU only)
residency.set_allowed_regions("acme_eu", ["eu", "uk"])

# 4. The agent runtime checks these before every action:
#    - consent.check_consent() before personalization
#    - profiling.is_profiling_allowed() before weight adaptation
#    - residency.validate_provider() before LLM calls
#    - pii.tokenize() before sending text to LLM
#    - audit.log_event() for every action
#    - retention.enforce() on schedule
```

---

## Module Reference

| Module | Purpose | Framework |
|--------|---------|-----------|
| [Consent Manager](../reference/enterprise/consent.md) | Per-purpose consent lifecycle | GDPR Art. 7 |
| [GDPR Manager](../reference/enterprise/gdpr.md) | DSAR lifecycle (Arts. 13-22) | GDPR |
| [Retention Manager](../reference/enterprise/retention.md) | Automated data purging | GDPR + SOC 2 |
| [Profiling Manager](../reference/enterprise/profiling.md) | Profiling opt-out | GDPR Art. 22 |
| [Explainability Engine](../reference/enterprise/explainability.md) | Decision explanations | GDPR Arts. 13-15, 22 |
| [Data Residency](../reference/enterprise/data-residency.md) | Geographic data controls | GDPR Arts. 44-49 |
| [Audit Logger](../reference/security/audit-logger.md) | Tamper-evident logging | SOC 2 CC4/CC7 |
| [PII Tokenizer](../reference/security/pii-tokenizer.md) | Reversible PII redaction | GDPR Art. 25 |
| [Tenant Encryption](../reference/security/tenant-encryption.md) | Per-tenant AES-256 keys | GDPR Art. 32, SOC 2 CC6.1 |
| [Compliance Engine](../reference/security/compliance.md) | Pre-action policy checks | All frameworks |
