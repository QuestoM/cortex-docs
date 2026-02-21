# MCP Resource Bridge API Reference

## Module: `corteX.interop.mcp.resource_bridge`

Converts MCP resources into context strings for LLM injection. Resources are cached per server so they can be listed, fetched, and formatted without repeated MCP calls.

---

## Classes

### `ResourceInfo`

**Type**: `@dataclass`

Metadata about a single MCP resource.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `uri` | `str` | *(required)* | Resource URI (e.g. `file:///data/config.json`) |
| `name` | `str` | *(required)* | Human-readable resource name |
| `description` | `str` | `""` | Resource description |
| `mime_type` | `str` | `"text/plain"` | MIME type of the resource content |

---

### `MCPResourceBridge`

Converts MCP resources into context strings for LLM injection.

Resources are cached per server so they can be listed, fetched, and formatted without repeated MCP calls.

#### Constructor

```python
MCPResourceBridge()
```

No arguments. Creates an empty resource cache.

#### Methods

##### `register_resources`

```python
def register_resources(
    self,
    server_name: str,
    resources: List[Dict[str, Any]],
) -> None
```

Register discovered resources for a server.

**Parameters**:

- `server_name` (`str`): Name of the MCP server that owns these resources.
- `resources` (`List[Dict[str, Any]]`): List of resource dicts. Each dict should have at least `uri` and `name` keys. Optional keys: `description`, `mimeType` (or `mime_type`).

**Example**:

```python
from corteX.interop.mcp.resource_bridge import MCPResourceBridge

bridge = MCPResourceBridge()
bridge.register_resources("filesystem", [
    {
        "uri": "file:///data/config.json",
        "name": "App Configuration",
        "description": "Main application config",
        "mimeType": "application/json",
    },
    {
        "uri": "file:///data/readme.md",
        "name": "README",
        "mimeType": "text/markdown",
    },
])
```

##### `list_available`

```python
def list_available(
    self,
    server_name: Optional[str] = None,
) -> List[ResourceInfo]
```

List registered resources, optionally filtered by server.

**Parameters**:

- `server_name` (`Optional[str]`): Filter by server name. When `None`, returns resources from all servers.

**Returns**: `List[ResourceInfo]` -- List of resource metadata objects.

**Example**:

```python
# All resources
all_resources = bridge.list_available()

# Resources from a specific server
fs_resources = bridge.list_available("filesystem")
for r in fs_resources:
    print(f"{r.name}: {r.uri} ({r.mime_type})")
```

##### `get_as_context`

```python
def get_as_context(
    self,
    server_name: str,
    uri: str,
    content: str,
) -> str
```

Format a resource's content for LLM context injection. Returns a string with a header block identifying the source, followed by the resource content.

**Parameters**:

- `server_name` (`str`): Name of the MCP server.
- `uri` (`str`): Resource URI.
- `content` (`str`): The actual resource content to format.

**Returns**: `str` -- Formatted context block.

**Example**:

```python
context = bridge.get_as_context(
    "filesystem",
    "file:///data/config.json",
    '{"debug": true, "port": 8080}',
)
print(context)
```

Output:

```
--- MCP Resource: App Configuration ---
Server: filesystem
URI: file:///data/config.json
Type: application/json
Description: Main application config
---
{"debug": true, "port": 8080}
```

##### `clear`

```python
def clear(self, server_name: Optional[str] = None) -> None
```

Clear cached resources.

**Parameters**:

- `server_name` (`Optional[str]`): When provided, only that server's resources are removed. Otherwise all resources are cleared.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `server_count` | `int` | Number of servers with registered resources |
| `total_resources` | `int` | Total number of cached resources across all servers |

---

## Integration with MCPClientManager

The `MCPClientManager` automatically discovers resources during connection and can register them with the bridge:

```python
from corteX.interop.mcp.client import MCPClientManager
from corteX.interop.mcp.resource_bridge import MCPResourceBridge

manager = MCPClientManager(configs)
await manager.connect_all()

bridge = MCPResourceBridge()

# Register discovered resources for each server
for name in manager.connected_servers:
    conn = manager.get_connection(name)
    if conn and conn.resources:
        bridge.register_resources(name, conn.resources)

# List and format resources for LLM context
for resource in bridge.list_available():
    # Fetch content via MCP and inject as context
    context = bridge.get_as_context(
        "filesystem",
        resource.uri,
        "<fetched content>",
    )
```

---

## See Also

- [MCP Client](./mcp-client.md) -- Discovers resources during connection
- [MCP Tool Bridge](./mcp-tool-bridge.md) -- Tool conversion (companion module)
- [Types](./types.md) -- `MCPServerConfig` dataclass
