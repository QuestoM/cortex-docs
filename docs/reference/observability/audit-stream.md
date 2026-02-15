# Tenant Audit Stream API Reference

## Module: `corteX.observability.audit_stream`

Enterprise tamper-evident, hash-chained, per-tenant audit log. Brain analogy: hippocampal replay creating immutable episodic memories. Each entry is SHA-256 chained to its predecessor, producing a tamper-evident log suitable for SOC2 CC7.2, GDPR Art. 30, and legal proceedings.

## Classes

### `AuditEventType`

**Type**: `str, Enum`

Types of enterprise audit events.

| Value | Description |
|-------|-------------|
| `LLM_CALL` | A call to a language model |
| `TOOL_EXECUTION` | A tool was executed |
| `POLICY_DECISION` | A security/compliance decision |
| `GOAL_EVENT` | Goal creation, update, or completion |
| `SESSION_START` | Session began |
| `SESSION_END` | Session ended |
| `DATA_ACCESS` | Data was accessed |
| `CONFIG_CHANGE` | Configuration was modified |

---

### `EnterpriseAuditEntry`

**Type**: `@dataclass`

A single tamper-evident audit entry with hash chain link.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `event_id` | `str` | Auto-generated 16-char hex ID |
| `tenant_id` | `str` | Tenant identifier (set by the stream) |
| `session_id` | `str` | Session identifier |
| `user_id` | `str` | User identifier |
| `timestamp` | `float` | Unix timestamp |
| `event_type` | `AuditEventType` | Type of event |
| `action` | `str` | Action performed |
| `justification` | `str` | Why the action was taken |
| `outcome` | `str` | Result of the action |
| `cost_tokens` | `int` | Tokens consumed |
| `data_level` | `str` | Data classification level |
| `capabilities_used` | `List[str]` | Capabilities exercised |
| `chain_hash` | `str` | SHA-256 hash linking to previous entry |
| `metadata` | `Dict[str, Any]` | Additional metadata |

#### Methods

- `to_dict() -> Dict[str, Any]` -- Serialize with `event_type` as string value
- `to_json() -> str` -- Serialize to JSON string
- `hashable_content() -> str` -- Content used for hash chain computation (excludes `chain_hash`)

---

### `TenantAuditStream`

Per-tenant tamper-evident audit stream with SHA-256 hash chain. Append-only log where each entry's `chain_hash` is computed from the entry content combined with the previous entry's `chain_hash`.

#### Constructor

```python
TenantAuditStream(tenant_id: str, max_entries: int = 50_000)
```

**Parameters**:

- `tenant_id` (`str`): Tenant identifier.
- `max_entries` (`int`, default=50,000): Maximum entries before oldest are evicted (minimum 100).

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `tenant_id` | `str` | Tenant identifier |
| `length` | `int` | Number of entries in the stream |

#### Methods

##### `append`

```python
def append(self, entry: EnterpriseAuditEntry) -> EnterpriseAuditEntry
```

Append entry with computed chain hash. The entry's `tenant_id` is set to the stream's tenant, and `chain_hash` is computed as `SHA-256(previous_hash + ":" + hashable_content)`.

**Parameters**:

- `entry` (`EnterpriseAuditEntry`): Entry to append.

**Returns**: `EnterpriseAuditEntry` -- The entry with `chain_hash` populated.

##### `query`

```python
def query(self, filters: Dict[str, Any]) -> List[EnterpriseAuditEntry]
```

Query entries with filters.

**Filter keys**:

| Key | Type | Description |
|-----|------|-------------|
| `event_type` | `str` or `AuditEventType` | Filter by event type |
| `session_id` | `str` | Filter by session |
| `user_id` | `str` | Filter by user |
| `start_time` | `float` | Entries after this timestamp |
| `end_time` | `float` | Entries before this timestamp |
| `action` | `str` | Substring match on action |
| `data_level` | `str` | Exact match on data level |
| `limit` | `int` | Maximum results (default 100) |

##### `verify_chain`

```python
def verify_chain(self) -> Tuple[bool, Optional[int]]
```

Verify hash chain integrity by recomputing hashes from genesis.

**Returns**: `Tuple[bool, Optional[int]]` -- `(True, None)` if valid, `(False, break_index)` if tampered.

##### `export`

```python
def export(self, format: str = "jsonl") -> List[str]
```

Export entries for compliance.

**Parameters**:

- `format` (`str`): `"jsonl"` (default) or `"csv"`.

**Returns**: `List[str]` -- Serialized entries (JSONL) or CSV rows with header.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: Dict with `tenant_id`, `total_entries`, `max_entries`, `entries_by_type`, `last_hash` (truncated).

##### `clear`

```python
def clear(self) -> int
```

Clear all entries and reset chain to genesis hash. Returns count removed.

---

## Hash Chain Design

```
entry[0].chain_hash = SHA-256("0"*64 + ":" + entry[0].hashable_content())
entry[1].chain_hash = SHA-256(entry[0].chain_hash + ":" + entry[1].hashable_content())
entry[n].chain_hash = SHA-256(entry[n-1].chain_hash + ":" + entry[n].hashable_content())
```

The genesis hash is 64 zero characters. Each subsequent hash links to its predecessor, creating an immutable chain. Any modification to a past entry invalidates all subsequent hashes.

---

## Compliance Coverage

| Standard | Section | Coverage |
|----------|---------|----------|
| SOC2 | CC7.2 | Security event logging |
| GDPR | Art. 30 | Records of processing activities |
| HIPAA | 164.312(b) | Audit controls |
| ISO 27001 | A.12.4 | Logging and monitoring |

---

## Example

```python
from corteX.observability.audit_stream import (
    TenantAuditStream, EnterpriseAuditEntry, AuditEventType,
)

stream = TenantAuditStream("acme", max_entries=50_000)

# Append audit entries
entry = stream.append(EnterpriseAuditEntry(
    session_id="sess_001",
    user_id="user_42",
    event_type=AuditEventType.LLM_CALL,
    action="generate_code",
    justification="User requested code generation",
    outcome="Generated 150 lines of Python",
    cost_tokens=2500,
    data_level="internal",
    capabilities_used=["tool:code_interpreter:execute"],
))
print(f"Hash: {entry.chain_hash[:16]}...")

stream.append(EnterpriseAuditEntry(
    session_id="sess_001",
    user_id="user_42",
    event_type=AuditEventType.TOOL_EXECUTION,
    action="code_interpreter",
    justification="Execute generated code for testing",
    outcome="All tests passed",
    cost_tokens=0,
    data_level="internal",
))

# Verify chain integrity
valid, break_idx = stream.verify_chain()
print(f"Chain valid: {valid}")

# Query entries
llm_entries = stream.query({
    "event_type": "llm_call",
    "session_id": "sess_001",
    "limit": 50,
})

# Export for compliance
jsonl_lines = stream.export("jsonl")
csv_lines = stream.export("csv")

# Statistics
stats = stream.get_stats()
print(f"Total entries: {stats['total_entries']}")
print(f"By type: {stats['entries_by_type']}")
```

---

## See Also

- [Compliance Engine](../security/compliance.md) -- Framework-specific compliance rules
- [Data Classifier](../security/classification.md) -- Data level classification
- [Metrics Collector](./metrics.md) -- Real-time operational metrics
- [Decision Tracer](./tracer.md) -- Decision-level tracing
