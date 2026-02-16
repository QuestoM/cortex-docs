# GDPR Manager API Reference

## Module: `corteX.enterprise.gdpr`

Full GDPR Data Subject Access Request (DSAR) lifecycle manager covering Articles 13-22. Handles: access (Art. 15), rectification (Art. 16), erasure (Art. 17), restriction (Art. 18), portability (Art. 20), objection (Art. 21), and transparency (Art. 13-14).

Erasure cascades across ALL data stores (memory, conversations, weights, preferences, audit logs). All operations are logged to an immutable audit trail.

## Enums

### `DSARType`

**Type**: `str, Enum`

GDPR Data Subject Access Request types.

| Value | GDPR Article | Description |
|-------|-------------|-------------|
| `access` | Art. 15 | Right of access -- export all user data |
| `rectification` | Art. 16 | Right to correction |
| `erasure` | Art. 17 | Right to be forgotten |
| `restriction` | Art. 18 | Right to restrict processing |
| `portability` | Art. 20 | Right to data portability |
| `objection` | Art. 21 | Right to object to processing |
| `transparency` | Art. 13-14 | Right to information |

---

### `DSARStatus`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `pending` | Request received, not yet processed |
| `in_progress` | Request being processed |
| `completed` | Request fully processed |
| `partially_completed` | Some stores processed, some failed |
| `denied` | Request denied (with reason) |
| `failed` | Request failed |

---

### `ErasureScope`

**Type**: `str, Enum`

Granular control over what data to erase for Art. 17 requests.

| Value | Description |
|-------|-------------|
| `all` | Everything across all stores |
| `conversations` | Message history only |
| `working_memory` | Working memory items |
| `episodic_memory` | Episodic memory items |
| `semantic_memory` | Semantic memory items |
| `cold_storage` | Cold/archival storage |
| `weight_deltas` | User-specific weight adjustments |
| `preferences` | User insight weights |
| `audit_logs` | Audit trail entries for user |

---

## Classes

### `GDPRManager`

Manages GDPR Data Subject Access Requests for corteX. Integrates with `MemoryFabric`, `WeightEngine`, and `AuditLogger` for full data lifecycle management.

#### Constructor

```python
GDPRManager(
    memory: Optional[MemoryFabric] = None,
    weights: Optional[WeightEngine] = None,
    audit: Optional[AuditLogger] = None,
    conversations: Optional[Dict[str, List[Dict[str, Any]]]] = None,
    user_preferences: Optional[Dict[str, Dict[str, Any]]] = None,
)
```

**Parameters**:

- `memory` (`Optional[MemoryFabric]`): Memory fabric instance for accessing stored memories.
- `weights` (`Optional[WeightEngine]`): Weight engine for accessing/erasing weight deltas.
- `audit` (`Optional[AuditLogger]`): Audit logger for compliance logging.
- `conversations` (`Optional[Dict]`): Conversation store (keyed by `tenant_id:user_id`).
- `user_preferences` (`Optional[Dict]`): User preference store (keyed by `tenant_id:user_id`).

#### Methods

##### `export_user_data` (Art. 15)

```python
async def export_user_data(
    self, tenant_id: str, user_id: str, format: str = "json",
) -> dict
```

Right of Access -- export ALL data for a user across all stores. Returns a comprehensive export including conversations, all memory tiers, cold storage, weight deltas, preferences, restrictions, and objections.

**Returns**: `dict` -- Complete data export with metadata.

##### `rectify_user_data` (Art. 16)

```python
async def rectify_user_data(
    self, tenant_id: str, user_id: str, corrections: dict,
) -> bool
```

Right to Rectification -- apply corrections to user data. Corrections use dot-notation paths (e.g., `"preferences.language"`, `"working_memory.key_name"`).

**Parameters**:

- `corrections` (`dict`): Map of `path -> value` corrections.

**Returns**: `bool` -- `True` if any corrections were applied.

##### `erase_user_data` (Art. 17)

```python
async def erase_user_data(
    self, tenant_id: str, user_id: str, scope: str = "all",
) -> dict
```

Right to Erasure (right to be forgotten) -- cascade delete across ALL stores.

**Parameters**:

- `scope` (`str`): Erasure scope. `"all"` for complete erasure, or a specific `ErasureScope` value.

**Returns**: `dict` with keys:
- `total_deleted` (`int`): Total items erased.
- `results` (`List[dict]`): Per-store erasure details.
- `all_success` (`bool`): Whether all store erasures succeeded.

##### `restrict_processing` (Art. 18)

```python
async def restrict_processing(
    self, tenant_id: str, user_id: str, restricted: bool = True,
) -> bool
```

Right to Restriction -- mark a user as processing-restricted. When restricted, the agent should only store (not process) data.

**Returns**: `bool` -- Always `True`.

##### `is_processing_restricted`

```python
def is_processing_restricted(self, tenant_id: str, user_id: str) -> bool
```

Check if processing is restricted for a user.

##### `export_portable_data` (Art. 20)

```python
async def export_portable_data(
    self, tenant_id: str, user_id: str, format: str = "json",
) -> dict
```

Right to Data Portability -- structured, machine-readable export. Returns a subset of data in a portable schema (conversations, preferences, episodic data).

##### `register_objection` (Art. 21)

```python
async def register_objection(
    self, tenant_id: str, user_id: str, scope: str = "profiling",
) -> bool
```

Right to Object -- register an objection to profiling or processing.

##### `has_objection`

```python
def has_objection(self, tenant_id: str, user_id: str) -> bool
```

Check if a user has registered an objection.

##### `get_processing_info` (Art. 13-14)

```python
async def get_processing_info(
    self, tenant_id: str, user_id: str,
) -> dict
```

Transparency -- returns what data is processed and why, including legal basis, data categories, retention periods, and available rights.

##### `get_request_log`

```python
def get_request_log(self) -> List[Dict[str, Any]]
```

Get all DSAR request/response records for compliance auditing. Returns serialized `DSARResponse` objects.

---

## Constants

| Name | Value | Description |
|------|-------|-------------|
| `DSAR_RESPONSE_DEADLINE_DAYS` | `30` | Maximum days to respond to a DSAR (Art. 12(3)) |

---

## Example

```python
from corteX.enterprise.gdpr import GDPRManager

# Initialize with data stores
gdpr = GDPRManager(
    memory=memory_fabric,
    weights=weight_engine,
    audit=audit_logger,
)

# Art. 15 -- Right of Access
export = await gdpr.export_user_data("acme", "user_42")
print(f"Exported {len(export['data'])} categories")

# Art. 16 -- Right to Rectification
await gdpr.rectify_user_data("acme", "user_42", {
    "preferences.language": "de",
    "preferences.timezone": "Europe/Berlin",
})

# Art. 17 -- Right to Erasure (cascade delete)
result = await gdpr.erase_user_data("acme", "user_42", scope="all")
print(f"Erased {result['total_deleted']} items, success={result['all_success']}")

# Art. 17 -- Selective erasure
result = await gdpr.erase_user_data("acme", "user_42", scope="conversations")

# Art. 18 -- Restrict processing
await gdpr.restrict_processing("acme", "user_42")
if gdpr.is_processing_restricted("acme", "user_42"):
    print("Processing restricted -- store only, no analysis")

# Art. 20 -- Data portability
portable = await gdpr.export_portable_data("acme", "user_42")

# Art. 21 -- Object to profiling
await gdpr.register_objection("acme", "user_42", scope="profiling")

# Art. 13-14 -- Transparency
info = await gdpr.get_processing_info("acme", "user_42")
print(info["purposes"])

# Compliance audit trail
log = gdpr.get_request_log()
print(f"{len(log)} DSAR requests processed")
```

---

## See Also

- [Consent Manager](./consent.md) -- Granular consent lifecycle
- [Retention Manager](./retention.md) -- Automated data retention enforcement
- [Profiling Manager](./profiling.md) -- Art. 22 profiling opt-out
- [Data Residency](./data-residency.md) -- Art. 44-49 cross-border data controls
- [Audit Logger](../security/audit-logger.md) -- Tamper-evident audit logging
