# Tenant Manager API Reference

## Module: `corteX.tenancy.manager`

Lifecycle manager for all active tenants. Provides creation, retrieval, hot-reload, and GDPR-compliant destruction of tenant resources in a single process.

## Classes

### `TenantSummary`

**Type**: `@dataclass`

Lightweight snapshot of a tenant's current state.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `tenant_id` | `str` | Unique tenant identifier |
| `active_sessions` | `int` | Number of currently active sessions |
| `tokens_used_today` | `int` | Cumulative token usage for today (UTC) |
| `created_at` | `float` | Unix timestamp of when the tenant was provisioned |

---

### `ErasureReceipt`

**Type**: `@dataclass`

Proof-of-deletion artifact for GDPR Art. 17 compliance.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `tenant_id` | `str` | The deleted tenant |
| `erased_at` | `float` | Unix timestamp of erasure |
| `components_erased` | `List[str]` | Component names that were wiped (e.g., `"key_vault"`, `"config"`, `"usage_counters"`) |
| `success` | `bool` | Whether the full erasure completed without error |

---

### `TenantManager`

Lifecycle manager for all active tenants in a corteX process. Each tenant gets its own isolated set of resources: config, key vault, quota, DNA profile, and engine references.

#### Constructor

```python
TenantManager()
```

No parameters. Initializes an empty tenant registry.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `tenant_count` | `int` | Number of currently active tenants |

#### Methods

##### `create_tenant`

```python
def create_tenant(
    self, tenant_id: str, config: Dict[str, Any],
    keys: Optional[Dict[str, str]] = None
) -> Dict[str, Any]
```

Provision a new tenant with isolated resources. Creates a `KeyVault` and stores any provided API keys.

**Parameters**:

- `tenant_id` (`str`): Unique tenant identifier.
- `config` (`Dict[str, Any]`): Tenant configuration (deep-copied).
- `keys` (`Optional[Dict[str, str]]`): Mapping of provider name to API key.

**Returns**: `Dict[str, Any]` with keys `tenant_id`, `providers`, `created_at`.

**Raises**: `ValueError` if `tenant_id` already exists or is empty.

##### `get_tenant`

```python
def get_tenant(self, tenant_id: str) -> Optional[Dict[str, Any]]
```

Retrieve a tenant's info (without exposing the vault). Returns `None` if not found.

**Returns**: `Optional[Dict[str, Any]]` with keys `tenant_id`, `config`, `active_sessions`, `tokens_used_today`, `created_at`, `updated_at`.

##### `destroy_tenant`

```python
def destroy_tenant(self, tenant_id: str) -> Dict[str, Any]
```

Full data erasure for GDPR Art. 17 compliance. Destroys the vault, clears config, and removes the tenant. Returns an erasure receipt dict.

**Returns**: `Dict[str, Any]` with keys `tenant_id`, `erased_at`, `components_erased`, `success`.

**Raises**: `ValueError` if the tenant does not exist.

**Components erased**:

1. `key_vault` -- All API keys zeroed and vault destroyed
2. `config` -- Configuration cleared
3. `usage_counters` -- Session and token counters reset

##### `update_config`

```python
def update_config(self, tenant_id: str, config: Dict[str, Any]) -> None
```

Hot-reload tenant configuration. The new config replaces the existing one at the next turn boundary.

**Raises**: `ValueError` if the tenant does not exist.

##### `list_tenants`

```python
def list_tenants(self) -> List[TenantSummary]
```

Return a summary list of all active tenants.

##### `increment_sessions` / `decrement_sessions`

```python
def increment_sessions(self, tenant_id: str) -> None
def decrement_sessions(self, tenant_id: str) -> None
```

Adjust the active session counter for a tenant. Used by the runtime.

##### `add_tokens`

```python
def add_tokens(self, tenant_id: str, count: int) -> None
```

Add to today's token usage for a tenant.

##### `get_vault`

```python
def get_vault(self, tenant_id: str) -> Optional[KeyVault]
```

Retrieve the `KeyVault` for a tenant (internal use only).

---

## Tenant Isolation

Each tenant's resources are fully isolated:

| Resource | Isolation Level |
|----------|----------------|
| API Keys | Separate `KeyVault` per tenant |
| Configuration | Deep-copied dict per tenant |
| Session Counters | Per-tenant tracking |
| Token Usage | Per-tenant daily counters |

---

## Example

```python
from corteX.tenancy.manager import TenantManager

manager = TenantManager()

# Create a tenant
info = manager.create_tenant(
    "acme",
    config={"model": "gemini-3-pro-preview", "max_tokens": 4096},
    keys={"openai": "sk-abc123...", "gemini": "AIza..."},
)
print(f"Created: {info['tenant_id']}, providers: {info['providers']}")

# Retrieve tenant info
tenant = manager.get_tenant("acme")
print(f"Sessions: {tenant['active_sessions']}")

# Hot-reload configuration
manager.update_config("acme", {"model": "gemini-3-flash-preview"})

# Session bookkeeping
manager.increment_sessions("acme")
manager.add_tokens("acme", 1500)

# List all tenants
for summary in manager.list_tenants():
    print(f"{summary.tenant_id}: {summary.tokens_used_today} tokens")

# GDPR destruction
receipt = manager.destroy_tenant("acme")
print(f"Erased: {receipt['components_erased']}")
```

---

## See Also

- [Key Vault](../security/vault.md) -- Per-tenant API key storage
- [Tenant DNA](./dna.md) -- Learned usage profiles per tenant
- [Quota Tracker](./quota.md) -- Per-tenant resource limits
