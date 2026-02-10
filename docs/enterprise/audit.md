# Audit Logging

corteX provides comprehensive audit logging for enterprise deployments. Every significant system event can be recorded for compliance, debugging, and forensic analysis.

## Configuration

Audit logging is configured through `AuditConfig` within the `TenantConfig`:

```python
from corteX.enterprise.config import AuditConfig

audit = AuditConfig(
    enabled=True,
    log_messages=False,       # Don't log conversation content (privacy)
    log_tool_calls=True,      # Log every tool invocation
    log_weight_changes=True,  # Log weight system updates
    log_model_routing=True,   # Log model selection decisions
    log_destination="file",   # file, syslog, or webhook
    log_path="/var/log/cortex/audit.log",
    webhook_url=None,
    retention_days=90,
)
```

## What Gets Logged

| Event Category | `log_messages` | `log_tool_calls` | `log_weight_changes` | `log_model_routing` |
|---------------|:-:|:-:|:-:|:-:|
| Conversation content | x | | | |
| Tool invocations and results | | x | | |
| Weight engine updates | | | x | |
| Model selection decisions | | | | x |
| Safety policy activations | Always (when audit enabled) | | | |
| Enterprise policy evaluations | Always (when audit enabled) | | | |
| License status checks | Always (when audit enabled) | | | |

## Log Destinations

### File-Based Logging
```python
audit = AuditConfig(
    enabled=True,
    log_destination="file",
    log_path="/var/log/cortex/audit.log",
)
```

### Syslog
```python
audit = AuditConfig(
    enabled=True,
    log_destination="syslog",
)
```

### Webhook
```python
audit = AuditConfig(
    enabled=True,
    log_destination="webhook",
    webhook_url="https://siem.internal.acme.com/ingest",
)
```

## Enterprise Modulation Audit Trail

The `TargetedModulator` maintains its own detailed audit trail for every modulation operation:

```python
modulator = TargetedModulator()

# Every operation is audit-logged
modulator.silence("dangerous_tool", reason="investigating safety issue")

# Retrieve the full trail
trail = modulator.get_audit_trail()
# Each entry includes: event type, timestamp, turn number,
# modulation_id, target, details, and actor
```

The enterprise policy engine logs:
- `policy_added` -- new policy registered
- `policy_removed` -- policy removed
- `policy_activated` -- policy matched a target and generated a modulation
- `policy_evaluated` -- policy was checked during `apply_modulations`
- `policy_integrity_violation` -- tamper detection triggered
- `policy_rejected_tampered` -- policy registration rejected due to integrity failure

## Retention

The `retention_days` field controls how long audit logs are kept. Default is 90 days. For compliance frameworks like HIPAA, this should be set to at least 6 years (2190 days).

## Feature Gating

Audit logging is a feature-gated capability. It requires the `audit_logging` feature in the license:

```python
if config.license.has_feature("audit_logging"):
    config.audit.enabled = True
```

The `audit_logging` feature is available on the `enterprise` and `unlimited` license plans.
