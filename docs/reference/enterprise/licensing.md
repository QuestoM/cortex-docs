# License Manager API Reference

## Module: `corteX.enterprise.licensing`

Per-seat license manager with Ed25519 cryptographic validation. Supports both online and air-gapped on-prem deployments with offline grace periods. No cloud dependency required.

## License Key Format

```
LK-<base64_payload>.<base64_signature>
```

- **Payload**: JSON containing `tenant_id`, `plan`, `seats`, `features`, `expires_at`, `agents_per_seat`
- **Signature**: Ed25519 signature of the raw payload bytes
- **Validation**: The SDK embeds the public key; Questo signs with the private key

## Enums

### `LicenseStatus`

```python
class LicenseStatus(str, Enum)
```

| Value | Description |
|-------|-------------|
| `VALID` | License is active and valid |
| `EXPIRED` | License has expired beyond the grace period |
| `GRACE_PERIOD` | License expired but within 30-day offline grace |
| `INVALID` | License key is malformed or signature failed |
| `NOT_ACTIVATED` | No license has been activated |
| `SEATS_EXCEEDED` | Active seats exceed licensed maximum |

---

## Data Classes

### `LicenseInfo`

**Type**: `@dataclass`

Decoded license information extracted from a validated license key.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `license_key` | `str` | Original license key string |
| `organization_id` | `str` | Tenant/organization identifier |
| `plan` | `str` | Plan: `"starter"`, `"professional"`, `"enterprise"`, `"unlimited"` |
| `max_seats` | `int` | Maximum developer seats |
| `max_agents_per_seat` | `int` | Maximum agents per seat |
| `features` | `List[str]` | Enabled feature strings |
| `issued_at` | `float` | Unix timestamp of activation |
| `expires_at` | `float` | Unix timestamp of expiration |
| `signature` | `str` | Base64-encoded signature |

### `UsageMeter`

**Type**: `@dataclass`

Local usage metering stored on-prem, optionally synced.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `active_seats` | `int` | `0` | Currently active developer seats |
| `active_agents` | `int` | `0` | Currently active agents |
| `total_sessions_today` | `int` | `0` | Sessions today |
| `total_tokens_today` | `int` | `0` | Tokens consumed today |
| `total_tool_calls_today` | `int` | `0` | Tool calls today |
| `last_reset_date` | `str` | `""` | Date of last daily reset (YYYY-MM-DD) |
| `history` | `List[Dict]` | `[]` | Last 90 days of daily usage stats |

---

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `OFFLINE_GRACE_DAYS` | `30` | Days after expiry before enforcement |
| `SECONDS_PER_DAY` | `86400` | Seconds in a day |

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `CORTEX_LICENSE_PUBLIC_KEY_B64` | Base64-encoded Ed25519 public key for verification |
| `CORTEX_DEV_PRIVATE_KEY_B64` | Base64-encoded Ed25519 private key (dev/testing only) |

---

## Classes

### `LicenseManager`

Manages SDK licensing for on-prem deployment. License keys contain all info needed for offline validation. Usage metering works entirely offline. Sync is optional.

#### Constructor

```python
LicenseManager(persistence_path: Optional[str] = None)
```

**Parameters**:

- `persistence_path` (`Optional[str]`): File path for persisting license state. If provided, state is saved/loaded automatically

#### Methods

##### `activate`

```python
def activate(self, license_key: str) -> LicenseStatus
```

Activate a license key. Validates the key format, verifies the Ed25519 signature, extracts license info, and checks expiry.

**Validation steps**:

1. Check `LK-` prefix
2. Split payload and signature on `.` separator
3. Base64-decode both parts
4. Verify Ed25519 signature against embedded public key
5. JSON-decode the payload
6. Validate required fields (tenant_id, plan, seats, features, expires_at)
7. Check seat count > 0
8. Check expiry (with grace period)

**Returns**: `LicenseStatus` after activation.

**Example**:

```python
from corteX.enterprise.licensing import LicenseManager, LicenseStatus

manager = LicenseManager(persistence_path="~/.cortex/license.json")
status = manager.activate("LK-eyJ0ZW5hbnRfaWQiOi...sig")

if status == LicenseStatus.VALID:
    print("License activated successfully")
elif status == LicenseStatus.GRACE_PERIOD:
    print("License expired, operating in grace period")
```

##### `check_status`

```python
def check_status(self) -> LicenseStatus
```

Check current license status. Checks expiry, grace period, and seat limits.

##### `register_seat`

```python
def register_seat(self, developer_id: str) -> bool
```

Register a new developer seat. Returns `True` if within the licensed limit.

##### `release_seat`

```python
def release_seat(self, developer_id: str) -> None
```

Release a developer seat.

##### `record_session`

```python
def record_session(self, tokens_used: int = 0, tool_calls: int = 0) -> None
```

Record session usage for metering. Automatically resets daily counters and archives yesterday's stats (keeps last 90 days).

##### `get_usage_report`

```python
def get_usage_report(self) -> Dict[str, Any]
```

Get usage report for the current period.

**Returns**:

```python
{
    "active_seats": 3,
    "active_agents": 12,
    "today": {"sessions": 45, "tokens": 125000, "tool_calls": 230},
    "history_days": 30,
    "license_plan": "enterprise",
    "status": "valid",
}
```

##### `get_license_info`

```python
def get_license_info(self) -> Optional[Dict[str, Any]]
```

Get current license info (non-sensitive fields). Returns `None` if no license is activated.

---

## Functions

### `generate_license_key`

```python
def generate_license_key(
    tenant_id: str,
    plan: str,
    seats: int,
    features: List[str],
    expires_at: float,
    agents_per_seat: int = 50,
    private_key_b64: Optional[str] = None,
) -> str
```

Generate a signed license key. This is a utility for development/testing. In production, license keys are generated server-side by Questo's licensing service.

**Parameters**:

- `tenant_id` (str): Organization identifier
- `plan` (str): License plan
- `seats` (int): Maximum developer seats
- `features` (`List[str]`): Enabled feature strings
- `expires_at` (float): Unix timestamp for expiration
- `agents_per_seat` (int, default=50): Max agents per seat
- `private_key_b64` (`Optional[str]`): Ed25519 private key. Falls back to `CORTEX_DEV_PRIVATE_KEY_B64` env var

**Returns**: License key string in format `LK-<payload>.<signature>`.

---

## See Also

- [Enterprise Licensing Guide](../../enterprise/licensing.md)
- [Enterprise Config API](./config.md)
- [On-Premises Deployment](../../enterprise/on-prem.md)
