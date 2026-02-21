# MCP -- Model Context Protocol

The Model Context Protocol (MCP) is an open standard for connecting AI agents to external tool servers. corteX's MCP integration lets your agent discover and use tools from any MCP-compliant server -- file systems, databases, web browsers, custom APIs -- through a single, uniform interface.

## How MCP Works

An MCP server is a process that exposes tools (and optionally resources) over a standardized protocol. The corteX `MCPClientManager` connects to these servers, discovers their tools, and wraps each one as a `ToolWrapper` that integrates seamlessly with the agentic loop.

```
corteX Session
    |
    v
MCPClientManager
    |
    +-- MCPServerConnection ("filesystem")
    |       tools: [mcp__filesystem__read_file, mcp__filesystem__write_file, ...]
    |       resources: [file:///data/config.json, ...]
    |
    +-- MCPServerConnection ("database")
            tools: [mcp__database__query, mcp__database__insert, ...]
```

## Tool Namespacing

MCP tools are namespaced to prevent collisions between servers. A tool named `read_file` on a server named `filesystem` becomes:

```
mcp__filesystem__read_file
```

The naming convention is `mcp__{server_name}__{tool_name}`. This ensures that if two servers both expose a tool called `query`, the agent (and the LLM) can distinguish between them.

The `parse_tool_name()` function splits a namespaced name back into its components:

```python
from corteX.interop.mcp.tool_bridge import parse_tool_name

server, tool = parse_tool_name("mcp__filesystem__read_file")
# server = "filesystem", tool = "read_file"
```

## Transport Modes

corteX supports two MCP transport modes:

### stdio Transport

The server runs as a subprocess. corteX launches the process, communicates over stdin/stdout, and manages the lifecycle.

```python
from corteX.interop.types import MCPServerConfig

config = MCPServerConfig(
    name="filesystem",
    transport="stdio",
    command="npx",
    args=["-y", "@modelcontextprotocol/server-filesystem", "/data"],
    env={"NODE_ENV": "production"},
    timeout=30.0,
)
```

Best for: local tool servers, development, air-gapped environments.

### SSE Transport

The server runs as an HTTP service. corteX connects via Server-Sent Events for real-time communication.

```python
config = MCPServerConfig(
    name="remote-tools",
    transport="sse",
    url="https://tools.internal:8443/mcp",
    headers={"Authorization": "Bearer token-..."},
    timeout=30.0,
)
```

Best for: remote servers, shared infrastructure, production deployments.

## Tool Discovery

When corteX connects to an MCP server, it automatically:

1. Calls `list_tools()` to get available tools and their JSON Schema definitions
2. Wraps each tool as a `ToolWrapper` with the namespaced name
3. Optionally calls `list_resources()` to discover available resources

The discovered tools appear alongside native tools in the agent's tool list. The LLM sees them with their full descriptions and schemas, and can call them naturally during the agentic loop.

## Resource Discovery

MCP servers can also expose resources -- data endpoints with URIs that the agent can read. Resources are discovered during connection and stored on the connection object:

```python
# After connecting
conn = mcp_client.get_connection("filesystem")
for resource in conn.resources:
    print(f"{resource['uri']} - {resource['description']}")
```

Resources are metadata-only at discovery time. The agent reads them through the server's resource-reading tools.

## Execution Flow

When the LLM decides to call an MCP tool:

1. The agentic loop receives a tool call like `mcp__filesystem__read_file`
2. `MCPClientManager.execute_tool()` is called
3. The tool name is parsed to identify the server (`filesystem`) and tool (`read_file`)
4. A **capability check** verifies `mcp:filesystem:read_file` has `execute` permission
5. The tool's `ToolWrapper.execute()` calls the MCP server via the active `ClientSession`
6. The result is normalized to a string via `mcp_result_to_string()`
7. The string result is returned to the agentic loop for the LLM

## Security via CapabilitySet

Every MCP tool execution passes through corteX's capability system. The resource name follows the pattern `mcp:{server}:{tool}`, and the action is `execute`:

```python
from corteX.security.capabilities import CapabilitySet

caps = CapabilitySet()
caps.grant("mcp:filesystem:read_file", "execute")
caps.grant("mcp:filesystem:list_directory", "execute")
# mcp:filesystem:write_file is NOT granted -- blocked

agent = engine.create_agent(
    name="reader",
    mcp_servers=[filesystem_config],
    # capability_set is wired through session initialization
)
```

If the agent tries to call a tool that is not in the capability set, a `PermissionError` is raised. The agent never silently accesses tools outside its granted permissions.

## Connection Management

The `MCPClientManager` handles connection lifecycle:

- **`connect_all()`** -- connects to all configured servers in parallel, returns `{server: bool}` status
- **`disconnect_all()`** -- gracefully disconnects all servers
- **`connect_server(config)`** -- connects a single server
- **`disconnect_server(name)`** -- disconnects a single server
- **Auto-reconnect** -- configurable via `reconnect` and `max_reconnect_attempts` on `MCPServerConfig`

## Manual Tool Registration

If the `mcp` pip package is not installed, or if you want to register tools from external schemas (e.g., from a cached tool list), you can use `register_tools()`:

```python
mcp_client.register_tools("filesystem", [
    {
        "name": "read_file",
        "description": "Read the contents of a file",
        "inputSchema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "File path to read"}
            },
            "required": ["path"],
        },
    },
])
```

This creates `ToolWrapper` instances without a live server connection. The wrappers will raise `RuntimeError` if executed without an active client -- useful for testing, schema validation, and offline development.

## Result Normalization

MCP servers return results in various formats. The `mcp_result_to_string()` function normalizes all of them to plain strings:

| MCP Result Format | Handling |
|---|---|
| `{"content": [{"type": "text", "text": "..."}]}` | Concatenates text parts |
| `{"text": "..."}` | Returns text directly |
| `{"result": "..."}` | Returns result directly |
| Object with `.content` attribute | Extracts text from parts |
| Plain string | Returned as-is |

## Key Classes

| Class | Module | Purpose |
|---|---|---|
| `MCPServerConfig` | `corteX.interop.types` | Server connection configuration |
| `MCPClientManager` | `corteX.interop.mcp.client` | Connection + tool management |
| `MCPServerConnection` | `corteX.interop.mcp.client` | Per-server state tracking |
| `ToolWrapper` | `corteX.tools.decorator` | Unified tool representation |
| `SessionInteropMixin` | `corteX.session.interop` | Wires MCP into session lifecycle |

## See Also

- [Protocol Interoperability](index.md) -- Overview and comparison with A2A
- [Connect to MCP Servers](../../guides/interop/mcp-servers.md) -- Step-by-step how-to guide
- [A2A -- Agent-to-Agent Protocol](a2a.md) -- Agent delegation protocol
