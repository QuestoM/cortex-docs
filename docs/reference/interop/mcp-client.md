# MCP Client Manager API Reference

## Module: `corteX.interop.mcp.client`

Manages connections to multiple MCP servers. Each server connection discovers tools and exposes them as `ToolWrapper` instances with namespaced names: `mcp__{server_name}__{tool_name}`.

---

## Classes

### `MCPServerConnection`

**Type**: `@dataclass`

Tracks the state of a connection to a single MCP server.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `config` | `MCPServerConfig` | *(required)* | The server configuration |
| `tools` | `Dict[str, ToolWrapper]` | `{}` | Discovered tools keyed by namespaced name |
| `resources` | `List[Dict[str, Any]]` | `[]` | Discovered MCP resources |
| `connected` | `bool` | `False` | Whether the connection is active |
| `error` | `Optional[str]` | `None` | Error message if connection failed |

---

### `MCPClientManager`

Manages connections to multiple MCP servers, tool discovery, capability enforcement, and tool execution.

#### Constructor

```python
MCPClientManager(
    configs: List[MCPServerConfig],
    capability_set: Optional[CapabilitySet] = None,
)
```

**Parameters**:

- `configs` (`List[MCPServerConfig]`): List of MCP server configurations to manage.
- `capability_set` (`Optional[CapabilitySet]`): Optional capability set for permission checks on tool execution. When provided, each `execute_tool` call verifies `mcp:{server}:{tool}` has the `execute` permission.

#### Methods

##### `connect_all`

```python
async def connect_all(self) -> Dict[str, bool]
```

Connect to every configured server in parallel. Each server performs the MCP `initialize` handshake, discovers tools and resources.

**Returns**: `Dict[str, bool]` -- Mapping of `{server_name: success}`. Failed connections are logged and recorded with their error.

**Example**:

```python
from corteX.interop.types import MCPServerConfig
from corteX.interop.mcp.client import MCPClientManager

configs = [
    MCPServerConfig(name="fs", transport="stdio", command="npx",
                    args=["-y", "@modelcontextprotocol/server-filesystem", "/tmp"]),
    MCPServerConfig(name="api", transport="sse",
                    url="https://mcp.example.com/sse"),
]

manager = MCPClientManager(configs)
results = await manager.connect_all()
# {"fs": True, "api": True}
```

##### `connect_server`

```python
async def connect_server(self, config: MCPServerConfig) -> bool
```

Connect to a single MCP server and discover its tools.

When the `mcp` library is not installed, the connection is marked as ready but no tools are discovered -- they can be registered later via `register_tools`.

**Parameters**:

- `config` (`MCPServerConfig`): Server configuration.

**Returns**: `bool` -- `True` if connection succeeded.

##### `disconnect_all`

```python
async def disconnect_all(self) -> None
```

Disconnect from all servers and clear state. Closes MCP client sessions and terminates subprocesses.

##### `disconnect_server`

```python
async def disconnect_server(self, name: str) -> None
```

Disconnect a specific server by name. Clears its tools and resources.

**Parameters**:

- `name` (`str`): Server name to disconnect.

##### `get_tools`

```python
def get_tools(self) -> List[ToolWrapper]
```

Return all discovered tools across every connected server.

**Returns**: `List[ToolWrapper]` -- Flat list of all available MCP tools.

##### `get_tools_for_server`

```python
def get_tools_for_server(self, name: str) -> List[ToolWrapper]
```

Return tools from a specific server.

**Parameters**:

- `name` (`str`): Server name.

**Returns**: `List[ToolWrapper]` -- Tools from the named server, or empty list if not connected.

##### `execute_tool`

```python
async def execute_tool(
    self,
    tool_name: str,
    arguments: Dict[str, Any],
) -> str
```

Execute an MCP tool by its full namespaced name (`mcp__{server}__{tool}`).

**Parameters**:

- `tool_name` (`str`): Full namespaced tool name.
- `arguments` (`Dict[str, Any]`): Tool arguments as key-value pairs.

**Returns**: `str` -- Tool execution result as a string.

**Raises**:

- `ValueError` -- If `tool_name` is not an MCP tool, the server is not connected, or the tool is not found.
- `PermissionError` -- If the capability set denies execution for this tool.

**Example**:

```python
result = await manager.execute_tool(
    "mcp__fs__read_file",
    {"path": "/tmp/data.txt"},
)
print(result)  # File contents
```

##### `register_tools`

```python
def register_tools(
    self,
    server_name: str,
    tool_schemas: List[Dict[str, Any]],
) -> None
```

Register tools from an external schema list. Creates `ToolWrapper` instances via the tool bridge. If the server was not previously connected, a new connection record is created.

**Parameters**:

- `server_name` (`str`): Logical server name for namespacing.
- `tool_schemas` (`List[Dict[str, Any]]`): List of MCP tool definitions with `name`, `description`, and `inputSchema` keys.

**Example**:

```python
manager.register_tools("custom", [
    {
        "name": "greet",
        "description": "Say hello",
        "inputSchema": {
            "type": "object",
            "properties": {
                "name": {"type": "string", "description": "Who to greet"},
            },
        },
    },
])
```

##### `get_connection`

```python
def get_connection(self, name: str) -> Optional[MCPServerConnection]
```

Return the connection record for a server (or `None` if not found).

**Parameters**:

- `name` (`str`): Server name.

**Returns**: `Optional[MCPServerConnection]`

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `connected_servers` | `List[str]` | Names of currently connected servers |
| `available_tool_count` | `int` | Total number of tools across all connected servers |

---

## Capability Enforcement

When a `CapabilitySet` is provided, every `execute_tool` call checks:

```python
resource = f"mcp:{server_name}:{tool_name}"
capability_set.has(resource, "execute")
```

If the check fails, `PermissionError` is raised before any tool invocation.

```python
from corteX.security.capabilities import CapabilitySet
from corteX.interop.mcp.client import MCPClientManager

caps = CapabilitySet()
caps.grant("mcp:fs:read_file", "execute")
# Only read_file is allowed; write_file will raise PermissionError

manager = MCPClientManager(configs, capability_set=caps)
```

---

## See Also

- [Types](./types.md) -- `MCPServerConfig` dataclass
- [MCP Tool Bridge](./mcp-tool-bridge.md) -- Tool conversion utilities
- [MCP Transport](./mcp-transport.md) -- Low-level transport layer
- [MCP Resource Bridge](./mcp-resource-bridge.md) -- Resource context injection
