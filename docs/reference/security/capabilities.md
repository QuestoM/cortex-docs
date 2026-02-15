# Capability Set API Reference

## Module: `corteX.security.capabilities`

Capability-based security with immutable permission tokens. Replaces role-based access with fine-grained, attenuatable capability sets. A `CapabilitySet` can only shrink (via `attenuate`) -- it can never gain new powers after creation. Sub-agents automatically receive an attenuated copy that strips write actions.

## Classes

### `Capability`

**Type**: `@dataclass(frozen=True)`

A single, immutable permission grant.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `resource` | `str` | Resource this capability applies to (e.g., `"tool:search_db"`, `"model:gpt-4"`, `"memory:write"`) |
| `actions` | `FrozenSet[str]` | Allowed actions (e.g., `frozenset({"read", "execute"})`) |
| `constraints` | `Dict[str, Any]` | Additional limits (e.g., `{"max_calls": 10, "domains": ["*.acme.com"]}`) |
| `expires_at` | `Optional[float]` | UTC timestamp after which this capability is invalid. `None` = no expiry. |

#### Methods

##### `to_dict`

```python
def to_dict(self) -> Dict[str, Any]
```

Serialize to a plain dict with `resource`, `actions` (sorted list), `constraints`, and optional `expires_at`.

##### `from_dict`

```python
@classmethod
def from_dict(cls, data: Dict[str, Any]) -> Capability
```

Deserialize from a plain dict.

---

### `CapabilitySet`

An immutable set of `Capability` objects. Supports attenuation (shrink-only) and automatic sub-agent delegation.

#### Constructor

```python
CapabilitySet(capabilities: Optional[List[Capability]] = None)
```

**Parameters**:

- `capabilities` (`Optional[List[Capability]]`): Initial capabilities. Stored as an immutable tuple.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `count` | `int` | Number of capabilities in this set |

#### Methods

##### `has`

```python
def has(self, resource: str, action: str) -> bool
```

Check if this set grants an action on a resource. Expired capabilities are treated as absent.

**Parameters**:

- `resource` (`str`): Resource to check (e.g., `"tool:search_db"`).
- `action` (`str`): Action to check (e.g., `"execute"`).

**Returns**: `bool`

##### `get_capabilities`

```python
def get_capabilities(self, resource: Optional[str] = None) -> List[Capability]
```

Return capabilities, optionally filtered by resource. Returns all if `resource` is `None`.

##### `is_expired`

```python
@staticmethod
def is_expired(cap: Capability, now: Optional[float] = None) -> bool
```

Check whether a capability has expired.

##### `attenuate`

```python
def attenuate(self, subset: List[Capability]) -> CapabilitySet
```

Create a reduced capability set. Each capability in `subset` must be a subset (same resource, subset of actions) of an existing capability. Non-subset capabilities are silently dropped.

**Parameters**:

- `subset` (`List[Capability]`): Requested reduced capabilities.

**Returns**: `CapabilitySet` -- New set with at most the powers of `self`.

##### `for_sub_agent`

```python
def for_sub_agent(self) -> CapabilitySet
```

Auto-attenuate for sub-agent delegation. Removes `write`, `delete`, and `admin` actions from every capability, keeping only `read` and `execute`. Expired capabilities are excluded.

**Returns**: `CapabilitySet` -- Attenuated set safe for sub-agents.

##### `to_dict` / `from_dict`

```python
def to_dict(self) -> Dict[str, Any]

@classmethod
def from_dict(cls, data: Dict[str, Any]) -> CapabilitySet
```

Serialize/deserialize the full capability set.

---

## Attenuation Rules

| Operation | Allowed? | Description |
|-----------|----------|-------------|
| Remove actions | Yes | `{read, write}` -> `{read}` |
| Remove capabilities | Yes | Drop entire capability |
| Add actions | No | Cannot add `write` to `{read}` |
| Add capabilities | No | Cannot add new resource grants |
| Extend expiry | No | Cannot extend or remove expiration |

---

## Sub-Agent Delegation

When `for_sub_agent()` is called, the following actions are automatically stripped:

- `write` -- Cannot modify state
- `delete` -- Cannot remove resources
- `admin` -- Cannot change configuration

Only `read` and `execute` survive. This enforces the principle of least privilege.

---

## Example

```python
from corteX.security.capabilities import Capability, CapabilitySet

# Create capabilities for a tenant
caps = CapabilitySet([
    Capability(
        resource="tool:search_db",
        actions=frozenset({"read", "execute"}),
    ),
    Capability(
        resource="tool:file_write",
        actions=frozenset({"read", "write", "execute"}),
    ),
    Capability(
        resource="model:gpt-4",
        actions=frozenset({"read", "execute"}),
        constraints={"max_calls": 100},
    ),
])

# Check permissions
assert caps.has("tool:search_db", "execute")
assert not caps.has("tool:search_db", "admin")

# Attenuate for a specific task
reduced = caps.attenuate([
    Capability(resource="tool:search_db", actions=frozenset({"read"})),
])
assert reduced.count == 1
assert reduced.has("tool:search_db", "read")
assert not reduced.has("tool:file_write", "write")

# Auto-attenuate for sub-agent
sub_caps = caps.for_sub_agent()
assert not sub_caps.has("tool:file_write", "write")
assert sub_caps.has("tool:file_write", "read")

# Serialize
data = caps.to_dict()
restored = CapabilitySet.from_dict(data)
```

---

## See Also

- [Risk Attenuator](./attenuation.md) -- Risk-based automatic attenuation
- [Key Vault](./vault.md) -- API key storage scoped by capabilities
- [Compliance Engine](./compliance.md) -- Policy enforcement alongside capabilities
