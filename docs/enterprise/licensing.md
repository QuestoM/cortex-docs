# Licensing

corteX uses per-seat licensing with Ed25519 cryptographic validation. License keys are self-contained signed tokens that work entirely offline.

## License Plans

| Plan | Seats | Features |
|------|-------|----------|
| **Starter** | 1 | Basic agents, weight system, goal tracking |
| **Professional** | Configurable | + Multi-model, tool framework, streaming, enterprise config |
| **Enterprise** | Configurable | + Audit logging, compliance, custom models, priority support |
| **Unlimited** | Unlimited | All features |

## License Key Format

License keys follow the format:

```
LK-<base64_payload>.<base64_signature>
```

Where:
- **Payload**: JSON containing `tenant_id`, `plan`, `seats`, `features`, `expires_at`, and `agents_per_seat`
- **Signature**: Ed25519 signature of the raw payload bytes
- **Validation**: The SDK embeds the public key; Questo signs with the private key

## Offline Validation

License keys contain all information needed for local validation. No network call is required:

1. Check the `LK-` prefix
2. Split payload and signature on the `.` separator
3. Base64-decode both parts
4. Verify the Ed25519 signature using the embedded public key
5. JSON-decode the payload and extract license fields
6. Check expiration with grace period

```python
from corteX.enterprise import LicenseManager, LicenseStatus

manager = LicenseManager(persistence_path="~/.cortex/license.json")
status = manager.activate("LK-eyJ0ZW5hb...")

if status == LicenseStatus.VALID:
    print("License activated successfully")
elif status == LicenseStatus.GRACE_PERIOD:
    print("License expired but within 30-day grace period")
elif status == LicenseStatus.EXPIRED:
    print("License expired beyond grace period")
```

## Grace Period

On-prem deployments get a **30-day offline grace period** after license expiration. This prevents service disruption when license renewal is delayed:

```python
# License statuses
LicenseStatus.VALID           # Within expiration date
LicenseStatus.GRACE_PERIOD    # Expired but within 30 days
LicenseStatus.EXPIRED         # Beyond grace period
LicenseStatus.INVALID         # Signature verification failed
LicenseStatus.NOT_ACTIVATED   # No license key provided
LicenseStatus.SEATS_EXCEEDED  # More active seats than allowed
```

## Seat Management

```python
manager = LicenseManager()
manager.activate("LK-...")

# Register a developer seat
success = manager.register_seat("developer_id_1")  # True if within limit

# Release a seat
manager.release_seat("developer_id_1")

# Check status
status = manager.check_status()
```

## Usage Metering

Usage is metered locally and stored on-prem. Sync to a license server is optional and can be completely disabled:

```python
# Record session usage
manager.record_session(tokens_used=5000, tool_calls=12)

# Get usage report
report = manager.get_usage_report()
# {
#   "active_seats": 3,
#   "today": {"sessions": 15, "tokens": 75000, "tool_calls": 180},
#   "history_days": 42,
#   "license_plan": "enterprise",
#   "status": "valid"
# }
```

Usage history is retained for 90 days and rolls over automatically.

## Feature Gating

Each plan unlocks specific features:

```python
from corteX.enterprise.config import LicenseConfig

license = LicenseConfig(plan="professional")
license.has_feature("multi_model")       # True
license.has_feature("audit_logging")     # False (enterprise+ only)
license.has_feature("basic_agents")      # True
```

The `unlimited` plan uses a wildcard (`"*"`) that enables all features.

## State Persistence

License state (activation, usage counters) persists to disk automatically:

```python
manager = LicenseManager(persistence_path="~/.cortex/license.json")
# State is loaded on init and saved after every operation
```
