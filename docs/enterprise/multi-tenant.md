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

---

## TenantContext: Automatic Async Propagation

**corteX Innovation**: Python contextvars-based tenant context that automatically propagates across async calls without explicit passing.

The `TenantContext` system provides automatic tenant isolation in multi-tenant deployments using Python's contextvars, ensuring that all async operations maintain proper tenant boundaries.

### Why: The Async Propagation Problem

In async Python applications, tenant information must flow through multiple async calls:

```python
# Traditional approach - explicit tenant passing (brittle)
async def handle_request(tenant_id: str):
    await process_message(tenant_id)

async def process_message(tenant_id: str):
    await call_llm(tenant_id)

async def call_llm(tenant_id: str):
    config = get_tenant_config(tenant_id)  # Must pass everywhere
```

This creates:
- **Boilerplate**: Every function signature needs `tenant_id`
- **Error-prone**: Easy to forget to pass tenant_id
- **Security risk**: Mixing up tenant_id parameters can leak data

TenantContext solves this with **automatic async propagation** using Python's contextvars.

### How It Works

```python
from corteX.tenancy import TenantContext, get_current_tenant

# Set tenant context once at the request boundary
async def handle_request(tenant_id: str, user_id: str, end_user_id: str):
    # Set tenant context - propagates to all child async calls
    with TenantContext(
        tenant_id=tenant_id,
        user_id=user_id,
        end_user_id=end_user_id,
    ):
        # All functions called within this block automatically have access
        await process_message()

async def process_message():
    # No tenant_id parameter needed - automatically available
    tenant = get_current_tenant()
    print(f"Processing for tenant: {tenant.tenant_id}")
    await call_llm()

async def call_llm():
    # Still no tenant_id parameter - propagated automatically
    tenant = get_current_tenant()
    config = tenant.get_config()  # Tenant-specific config
    logger.info(f"LLM call for {tenant.tenant_id}")
```

### Three-Layer Tenant Model

TenantContext supports a 3-layer tenant hierarchy:

```
Questo (SDK Provider)
  └─ acme_corp (SaaS Developer)
       └─ alice@acme.com (Developer's End-User)
```

| Layer | Field | Purpose |
|-------|-------|---------|
| **SDK Provider** | (implicit) | Questo provides the corteX SDK |
| **SaaS Developer** | `tenant_id` | The developer building on corteX (e.g., "acme_corp") |
| **End-User** | `end_user_id` | The developer's customer (e.g., "alice@acme.com") |

```python
# Example: Acme Corp (SaaS) uses corteX for their customer Alice
with TenantContext(
    tenant_id="acme_corp",  # Acme is the corteX customer
    user_id="dev@acme.com",  # Acme developer using corteX
    end_user_id="alice@acme.com",  # Alice is Acme's customer
):
    # All tenant-specific resources are isolated to acme_corp
    # All audit logs include end_user_id for Acme's compliance
    await session.run("Help me with my account")
```

### Automatic Isolation

All SDK components automatically respect tenant context:

#### 1. EventBus Isolation

The EventBus uses instance-level subscribers (not class-level), ensuring events don't leak between tenants:

```python
# Tenant A's session
with TenantContext(tenant_id="tenant_a"):
    session_a = cortex.Session()
    session_a.event_bus.subscribe("tool_executed", handler_a)

# Tenant B's session (independent event bus)
with TenantContext(tenant_id="tenant_b"):
    session_b = cortex.Session()
    session_b.event_bus.subscribe("tool_executed", handler_b)

# Tenant A's events only go to handler_a
# Tenant B's events only go to handler_b
```

#### 2. State File Isolation

StateFileManager automatically partitions state files by tenant:

```python
from corteX.engine.cognitive import StateFileManager

# Inside a tenant context
with TenantContext(tenant_id="acme_corp"):
    manager = StateFileManager(session_id="cs-001")
    # Automatically saves to: /state/acme_corp/cs-001_crystallized.json
    manager.save_crystallized(state)

with TenantContext(tenant_id="beta_inc"):
    manager = StateFileManager(session_id="cs-001")
    # Automatically saves to: /state/beta_inc/cs-001_crystallized.json
    manager.save_crystallized(state)

# No cross-tenant access possible
```

#### 3. Tool Registry Isolation

Each session has its own tool registry (not global):

```python
from corteX.tools import tool

# Register tool for tenant A
with TenantContext(tenant_id="tenant_a"):
    session_a = cortex.Session()

    @tool(session=session_a)
    def restricted_tool():
        """Only available to tenant_a"""
        pass

# Tenant B doesn't see restricted_tool
with TenantContext(tenant_id="tenant_b"):
    session_b = cortex.Session()
    # restricted_tool is NOT in session_b.tool_registry
```

#### 4. Cost Tracking Isolation

CostTracker automatically aggregates costs by tenant:

```python
from corteX.engine.routing import CostTracker

tracker = CostTracker()

with TenantContext(tenant_id="acme_corp"):
    tracker.record_usage(
        session_id="cs-001",
        model_id="gemini-3-pro",
        input_tokens=1000,
        output_tokens=500,
        cost=0.00031,
    )

# Get tenant-level costs (automatically uses current tenant context)
tenant_cost = tracker.get_total_cost(scope="tenant")
# Returns costs for "acme_corp" only
```

#### 5. API Key Isolation

LLM providers require explicit API key configuration (no env fallback in multi-tenant mode):

```python
# Each tenant brings their own API keys
with TenantContext(tenant_id="acme_corp"):
    engine = cortex.Engine(
        providers={
            "gemini": {"api_key": acme_gemini_key},
            "openai": {"api_key": acme_openai_key},
        }
    )

with TenantContext(tenant_id="beta_inc"):
    engine = cortex.Engine(
        providers={
            "gemini": {"api_key": beta_gemini_key},
        }
    )

# No key leakage between tenants
```

### Accessing Tenant Context

Get the current tenant anywhere in the call stack:

```python
from corteX.tenancy import get_current_tenant, TenantNotFoundError

try:
    tenant = get_current_tenant()
    print(f"Tenant: {tenant.tenant_id}")
    print(f"User: {tenant.user_id}")
    print(f"End-user: {tenant.end_user_id}")
except TenantNotFoundError:
    # No tenant context set - handle gracefully
    logger.warning("No tenant context available")
```

### Testing with Tenant Context

Use tenant context in tests to simulate multi-tenant scenarios:

```python
import pytest
from corteX.tenancy import TenantContext

@pytest.mark.asyncio
async def test_tenant_isolation():
    # Test tenant A
    with TenantContext(tenant_id="tenant_a"):
        session_a = cortex.Session()
        result_a = await session_a.run("Test message")

    # Test tenant B
    with TenantContext(tenant_id="tenant_b"):
        session_b = cortex.Session()
        result_b = await session_b.run("Test message")

    # Verify isolation
    assert session_a.tenant_id != session_b.tenant_id
    assert session_a.event_bus is not session_b.event_bus
```

### Best Practices

1. **Set context at request boundary**: In web apps, set TenantContext in middleware
2. **Never mix tenant contexts**: Don't nest TenantContext blocks for different tenants
3. **Always handle TenantNotFoundError**: Some functions may be called outside tenant context
4. **Use explicit tenant_id for queries**: When querying across tenants (admin operations), pass tenant_id explicitly
5. **Audit all tenant switches**: Log whenever tenant context changes

### Migration from Global State

If you have existing code using global singletons, migrate to instance-level:

```python
# Before (global singleton - WRONG)
_global_event_bus = EventBus()

def get_event_bus():
    return _global_event_bus

# After (instance-level - CORRECT)
class Session:
    def __init__(self):
        self.event_bus = EventBus()  # Per-session instance

session = Session()
session.event_bus.subscribe(...)
```

---

## API Reference

```python
from corteX.tenancy import (
    TenantContext,
    get_current_tenant,
    TenantNotFoundError,
)
from corteX.enterprise.config import (
    TenantConfig,
    SafetyPolicy,
    ModelPolicy,
    ToolPolicy,
    AuditConfig,
    LicenseConfig,
)
```

See also:
- [Security & Isolation](security.md) - Security controls and data isolation
- [Audit Logging](audit.md) - Per-tenant audit trails
- [Intelligent Model Routing](../concepts/model-routing.md) - Tenant-specific model overrides
