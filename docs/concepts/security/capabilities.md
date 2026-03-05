# Capability-Based Security

corteX replaces traditional role-based access control with
capability-based security - immutable permission tokens that can only
shrink, never grow. A `CapabilitySet` is created once and can be
attenuated (reduced) for sub-agents, enforcing the principle of least
privilege at every delegation boundary.

## How It Works

Each `Capability` is a frozen dataclass representing a single
permission grant with four attributes:

- **resource** - what the capability applies to (e.g., `"tool:search_db"`,
  `"model:gpt-4"`, `"memory:write"`)
- **actions** - a frozen set of allowed operations (e.g.,
  `{"read", "execute"}`)
- **constraints** - additional limits like `{"max_calls": 10}` or
  `{"domains": ["*.acme.com"]}`
- **expires_at** - optional UTC timestamp for automatic expiry

A `CapabilitySet` wraps multiple capabilities into an immutable
collection. The critical invariant is that capabilities can only be
removed through `attenuate()` - you can never add new capabilities
after creation. When checking permissions with `has(resource, action)`,
expired capabilities are automatically treated as absent.

## Key Features

- **Immutable tokens** - `Capability` uses `frozen=True` dataclass; cannot be modified after creation
- **Monotonic attenuation** - `attenuate()` only accepts subsets of existing capabilities
- **Automatic sub-agent delegation** - `for_sub_agent()` strips `write`, `delete`, and `admin` actions
- **Time-based expiry** - capabilities can auto-expire via `expires_at` timestamp
- **Serialization** - full `to_dict()`/`from_dict()` support for persistence and transport
- **Silent drop on violation** - requesting a capability not covered by the parent set is silently dropped with a warning log

## Integration

The capability system sits between the orchestrator and every tool or
model call. Before executing any action, the runtime checks
`capability_set.has(resource, action)`. Sub-agents spawned during
execution receive an automatically attenuated copy via
`for_sub_agent()`, which removes all write, delete, and admin
permissions.

The Risk Attenuator dynamically narrows capabilities further based on
computed risk scores. The Compliance Engine can mandate additional
capability restrictions based on active regulatory frameworks.

## Usage Example

```python
from corteX.security.capabilities import Capability, CapabilitySet

# Define capabilities for an agent
caps = CapabilitySet([
    Capability(
        resource="tool:search_db",
        actions=frozenset({"read", "execute"}),
    ),
    Capability(
        resource="memory:working",
        actions=frozenset({"read", "write", "delete"}),
    ),
    Capability(
        resource="model:gpt-4",
        actions=frozenset({"execute"}),
        constraints={"max_calls": 50},
    ),
])

# Check permissions
assert caps.has("tool:search_db", "read")      # True
assert not caps.has("tool:search_db", "write")  # False

# Attenuate for a sub-agent (strips write/delete/admin)
sub_caps = caps.for_sub_agent()
assert sub_caps.has("memory:working", "read")      # True
assert not sub_caps.has("memory:working", "write")  # Stripped

# Manual attenuation with specific subset
narrow = caps.attenuate([
    Capability(
        resource="tool:search_db",
        actions=frozenset({"read"}),  # Only read, no execute
    ),
])
assert narrow.count == 1
```

## See Also

- [Risk-Based Attenuation](risk-attenuation.md) - dynamic capability reduction based on risk
- [Key Vault](key-vault.md) - secure credential storage
- [Compliance Engine](compliance-engine.md) - regulatory policy enforcement
