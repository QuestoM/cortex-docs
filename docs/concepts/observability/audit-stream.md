# Tenant Audit Stream

The Tenant Audit Stream provides a per-tenant, tamper-evident,
hash-chained audit log suitable for SOC 2 CC7.2, GDPR Art. 30, and
legal proceedings. Like hippocampal replay creating immutable episodic
memories, each entry is cryptographically linked to its predecessor
so that tampering is immediately detectable.

## How It Works

The `TenantAuditStream` maintains an append-only log of
`EnterpriseAuditEntry` objects. Each entry is SHA-256 chained to its
predecessor using the formula:

```
chain_hash = SHA-256(previous_chain_hash + ":" + entry_content)
```

The first entry chains from a genesis hash of 64 zeros. When a new
entry is appended, the stream computes its hash from the previous
entry's hash and the new entry's serialized content (event ID, tenant,
session, user, timestamp, event type, action, justification, outcome,
tokens, data level, and capabilities used).

Eight event types cover the full agent lifecycle:

| Event Type | Description |
|------------|-------------|
| `LLM_CALL` | Model invocation |
| `TOOL_EXECUTION` | Tool usage |
| `POLICY_DECISION` | Security/compliance decision |
| `GOAL_EVENT` | Goal creation, completion, or failure |
| `SESSION_START` | Agent session begins |
| `SESSION_END` | Agent session ends |
| `DATA_ACCESS` | Sensitive data accessed |
| `CONFIG_CHANGE` | Configuration modification |

## Key Features

- **SHA-256 hash chain** - each entry links to its predecessor; tampering breaks the chain
- **Chain verification** - `verify_chain()` walks the entire chain and reports the exact index of any break
- **Rich entry schema** - event ID, tenant/session/user IDs, timestamps, justification, outcome, cost, data level, capabilities
- **Flexible querying** - filter by event type, session, user, time range, action, data level, with limit
- **Dual export formats** - JSONL for machine processing, CSV for spreadsheets and compliance tools
- **Bounded storage** - configurable max entries (default 50,000) with FIFO eviction
- **Per-tenant isolation** - each tenant gets its own stream with independent hash chain
- **Statistics** - `get_stats()` returns entry counts by type and last hash prefix

## Integration

The Audit Stream is written to by the orchestrator, compliance engine,
and security components. Every LLM call, tool execution, and policy
decision generates an audit entry. The Compliance Engine's `LOG`
actions are fulfilled by appending to the audit stream.

For SOC 2 audits, the stream can be exported as JSONL and the hash
chain verified independently. For GDPR, the stream supports the
right to erasure through `clear()` while maintaining chain integrity
verification before clearance.

## Usage Example

```python
from corteX.observability.audit_stream import (
    TenantAuditStream, EnterpriseAuditEntry, AuditEventType,
)

stream = TenantAuditStream(tenant_id="acme-corp", max_entries=10_000)

# Append audit entries
entry = stream.append(EnterpriseAuditEntry(
    session_id="sess-001",
    user_id="user-42",
    event_type=AuditEventType.LLM_CALL,
    action="generate_report",
    justification="User requested quarterly summary",
    outcome="success",
    cost_tokens=2500,
    data_level="confidential",
    capabilities_used=["model:gemini-2.5-pro", "tool:search_db"],
))
# entry.chain_hash is now set (SHA-256 linked to genesis)

# Verify chain integrity
is_valid, break_index = stream.verify_chain()
assert is_valid  # No tampering detected

# Query with filters
recent_llm = stream.query({
    "event_type": "llm_call",
    "session_id": "sess-001",
    "limit": 50,
})

# Export for compliance
jsonl_lines = stream.export(format="jsonl")
csv_lines = stream.export(format="csv")

# Statistics
stats = stream.get_stats()
# {"tenant_id": "acme-corp", "total_entries": 1, "entries_by_type": {"llm_call": 1}, ...}
```

## See Also

- [Compliance Engine](../security/compliance-engine.md) - drives audit logging requirements
- [Decision Tracer](decision-tracer.md) - traces decision reasoning (complementary to audit)
- [Metrics Collector](metrics-collector.md) - quantitative metrics for monitoring
- [Data Classification](../security/data-classification.md) - provides data level for audit entries
