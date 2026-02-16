# Consent Manager API Reference

## Module: `corteX.enterprise.consent`

GDPR Art. 7 compliant consent lifecycle management. Provides `ConsentManager` for recording, withdrawing, querying, and exporting per-purpose consent for each tenant+user pair. Integrates with `ComplianceEngine` to automatically halt processing when consent is missing or withdrawn.

GDPR coverage: Art. 7(1) proof of consent, Art. 7(2) per-purpose granularity, Art. 7(3) easy withdrawal, Art. 7(4) granularity, Art. 20 export.

Thread-safe via internal lock for concurrent orchestrator access.

## Enums

### `ConsentPurpose`

**Type**: `str, Enum`

GDPR-aligned processing purposes. Each purpose maps to a specific category of data processing that requires independent, granular consent under GDPR Art. 6/7.

| Value | Description |
|-------|-------------|
| `personalization` | Agent behavior personalization via synaptic weights |
| `analytics` | Usage analytics and aggregated metrics |
| `learning` | Learning from user interactions to improve responses |
| `profiling` | Automated profiling and decision-making |
| `marketing` | Marketing communications and outreach |

---

### `ConsentMechanism`

**Type**: `str, Enum`

How consent was obtained (for audit/proof purposes per Art. 7(1)).

| Value | Description |
|-------|-------------|
| `explicit_opt_in` | Active, explicit opt-in by the user |
| `checkbox` | Checkbox-based consent (UI) |
| `signed_form` | Signed consent form |
| `api_call` | Consent recorded via API |
| `verbal` | Verbal consent (with documentation) |

---

### `ConsentStatus`

**Type**: `str, Enum`

Current status of a consent record.

| Value | Description |
|-------|-------------|
| `active` | Consent is currently valid |
| `withdrawn` | Consent has been withdrawn by the data subject |
| `expired` | Consent has expired (past `expires_at`) |

---

## Data Classes

### `ConsentRecord`

**Type**: `@dataclass`

A single consent grant or withdrawal record. Immutable once created; withdrawals produce new records.

| Attribute | Type | Description |
|-----------|------|-------------|
| `record_id` | `str` | Unique identifier (auto-generated UUID) |
| `tenant_id` | `str` | Enterprise customer tenant ID |
| `user_id` | `str` | Data subject who granted/withdrew consent |
| `purpose` | `ConsentPurpose` | Processing purpose consented to |
| `mechanism` | `ConsentMechanism` | How consent was collected |
| `policy_version` | `str` | Privacy policy version at time of consent |
| `status` | `ConsentStatus` | Current status |
| `granted_at` | `float` | Unix timestamp when consent was granted |
| `withdrawn_at` | `Optional[float]` | Unix timestamp when withdrawn (None if active) |
| `expires_at` | `Optional[float]` | Expiry timestamp (None = no expiry) |
| `metadata` | `dict` | Additional context (IP address, user-agent, etc.) |

#### Methods

##### `is_active`

```python
def is_active(self) -> bool
```

Check if this consent is currently valid and active. Returns `False` if status is not `ACTIVE` or if the consent has expired.

##### `to_dict`

```python
def to_dict(self) -> dict
```

Serialize to dictionary for storage/export.

##### `from_dict`

```python
@classmethod
def from_dict(cls, data: dict) -> ConsentRecord
```

Deserialize from dictionary.

---

## Classes

### `ConsentManager`

Manage consent lifecycle per tenant, user, and purpose. Accepts raw strings or enum values for `purpose` and `mechanism`.

#### Constructor

```python
ConsentManager()
```

No parameters. Creates a new thread-safe consent manager with empty state.

#### Methods

##### `record_consent`

```python
def record_consent(
    self,
    tenant_id: str,
    user_id: str,
    purpose: str,
    mechanism: str,
    policy_version: str,
    expires_at: Optional[float] = None,
    metadata: Optional[dict] = None,
) -> ConsentRecord
```

Record a new consent grant (GDPR Art. 7(1)).

**Parameters**:

- `tenant_id` (`str`): Tenant identifier.
- `user_id` (`str`): Data subject identifier.
- `purpose` (`str`): Processing purpose (must match a `ConsentPurpose` value).
- `mechanism` (`str`): How consent was obtained (must match a `ConsentMechanism` value).
- `policy_version` (`str`): Version of the privacy policy accepted.
- `expires_at` (`Optional[float]`): Unix timestamp for consent expiry. `None` for no expiry.
- `metadata` (`Optional[dict]`): Additional proof context (IP, user-agent, etc.).

**Returns**: `ConsentRecord` -- The recorded consent.

**Raises**: `ValueError` if `purpose` or `mechanism` is not a valid enum value.

##### `withdraw_consent`

```python
def withdraw_consent(
    self, tenant_id: str, user_id: str, purpose: str,
) -> bool
```

Withdraw consent for a specific purpose (GDPR Art. 7(3)). Triggers any registered withdrawal callbacks.

**Parameters**:

- `tenant_id` (`str`): Tenant identifier.
- `user_id` (`str`): Data subject identifier.
- `purpose` (`str`): Purpose to withdraw consent for.

**Returns**: `bool` -- `True` if withdrawn, `False` if no active consent found.

##### `check_consent`

```python
def check_consent(
    self, tenant_id: str, user_id: str, purpose: str,
) -> bool
```

Check whether active, non-expired consent exists. Automatically marks expired consents.

**Returns**: `bool` -- `True` if active consent exists.

##### `export_consents`

```python
def export_consents(
    self, tenant_id: str, user_id: str,
) -> List[ConsentRecord]
```

Export all consent records for a user (GDPR Art. 15/20 -- right of access / data portability). Records are ordered by `granted_at`.

**Returns**: `List[ConsentRecord]` -- Full consent history.

##### `get_active_consents`

```python
def get_active_consents(
    self, tenant_id: str, user_id: str,
) -> List[ConsentRecord]
```

Get only currently active, non-expired consents for a user. Automatically cleans up expired entries.

##### `check_all_purposes`

```python
def check_all_purposes(
    self, tenant_id: str, user_id: str,
) -> Dict[ConsentPurpose, bool]
```

Check consent status for every defined purpose. Returns a dict mapping each `ConsentPurpose` to `True`/`False`.

##### `withdraw_all`

```python
def withdraw_all(self, tenant_id: str, user_id: str) -> int
```

Withdraw all active consents for a user. Returns the number of consents withdrawn.

##### `on_withdrawal`

```python
def on_withdrawal(self, callback: Callable[[ConsentRecord], None]) -> None
```

Register a callback fired when any consent is withdrawn. Use this to halt downstream processing.

##### `has_consent_for_compliance`

```python
def has_consent_for_compliance(
    self, tenant_id: str, user_id: str, purpose: str,
) -> dict
```

Build a compliance context dict with `has_consent` resolved. Returns `{"has_consent": bool, "consent_purpose": str}` -- ready to pass to `ComplianceEngine.pre_check()`.

---

## Example

```python
from corteX.enterprise.consent import ConsentManager

mgr = ConsentManager()

# Record granular consent (GDPR Art. 7)
record = mgr.record_consent(
    tenant_id="acme",
    user_id="user_42",
    purpose="personalization",
    mechanism="explicit_opt_in",
    policy_version="2.1",
    metadata={"ip": "192.168.1.1", "ua": "Mozilla/5.0"},
)
print(record.record_id)  # UUID

# Check consent before processing
if mgr.check_consent("acme", "user_42", "personalization"):
    print("OK to personalize")

# Easy withdrawal (GDPR Art. 7(3))
mgr.withdraw_consent("acme", "user_42", "personalization")

# Check all purposes at once
status = mgr.check_all_purposes("acme", "user_42")
# {ConsentPurpose.PERSONALIZATION: False, ConsentPurpose.ANALYTICS: False, ...}

# Export for data portability (GDPR Art. 20)
history = mgr.export_consents("acme", "user_42")

# Register withdrawal callback to halt processing
mgr.on_withdrawal(lambda r: print(f"Consent withdrawn: {r.purpose.value}"))

# Build context for ComplianceEngine integration
ctx = mgr.has_consent_for_compliance("acme", "user_42", "analytics")
# {"has_consent": False, "consent_purpose": "analytics"}
```

---

## See Also

- [Compliance Engine](../security/compliance.md) -- Policy enforcement that checks consent
- [GDPR Manager](./gdpr.md) -- Full DSAR lifecycle (access, erasure, portability)
- [Profiling Manager](./profiling.md) -- GDPR Art. 22 automated decision opt-out
- [Retention Manager](./retention.md) -- Data retention enforcement
