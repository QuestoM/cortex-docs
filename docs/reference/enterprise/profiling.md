# Profiling Manager API Reference

## Module: `corteX.enterprise.profiling`

GDPR Art. 22 compliance -- profiling opt-out management. Gives data subjects the right not to be subject to decisions based solely on automated processing, including profiling. When opted out, synaptic weight personalization is disabled and behavioral pattern learning is skipped for that user.

Thread-safe via per-instance lock for concurrent access.

## Enums

### `ProfilingDecision`

**Type**: `str, Enum`

Reason for a profiling status change.

| Value | Description |
|-------|-------------|
| `user_request` | User explicitly opted in/out |
| `admin_request` | Administrator changed status |
| `gdpr_enforcement` | Automated GDPR enforcement |
| `consent_withdrawal` | Consent for profiling was withdrawn |
| `policy_default` | Tenant or system default policy |

---

## Data Classes

### `ProfilingStatus`

**Type**: `@dataclass`

Current profiling opt-out status for a user.

| Attribute | Type | Description |
|-----------|------|-------------|
| `status_id` | `str` | Unique ID (auto-generated UUID) |
| `tenant_id` | `str` | Tenant identifier |
| `user_id` | `str` | Data subject identifier |
| `opted_out` | `bool` | `True` if user has opted out of profiling |
| `timestamp` | `float` | When this status was set |
| `reason` | `str` | Human-readable reason for the change |
| `decision` | `ProfilingDecision` | What triggered the change |
| `previous_status` | `Optional[bool]` | Previous opt-out state (for audit) |

### `ProfilingGuardResult`

**Type**: `@dataclass`

Result of a profiling permission check with audit-ready explanation.

| Attribute | Type | Description |
|-----------|------|-------------|
| `allowed` | `bool` | Whether profiling is allowed |
| `tenant_id` | `str` | Tenant identifier |
| `user_id` | `str` | User identifier |
| `reason` | `str` | Human-readable explanation |
| `checked_at` | `float` | Timestamp of the check |

---

## Classes

### `ProfilingManager`

Manage GDPR Art. 22 profiling opt-out for all tenants.

#### Constructor

```python
ProfilingManager(*, default_opted_out: bool = False)
```

**Parameters**:

- `default_opted_out` (`bool`): Default profiling status for new users. `False` means profiling is allowed by default.

#### Methods

##### `opt_out`

```python
def opt_out(
    self, tenant_id: str, user_id: str, reason: str = "",
    decision: ProfilingDecision = ProfilingDecision.USER_REQUEST,
) -> ProfilingStatus
```

Opt a user out of automated profiling (GDPR Art. 22). Triggers any registered opt-out callbacks.

**Parameters**:

- `tenant_id` (`str`): Non-empty tenant identifier.
- `user_id` (`str`): Non-empty user identifier.
- `reason` (`str`): Human-readable reason for the opt-out.
- `decision` (`ProfilingDecision`): What triggered this change.

**Returns**: `ProfilingStatus` -- The new status record.

**Raises**: `ValueError` if `tenant_id` or `user_id` is empty.

##### `opt_in`

```python
def opt_in(
    self, tenant_id: str, user_id: str, reason: str = "",
    decision: ProfilingDecision = ProfilingDecision.USER_REQUEST,
) -> ProfilingStatus
```

Opt a user back in to automated profiling.

##### `get_status`

```python
def get_status(self, tenant_id: str, user_id: str) -> ProfilingStatus
```

Get current profiling status. Returns a default status if the user has never been explicitly set.

##### `is_profiling_allowed`

```python
def is_profiling_allowed(self, tenant_id: str, user_id: str) -> bool
```

Primary guard: `True` if profiling is allowed for this user. Checks tenant policy first, then user-level status, then falls back to default.

##### `check_profiling_allowed`

```python
def check_profiling_allowed(
    self, tenant_id: str, user_id: str,
) -> ProfilingGuardResult
```

Rich permission check with human-readable explanation for audit logging.

##### `set_tenant_policy`

```python
def set_tenant_policy(self, tenant_id: str, force_opt_out: bool) -> None
```

Force all users in a tenant to be opted out (or not). Tenant policy takes precedence over individual user settings.

##### `get_history`

```python
def get_history(self, tenant_id: str, user_id: str) -> List[ProfilingStatus]
```

Get full opt-in/opt-out history for a user. Returns a list of all `ProfilingStatus` records.

##### `get_opted_out_users`

```python
def get_opted_out_users(self, tenant_id: str) -> List[str]
```

List all user IDs that have opted out for a given tenant.

##### `on_opt_out`

```python
def on_opt_out(self, callback: Callable[[ProfilingStatus], None]) -> None
```

Register a callback fired when any user opts out. Use this to disable weight personalization.

##### `on_opt_in`

```python
def on_opt_in(self, callback: Callable[[ProfilingStatus], None]) -> None
```

Register a callback fired when any user opts back in.

##### `remove_user`

```python
def remove_user(self, tenant_id: str, user_id: str) -> bool
```

Remove all profiling data for a user (GDPR erasure support). Returns `True` if data existed.

---

## Example

```python
from corteX.enterprise.profiling import ProfilingManager, ProfilingDecision

pm = ProfilingManager()

# User opts out of profiling (GDPR Art. 22)
pm.opt_out("acme", "user_42", reason="User requested via settings")

# Guard check before personalizing weights
if not pm.is_profiling_allowed("acme", "user_42"):
    print("Skipping weight personalization")

# Rich check for audit logging
result = pm.check_profiling_allowed("acme", "user_42")
print(result.reason)
# "User opted out of profiling (reason: User requested via settings)"

# Tenant-wide policy: force all users opted out
pm.set_tenant_policy("acme", force_opt_out=True)

# Register callback to halt personalization
pm.on_opt_out(lambda s: disable_weights(s.tenant_id, s.user_id))

# User opts back in
pm.opt_in("acme", "user_42", reason="User re-enabled profiling")

# View history for audit
history = pm.get_history("acme", "user_42")
for status in history:
    print(f"  {status.decision.value}: opted_out={status.opted_out}")

# List all opted-out users
opted_out = pm.get_opted_out_users("acme")

# GDPR erasure
pm.remove_user("acme", "user_42")
```

---

## See Also

- [Consent Manager](./consent.md) -- Granular consent lifecycle
- [GDPR Manager](./gdpr.md) -- Full DSAR lifecycle
- [Explainability Engine](./explainability.md) -- Decision explanation (Art. 22 complement)
- [Compliance Engine](../security/compliance.md) -- Policy enforcement
