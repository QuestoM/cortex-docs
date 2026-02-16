# Audit Logger API Reference

## Module: `corteX.security.audit_logger`

Persistent, tamper-evident audit logger for enterprise deployments. SOC 2 CC4.1/CC4.2/CC7.1/CC7.2 and GDPR Art 30 compliant. Each audit entry is linked to the previous via a SHA-256 hash chain, making any tampering detectable. Supports file persistence via `AuditFileStore`, in-memory ring buffer, and structured logging.

## Enums

### `AuditEventType`

**Type**: `str, Enum`

Auditable event categories for enterprise compliance.

| Value | Description |
|-------|-------------|
| `llm_call` | LLM API call executed |
| `llm_failure` | LLM API call failed |
| `tool_execution` | Tool successfully executed |
| `tool_failure` | Tool execution failed |
| `weight_change` | Synaptic weight modified |
| `security_block` | Action blocked by security policy |
| `injection_attempt` | Prompt injection detected |
| `classification_event` | Data classification triggered |
| `session_start` | Session created |
| `session_end` | Session ended |
| `policy_decision` | Compliance/policy decision made |
| `compliance_check` | Compliance framework check |
| `config_change` | Configuration changed |
| `data_access` | Data was accessed |
| `goal_event` | Goal tracking event |
| `custom` | Custom / catch-all event type |

---

### `AuditSeverity`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `debug` | Diagnostic information |
| `info` | Normal operations |
| `warning` | Potential issues |
| `error` | Errors that need attention |
| `critical` | Critical security events |

---

## Data Classes

### `AuditEntry`

**Type**: `@dataclass`

A single tamper-evident audit log entry.

| Attribute | Type | Description |
|-----------|------|-------------|
| `entry_id` | `str` | Unique identifier (auto-generated) |
| `tenant_id` | `str` | Tenant that owns this event |
| `session_id` | `str` | Session context |
| `user_id` | `str` | User who triggered the event |
| `timestamp` | `float` | Unix epoch seconds |
| `event_type` | `AuditEventType` | Event category |
| `severity` | `AuditSeverity` | Severity level |
| `action` | `str` | Short description of what happened |
| `details` | `Dict[str, Any]` | Structured key-value payload |
| `outcome` | `str` | Result summary (success/failure/blocked) |
| `chain_hash` | `str` | SHA-256 hash linking to previous entry |
| `sequence_num` | `int` | Monotonically increasing sequence number |

#### Methods

- `to_dict() -> Dict[str, Any]` -- Serialize to dict for JSON storage.
- `to_json() -> str` -- Serialize to JSON string.
- `from_dict(data) -> AuditEntry` -- Deserialize from dict.
- `from_json(line) -> AuditEntry` -- Deserialize from JSON string.
- `compute_chain_hash(entry, previous_hash) -> str` -- Compute SHA-256 chain hash.

---

## Classes

### `AuditLogger`

Persistent, tamper-evident audit logger for enterprise deployments.

#### Constructor

```python
AuditLogger(
    config: Optional[AuditConfig] = None,
    tenant_id: str = "default",
    callback: Optional[Callable[[AuditEntry], None]] = None,
    max_file_bytes: int = 50 * 1024 * 1024,
)
```

**Parameters**:

- `config` (`Optional[AuditConfig]`): Audit configuration. Defaults to `AuditConfig(enabled=True)`.
- `tenant_id` (`str`): Tenant identifier for this logger.
- `callback` (`Optional[Callable]`): Optional callback fired for every log entry.
- `max_file_bytes` (`int`): Maximum log file size before rotation. Default: 50 MB.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `tenant_id` | `str` | Tenant identifier |
| `enabled` | `bool` | Whether logging is enabled |
| `sequence` | `int` | Current sequence number |
| `last_hash` | `str` | Last hash in the chain |
| `entry_count` | `int` | Number of entries in memory buffer |

#### Methods

##### `log_event`

```python
async def log_event(
    self,
    event_type: str,
    details: Dict[str, Any],
    user_id: str = "",
    session_id: str = "",
    severity: str = "info",
    action: str = "",
    outcome: str = "success",
) -> Optional[AuditEntry]
```

Append a tamper-evident audit entry. Computes hash chain, stores in memory buffer, persists to file (if configured), and fires callback.

**Parameters**:

- `event_type` (`str`): Event type string (resolved to `AuditEventType`, falls back to `CUSTOM`).
- `details` (`Dict[str, Any]`): Structured payload.
- `user_id` (`str`): User who triggered the event.
- `session_id` (`str`): Session context.
- `severity` (`str`): Severity level (resolved to `AuditSeverity`, falls back to `INFO`).
- `action` (`str`): Short description. Defaults to `event_type` if empty.
- `outcome` (`str`): Result summary. Default: `"success"`.

**Returns**: `Optional[AuditEntry]` -- The created entry, or `None` if disabled/filtered.

##### `query_logs`

```python
async def query_logs(
    self,
    start: Optional[datetime] = None,
    end: Optional[datetime] = None,
    event_type: Optional[str] = None,
    user_id: Optional[str] = None,
    session_id: Optional[str] = None,
    severity: Optional[str] = None,
    limit: int = 1000,
) -> List[AuditEntry]
```

Query in-memory audit entries with filters. All parameters are optional; combine for targeted queries.

##### `export_logs`

```python
async def export_logs(
    self,
    format: str = "json",
    start: Optional[datetime] = None,
    end: Optional[datetime] = None,
) -> str
```

Export audit logs as JSONL or CSV string. Supports `"json"` and `"csv"` formats.

##### `verify_integrity`

```python
def verify_integrity(self) -> bool
```

Verify the SHA-256 hash chain integrity. SOC 2 CC7.2 compliance. Returns `True` if the entire chain is valid (no tampering detected).

##### `verify_integrity_detailed`

```python
def verify_integrity_detailed(self) -> Dict[str, Any]
```

Detailed integrity report. Returns a dict with:
- `valid` (`bool`): Whether the chain is intact.
- `total_entries` (`int`): Number of entries checked.
- `break_index` (`Optional[int]`): Index where tampering was detected (if any).
- `first_sequence` / `last_sequence`: Sequence number range.

##### `load_from_files`

```python
async def load_from_files(self, max_entries: int = 50_000) -> int
```

Load entries from disk into memory buffer. Returns the number of entries loaded.

##### `rotate_and_archive`

```python
async def rotate_and_archive(self) -> Dict[str, int]
```

Enforce retention policy on log files. Returns `{"archived": int, "deleted": int}`.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Get audit logger statistics including entry counts by type, retention settings, and hash chain state.

---

## Hash Chain Design

| Property | Implementation |
|----------|---------------|
| **Algorithm** | SHA-256 |
| **Genesis** | `"0" * 64` (64 zero characters) |
| **Chain** | `SHA256(previous_hash + ":" + entry_content)` |
| **Content** | `entry_id|tenant_id|session_id|user_id|timestamp|event_type|severity|action|details_json|outcome|sequence_num` |
| **Verification** | Recompute entire chain; any mismatch = tampering detected |

!!! tip "SOC 2 Compliance"
    The hash chain satisfies SOC 2 CC7.2 (System Operations -- Monitoring) by providing tamper-evident audit logs. Run `verify_integrity()` periodically to detect any log tampering.

---

## Example

```python
from corteX.enterprise.config import AuditConfig
from corteX.security.audit_logger import AuditLogger

# Create audit logger with file persistence
logger = AuditLogger(
    config=AuditConfig(
        enabled=True,
        log_tool_calls=True,
        log_model_routing=True,
        log_weight_changes=True,
        log_path="/var/log/cortex/audit",
        retention_days=365,
    ),
    tenant_id="acme",
)

# Log events
entry = await logger.log_event(
    event_type="tool_execution",
    details={"tool": "web_search", "query": "latest pricing"},
    user_id="user_42",
    session_id="sess_abc",
    severity="info",
    action="execute_tool",
)

# Log security event
await logger.log_event(
    event_type="security_block",
    details={"reason": "injection_detected", "input_hash": "abc123"},
    user_id="user_42",
    severity="critical",
    outcome="blocked",
)

# Query logs
from datetime import datetime, timedelta
entries = await logger.query_logs(
    start=datetime.now() - timedelta(hours=1),
    event_type="security_block",
    severity="critical",
)

# Export for compliance audit
csv_export = await logger.export_logs(format="csv")

# Verify hash chain integrity (SOC 2 CC7.2)
if not logger.verify_integrity():
    report = logger.verify_integrity_detailed()
    print(f"TAMPERING DETECTED at entry {report['break_index']}")

# Statistics
stats = logger.get_stats()
print(f"Total entries: {stats['total_entries']}")
print(f"Entries by type: {stats['entries_by_type']}")
```

---

## See Also

- [Audit Stream](../observability/audit-stream.md) -- Real-time audit event streaming
- [Compliance Engine](./compliance.md) -- Policy enforcement with audit logging
- [GDPR Manager](../enterprise/gdpr.md) -- DSAR lifecycle with audit trail
- [Retention Manager](../enterprise/retention.md) -- Audit log retention policies
