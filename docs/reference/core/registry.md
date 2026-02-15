# Plugin Registry API Reference

## Module: `corteX.core.registry`

The `PluginRegistry` is the per-instance repository for loaded plugins (agents, subsystems, memory drivers). Each registry instance maintains its own isolated dictionaries, ensuring that plugins registered in one tenant/engine scope are invisible to others in the same process.

This is a critical component of corteX's multi-tenant isolation architecture.

## Class

### `PluginRegistry`

Per-instance plugin repository with tenant isolation.

#### Constructor

```python
PluginRegistry(
    *,
    allowed_plugin_roots: Optional[List[str]] = None,
    tenant_id: Optional[str] = None,
)
```

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `allowed_plugin_roots` | `Optional[List[str]]` | `None` | Filesystem directories from which `load_from_directory()` may import plugins. If empty, only the built-in corteX plugins directory is allowed. This is a security control. |
| `tenant_id` | `Optional[str]` | `"default"` | Unique identifier for this registry's tenant. Used in logging and module naming for dynamic imports. |

#### Example

```python
from corteX.core.registry import PluginRegistry

# Isolated registry for a specific tenant
registry = PluginRegistry(
    tenant_id="acme_corp",
    allowed_plugin_roots=["/opt/acme/plugins"],
)
```

---

#### Instance Methods

##### `register_plugin`

```python
def register_plugin(self, plugin_cls: Type[T]) -> Type[T]
```

Register a plugin class into this instance's storage. The plugin is automatically categorized based on its type:

- `IAgent` subclass -> stored in agents dict
- `ISubsystem` subclass -> stored in subsystems dict
- `IMemoryDriver` subclass -> stored in drivers dict

**Parameters**:

- `plugin_cls` (`Type[T]`): The plugin class to register (must implement `IPlugin`).

**Returns**: The same class (allows use as a decorator).

**Example**:

```python
from corteX.core.contracts import ISubsystem

# Register directly
registry.register_plugin(MySubsystem)

# Or use as a decorator
@registry.register_plugin
class AnotherSubsystem(ISubsystem):
    ...
```

##### `load_from_directory`

```python
def load_from_directory(self, directory: str) -> None
```

Dynamically import `.py` files from a directory to trigger plugin registration.

**Security Controls**:

- The resolved directory must be inside an allowed root (see `allowed_plugin_roots`) or the built-in corteX plugins path.
- If the directory is not in an allowed root, the load is **blocked** and a security error is logged.
- Every import attempt is logged at INFO level for audit.
- Uses `importlib.util.spec_from_file_location` to avoid mutating `sys.path`.

**Parameters**:

- `directory` (`str`): Path to the directory containing plugin `.py` files.

**Example**:

```python
registry = PluginRegistry(
    tenant_id="acme",
    allowed_plugin_roots=["/opt/acme/plugins"],
)

# This works -- directory is inside allowed root
registry.load_from_directory("/opt/acme/plugins/custom")

# This is BLOCKED -- not in allowed roots
registry.load_from_directory("/tmp/malicious_plugins")
# Logs: [SECURITY] load_from_directory BLOCKED for /tmp/malicious_plugins
```

##### `get_agent`

```python
def get_agent(self, name: str) -> Type[IAgent]
```

Get an agent class by name. Raises `KeyError` if not found.

##### `get_subsystem`

```python
def get_subsystem(self, name: str) -> Optional[Type[ISubsystem]]
```

Get a subsystem class by name. Returns `None` if not found.

##### `get_all_subsystems`

```python
def get_all_subsystems(self) -> List[str]
```

Return names of all registered subsystems.

##### `instantiate_subsystem`

```python
def instantiate_subsystem(
    self,
    name: str,
    config: Dict[str, Any],
) -> ISubsystem
```

Instantiate a registered subsystem by name and call its `initialize()` method with the provided config.

**Parameters**:

- `name` (`str`): Subsystem class name.
- `config` (`Dict[str, Any]`): Configuration passed to `initialize()`.

**Returns**: An initialized `ISubsystem` instance.

**Raises**: `ValueError` if the subsystem name is not found.

**Example**:

```python
subsystem = registry.instantiate_subsystem(
    "WebResearchSubsystem",
    config={"max_results": 10, "timeout": 30},
)
result = await subsystem.execute("Research AI frameworks", context)
```

##### `import_from`

```python
def import_from(self, source: PluginRegistry) -> None
```

Copy all registrations from another registry into this one. Useful when a tenant wants to start with the default (built-in) plugins and then add their own on top.

**Parameters**:

- `source` (`PluginRegistry`): The registry to copy from.

**Example**:

```python
# Start with built-in plugins, then add tenant-specific ones
tenant_registry = PluginRegistry(tenant_id="acme")
tenant_registry.import_from(PluginRegistry.get_default())
tenant_registry.load_from_directory("/opt/acme/plugins")
```

##### `clear`

```python
def clear(self) -> None
```

Remove all registrations. Useful in tests.

---

#### Class Methods (Legacy / Global Registry)

These class methods delegate to the module-level default `PluginRegistry` instance for backward compatibility.

##### `PluginRegistry.register`

```python
@classmethod
def register(cls, plugin_cls: Type[T]) -> Type[T]
```

Decorator that registers into the default (global) registry.

**Example**:

```python
from corteX.core.registry import PluginRegistry
from corteX.core.contracts import ISubsystem

@PluginRegistry.register
class MySubsystem(ISubsystem):
    ...
```

##### `PluginRegistry.get_default`

```python
@classmethod
def get_default(cls) -> PluginRegistry
```

Return the module-level default registry instance.

---

## Integration with Engine

The `Engine` class creates a per-engine `PluginRegistry` that starts with built-in plugins and can be extended:

```python
import cortex

engine = cortex.Engine(
    providers={"openai": {"api_key": "sk-..."}},
    tenant_id="acme_corp",
    allowed_plugin_roots=["/opt/acme/plugins"],
)

# The engine has its own isolated registry
print(engine.plugin_registry.get_all_subsystems())

# Load additional plugins at runtime
engine.plugin_registry.load_from_directory("/opt/acme/plugins/v2")
```

---

## Security Model

The PluginRegistry enforces a strict security model for dynamic plugin loading:

1. **Path Allowlist**: Only directories explicitly listed in `allowed_plugin_roots` (or the built-in corteX plugins directory) are permitted.

2. **Module Isolation**: Dynamically loaded modules use unique names prefixed with `_cortex_plugin_{tenant_id}` to prevent collisions across tenants.

3. **Audit Logging**: Every dynamic import is logged at INFO level with the module name, directory, and tenant ID.

4. **Blocked Loads**: Attempts to load from unauthorized directories are logged at ERROR level with `[SECURITY]` prefix and silently ignored (no exception raised to prevent information leakage).

```
# Example audit log output:
INFO  [registry] Dynamic import: module=_cortex_plugin_acme.my_tool, dir=/opt/acme/plugins, tenant=acme
ERROR [SECURITY] load_from_directory BLOCKED for /tmp/evil (not in allowed roots, tenant=acme)
```

---

## Module-Level Instances

```python
# Default registry used by @PluginRegistry.register and legacy API
_default_registry = PluginRegistry(tenant_id="global-default")
```

---

## See Also

- [Contracts API Reference](./contracts.md) - Plugin interfaces (IPlugin, ISubsystem, IAgent)
- [EventBus API Reference](./events.md) - Event system
- [Multi-Tenant Setup](../../enterprise/multi-tenant.md) - Enterprise tenant isolation
- [Security & Isolation](../../enterprise/security.md) - Security architecture
