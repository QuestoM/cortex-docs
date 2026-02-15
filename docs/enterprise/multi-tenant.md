# Multi-Tenant Configuration

Every corteX deployment is governed by a `TenantConfig` -- a single dataclass that encapsulates all policies for one customer or organization. This page explains how tenant isolation works and how to configure it.

## TenantConfig

The `TenantConfig` is the root configuration object. It holds sub-policies for safety, models, tools, audit, licensing, compliance, and data retention.

```python
from corteX.enterprise.config import (
    TenantConfig, SafetyPolicy, ModelPolicy, ToolPolicy,
    AuditConfig, LicenseConfig, SafetyLevel, DataRetention,
    ComplianceFramework,
)

config = TenantConfig(
    tenant_id="acme_corp",
    tenant_name="Acme Corporation",
    safety=SafetyPolicy(level=SafetyLevel.STRICT),
    models=ModelPolicy(
        allowed_models=["claude-opus-4-6", "gemini-3-flash-preview"],
        max_tokens_per_request=32000,
        max_tokens_per_session=500000,
    ),
    tools=ToolPolicy(
        blocked_tools=["bash_unrestricted"],
        require_approval_tools=["file_delete"],
        max_tool_calls_per_turn=10,
    ),
    audit=AuditConfig(enabled=True, log_tool_calls=True),
    license=LicenseConfig(plan="enterprise", max_seats=50),
    data_retention=DataRetention.SESSION,
    compliance=[ComplianceFramework.SOC2],
)
```

## Per-Tenant Isolation

Each tenant gets its own independent configuration. No settings leak between tenants:

```python
# Two completely independent configurations
acme = TenantConfig(tenant_id="acme", safety=SafetyPolicy(level=SafetyLevel.STRICT))
beta = TenantConfig(tenant_id="beta", safety=SafetyPolicy(level=SafetyLevel.PERMISSIVE))

# Each tenant's agents respect only their own config
assert acme.safety.level != beta.safety.level
```

## User-Overridable Settings

Admins control which settings individual users can change. The `user_overridable` set on `TenantConfig` defines the whitelist:

```python
config = TenantConfig(
    tenant_id="acme",
    safety=SafetyPolicy(level=SafetyLevel.STRICT),
    user_overridable={
        "safety.max_autonomy",
        "models.max_tokens_per_request",
    },
)

# Check if a user can override a setting
config.user_can_override("safety.max_autonomy")      # True
config.user_can_override("safety.pii_detection")      # False

# LOCKED safety level prevents ALL user overrides
config.safety.level = SafetyLevel.LOCKED
config.user_can_override("safety.max_autonomy")        # False
```

## Persistence

Configs are fully serializable to JSON for on-prem file-based storage:

```python
# Save
config.save("config/acme.json")

# Load
loaded = TenantConfig.load("config/acme.json")
assert loaded.tenant_id == "acme"
```

The `save()` method automatically creates parent directories and updates the `updated_at` timestamp.

## Sub-Policy Reference

| Sub-Policy | Class | Purpose |
|-----------|-------|---------|
| Safety | `SafetyPolicy` | Content filtering, PII, injection protection, autonomy caps |
| Models | `ModelPolicy` | Allowed/blocked models, token limits, thinking mode |
| Tools | `ToolPolicy` | Allowed/blocked tools, approval gates, call limits |
| Audit | `AuditConfig` | Log destinations, what to log, retention period |
| License | `LicenseConfig` | Plan, seats, feature gates |
| Data Retention | `DataRetention` | `NONE`, `SESSION`, `PERSISTENT`, or `AUDIT_ONLY` |
| Compliance | `ComplianceFramework` | `GDPR`, `SOC2`, `HIPAA`, `ISO27001`, `PCI_DSS`, `CCPA`, `FedRAMP` |

## Custom Settings

For organization-specific needs not covered by the standard sub-policies, use the `custom_settings` dictionary:

```python
config = TenantConfig(
    tenant_id="acme",
    custom_settings={
        "department": "engineering",
        "cost_center": "eng-042",
        "max_concurrent_sessions": 10,
    },
)
```
