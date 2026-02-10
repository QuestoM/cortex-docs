# Enterprise Overview

corteX is built enterprise-first. Every feature works on-premises, offline, and under strict administrative control. There is no hidden telemetry, no required cloud dependency, and no surprise network calls.

## Design Principles

| Principle | What It Means |
|-----------|--------------|
| **On-Prem First** | Everything works without internet access. Cloud features are opt-in, never required. |
| **Per-Tenant Isolation** | Each customer deployment gets its own `TenantConfig` with independent safety, model, tool, and audit policies. |
| **Admin Control, User Override** | Admins set the rules. Users can override only the settings the admin explicitly allows. |
| **Audit Everything** | Every config change, tool call, weight update, and model routing decision can be logged. |
| **Zero Trust by Default** | No outbound network calls happen unless an explicit tool or integration is configured. Data never leaves the deployment boundary. |

## Architecture at a Glance

```
+-----------------------------------------------------+
|                   TenantConfig                       |
|  +-----------+  +-----------+  +------------------+  |
|  | SafetyPolicy| ModelPolicy| ToolPolicy         |  |
|  +-----------+  +-----------+  +------------------+  |
|  +-----------+  +-----------+  +------------------+  |
|  | AuditConfig| LicenseConfig| ComplianceFramework|  |
|  +-----------+  +-----------+  +------------------+  |
+-----------------------------------------------------+
        |                |               |
   Safety Gate     Model Router    Tool Executor
        |                |               |
+-----------------------------------------------------+
|              Brain Engine (Weights, Memory, etc.)    |
+-----------------------------------------------------+
```

## Key Enterprise Features

### Multi-Tenant Configuration
Each deployment is governed by a `TenantConfig` that encapsulates all policies. Configs can be saved to JSON files and loaded at startup for fully offline operation. See [Multi-Tenant](multi-tenant.md).

### Safety Policies
Four safety levels (`PERMISSIVE`, `MODERATE`, `STRICT`, `LOCKED`) control content filtering, PII detection, prompt injection protection, and human-in-the-loop approval requirements. See [Safety](safety.md).

### Security Model
The SDK makes **zero outbound network calls** unless the developer explicitly configures a tool or integration that does so. All data stays within the deployment. See [Security](security.md).

### Audit Logging
Every tool call, weight change, model routing decision, and policy activation can be logged to file, syslog, or webhook with configurable retention. See [Audit](audit.md).

### Compliance
Built-in framework awareness for SOC 2, GDPR, HIPAA, ISO 27001, PCI DSS, CCPA, and FedRAMP. Compliance mode enforces data retention, audit, and safety rules. See [Compliance](compliance.md).

### Licensing
Per-seat licensing with Ed25519 cryptographic validation that works completely offline. License keys are self-contained signed tokens. A 30-day grace period ensures on-prem deployments are never disrupted. See [Licensing](licensing.md).

### On-Premises Deployment
Zero network dependency for core operation. Local models, file-based persistence, air-gapped update delivery via signed packages. See [On-Prem](on-prem.md).

### Update Delivery
Supports private PyPI registries, signed package archives for air-gapped environments, and optional version checking. Updates are never forced. See [Updates](updates.md).

## Quick Start: Enterprise Configuration

```python
from corteX.enterprise import TenantConfig, LicenseConfig, SafetyPolicy
from corteX.enterprise.config import SafetyLevel, ComplianceFramework

config = TenantConfig(
    tenant_id="acme_corp",
    tenant_name="Acme Corporation",
    license=LicenseConfig(
        license_key="LK-xxxx.yyyy",
        organization_id="acme",
        plan="enterprise",
        max_seats=50,
    ),
    safety=SafetyPolicy(
        level=SafetyLevel.STRICT,
        pii_detection=True,
        injection_protection=True,
        blocked_topics=["competitor_data", "salary_info"],
        require_human_approval=["file_delete", "deploy_production"],
    ),
    compliance=[ComplianceFramework.SOC2, ComplianceFramework.GDPR],
)

# Persist to disk for on-prem startup
config.save("config/acme_corp.json")

# Load at application startup
loaded = TenantConfig.load("config/acme_corp.json")
```
