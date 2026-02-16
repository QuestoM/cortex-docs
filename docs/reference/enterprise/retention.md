# Retention Manager API Reference

## Module: `corteX.enterprise.retention`

Automated data retention enforcement engine. Enforces retention policies based on `TenantConfig.data_retention` mode, per-data-category TTL settings, compliance framework presets (GDPR, SOC 2, HIPAA, PCI-DSS), and cascading user deletion (Art. 17 complement).

Enforcement runs on demand or can be scheduled via session lifecycle hooks. All purge operations support dry-run mode for safe review before deletion.

Brain analogy: synaptic pruning -- unused connections (data past TTL) are automatically removed to keep the system clean and compliant.

## Enums

### `DataCategory`

**Type**: `str, Enum`

Categories of data subject to retention policies.

| Value | Description |
|-------|-------------|
| `conversations` | Chat message history |
| `working_memory` | Short-lived working memory items |
| `episodic_memory` | Experience-based memory traces |
| `semantic_memory` | Knowledge and fact storage |
| `cold_storage` | Archival / cold storage tier |
| `weight_deltas` | User-specific weight adjustments |
| `audit_logs` | Audit trail entries |
| `session_data` | Session state and metadata |

---

## Data Models

### `RetentionPolicy`

**Type**: Pydantic `BaseModel`

Retention policy for a specific data category.

| Attribute | Type | Description |
|-----------|------|-------------|
| `category` | `DataCategory` | Data category this policy applies to |
| `ttl_seconds` | `Optional[int]` | Time-to-live in seconds. `None` = no expiry, `0` = immediate |
| `description` | `str` | Human-readable description |
| `compliance_source` | `str` | Which framework mandates this TTL |

### `ExpiredItem`

**Type**: Pydantic `BaseModel`

A single data item that has exceeded its retention period.

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_key` | `str` | Unique identifier for the item |
| `category` | `DataCategory` | Which data category |
| `created_at` | `float` | When the item was created |
| `expired_at` | `float` | When the TTL was exceeded |
| `age_seconds` | `float` | How old the item is |
| `ttl_seconds` | `int` | The TTL that was exceeded |
| `user_id` | `str` | User who owns the item |

### `PurgeResult`

**Type**: Pydantic `BaseModel`

Outcome of a purge operation.

| Attribute | Type | Description |
|-----------|------|-------------|
| `purge_id` | `str` | Unique purge operation ID |
| `tenant_id` | `str` | Tenant ID |
| `dry_run` | `bool` | Whether this was a preview-only run |
| `duration_ms` | `float` | Execution time in milliseconds |
| `categories_purged` | `Dict[str, CategoryPurgeDetail]` | Per-category breakdown |
| `total_deleted` | `int` | Total items deleted |
| `total_errors` | `int` | Total errors encountered |

### `RetentionReport`

**Type**: Pydantic `BaseModel`

Snapshot report of data retention status across all stores.

| Attribute | Type | Description |
|-----------|------|-------------|
| `report_id` | `str` | Unique report ID |
| `tenant_id` | `str` | Tenant ID |
| `policies` | `List[RetentionPolicy]` | Active retention policies |
| `category_stats` | `Dict[str, CategoryStats]` | Per-category statistics |
| `total_items` | `int` | Total data items across all stores |
| `total_expired` | `int` | Total expired items found/purged |
| `compliance_frameworks` | `List[str]` | Active frameworks |

### `CascadeDeleteResult`

**Type**: Pydantic `BaseModel`

Result of a cascading user data deletion.

| Attribute | Type | Description |
|-----------|------|-------------|
| `user_id` | `str` | Deleted user ID |
| `tenant_id` | `str` | Tenant ID |
| `categories_deleted` | `Dict[str, int]` | Items deleted per category |
| `total_deleted` | `int` | Total items deleted |

---

## Default TTLs

| Data Category | Default TTL | Human Readable |
|---------------|------------|----------------|
| `conversations` | 180 days | 6 months |
| `working_memory` | None | No expiry |
| `episodic_memory` | 365 days | 1 year |
| `semantic_memory` | None | No expiry |
| `cold_storage` | 365 days | 1 year |
| `weight_deltas` | 365 days | 1 year |
| `audit_logs` | 90 days | 3 months |
| `session_data` | 7 days | 1 week |

---

## Compliance Presets

Compliance frameworks override default TTLs (shortest TTL wins when multiple frameworks are active).

| Category | GDPR | SOC 2 | HIPAA | PCI-DSS |
|----------|------|-------|-------|---------|
| `conversations` | 90d | 365d | 6y | 90d |
| `working_memory` | 0 (immediate) | 30d | 30d | 0 |
| `episodic_memory` | 90d | 365d | 6y | 90d |
| `semantic_memory` | 180d | No expiry | No expiry | 365d |
| `audit_logs` | 90d | 365d | 6y | 365d |
| `session_data` | 0 | 30d | 30d | 0 |

---

## Classes

### `RetentionEnforcer`

Automated data retention enforcement engine.

#### Constructor

```python
RetentionEnforcer(
    config: TenantConfig,
    memory: Optional[MemoryFabric] = None,
    weights: Optional[WeightEngine] = None,
    audit: Optional[AuditLogger] = None,
    conversations: Optional[Dict[str, List[Dict[str, Any]]]] = None,
)
```

**Parameters**:

- `config` (`TenantConfig`): Tenant configuration with `data_retention` mode and `compliance` list.
- `memory` (`Optional[MemoryFabric]`): Memory fabric for scanning/purging memory stores.
- `weights` (`Optional[WeightEngine]`): Weight engine for purging weight deltas.
- `audit` (`Optional[AuditLogger]`): Audit logger for compliance logging.
- `conversations` (`Optional[Dict]`): Conversation store.

#### Methods

##### `set_ttl`

```python
def set_ttl(self, data_type: str, ttl_seconds: int) -> None
```

Set a custom TTL for a data category. Overrides defaults and compliance presets.

**Raises**: `ValueError` if `ttl_seconds` is negative.

##### `get_retention_policy`

```python
def get_retention_policy(self) -> dict
```

Get the current effective retention policy. Returns tenant ID, data retention mode, compliance frameworks, and per-category TTL with source (default/custom/framework).

##### `check_expired`

```python
async def check_expired(self) -> List[ExpiredItem]
```

Scan all stores for items past their TTL (read-only). Does not delete anything.

**Returns**: `List[ExpiredItem]` -- Items that have exceeded their retention period.

##### `purge_expired`

```python
async def purge_expired(self, dry_run: bool = False) -> PurgeResult
```

Delete expired items across all stores.

**Parameters**:

- `dry_run` (`bool`): If `True`, preview what would be deleted without actually deleting.

**Returns**: `PurgeResult` with per-category breakdown.

##### `enforce`

```python
async def enforce(self) -> RetentionReport
```

Full enforcement cycle: purge expired items and generate a retention report. Increments the internal enforcement counter.

**Returns**: `RetentionReport` -- Complete status report.

##### `cascade_delete_user`

```python
async def cascade_delete_user(
    self, tenant_id: str, user_id: str,
) -> CascadeDeleteResult
```

Delete ALL data for a user across ALL stores. Complements GDPR Art. 17 erasure.

**Returns**: `CascadeDeleteResult` with per-category deletion counts.

---

## Example

```python
from corteX.enterprise.config import TenantConfig, DataRetention, ComplianceFramework
from corteX.enterprise.retention import RetentionEnforcer

# Configure tenant with GDPR + SOC 2
config = TenantConfig(
    tenant_id="acme",
    data_retention=DataRetention.PERSISTENT,
    compliance=[ComplianceFramework.GDPR, ComplianceFramework.SOC2],
)

enforcer = RetentionEnforcer(
    config=config,
    memory=memory_fabric,
    audit=audit_logger,
)

# View effective retention policy
policy = enforcer.get_retention_policy()
for p in policy["policies"]:
    print(f"  {p['category']}: {p['ttl_human']} (source: {p['source']})")

# Custom TTL override
enforcer.set_ttl("conversations", 30 * 86400)  # 30 days

# Preview what would be purged (dry run)
preview = await enforcer.purge_expired(dry_run=True)
print(f"Would delete {preview.total_deleted} items")

# Actually purge
result = await enforcer.purge_expired(dry_run=False)
print(f"Purged {result.total_deleted} items in {result.duration_ms:.1f}ms")

# Full enforcement cycle
report = await enforcer.enforce()
print(f"Remaining items: {report.total_items}")

# Cascade user deletion (GDPR Art. 17)
cascade = await enforcer.cascade_delete_user("acme", "user_42")
print(f"Deleted {cascade.total_deleted} items for user_42")
```

---

## See Also

- [GDPR Manager](./gdpr.md) -- Full DSAR lifecycle with cascade erasure
- [Consent Manager](./consent.md) -- Consent-based processing control
- [Compliance Engine](../security/compliance.md) -- Framework-level policy enforcement
- [Audit Logger](../security/audit-logger.md) -- Tamper-evident audit trail
