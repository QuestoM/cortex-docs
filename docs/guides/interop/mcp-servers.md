# Connect to MCP Servers

This guide walks you through connecting a corteX agent to MCP (Model Context Protocol) servers so it can discover and use external tools.

## Prerequisites

- corteX SDK installed: `pip install cortex-ai`
- (Optional) MCP Python library for live connections: `pip install mcp`
- An MCP-compliant server to connect to

## Step 1: Configure an MCP Server

Create an `MCPServerConfig` for each server you want to connect to.

### stdio Transport (Local Server)

Use stdio when the MCP server runs as a local subprocess:

```python
from corteX.interop.types import MCPServerConfig

filesystem_server = MCPServerConfig(
    name="filesystem",
    transport="stdio",
    command="npx",
    args=["-y", "@modelcontextprotocol/server-filesystem", "/data"],
    timeout=30.0,
)
```

| Parameter | Required | Description |
|---|---|---|
| `name` | Yes | Server identifier (used in tool namespacing) |
| `transport` | No | `"stdio"` (default) or `"sse"` |
| `command` | Yes (stdio) | Command to launch the server process |
| `args` | No | Additional command-line arguments |
| `env` | No | Environment variables for the subprocess |
| `timeout` | No | Connection timeout in seconds (default: 30) |
| `reconnect` | No | Auto-reconnect on connection loss (default: True) |
| `max_reconnect_attempts` | No | Max reconnect attempts (default: 3) |

### SSE Transport (Remote Server)

Use SSE when the MCP server runs as an HTTP service:

```python
remote_server = MCPServerConfig(
    name="database",
    transport="sse",
    url="https://mcp-tools.internal:8443/sse",
    headers={"Authorization": "Bearer sk-token-..."},
    timeout=30.0,
)
```

| Parameter | Required | Description |
|---|---|---|
| `url` | Yes (sse) | HTTP+SSE endpoint URL |
| `headers` | No | HTTP headers (e.g., auth tokens) |

## Step 2: Create an Agent with MCP Servers

Pass your server configs to `create_agent()`:

```python
import cortex

engine = cortex.Engine(providers={"openai": {"api_key": "sk-..."}})

agent = engine.create_agent(
    name="assistant",
    system_prompt="You are a helpful assistant with file system access.",
    mcp_servers=[filesystem_server, remote_server],
)
```

## Step 3: Connect and Discover Tools

When you start a session, call `connect_interop()` to establish connections and discover tools:

```python
session = agent.start_session(user_id="user_123")

# Connect to all MCP servers and discover tools
status = await session.connect_interop()
print(status)
# {"mcp": {"filesystem": True, "database": True}, "a2a": {}}
```

After connection, MCP tools are automatically merged into the agent's tool list with namespaced names:

- `mcp__filesystem__read_file`
- `mcp__filesystem__write_file`
- `mcp__filesystem__list_directory`
- `mcp__database__query`
- `mcp__database__insert`

## Step 4: Use the Agent Normally

The agent can now use MCP tools just like native tools. The LLM sees all tools (native + MCP) and calls them naturally:

```python
response = await session.run("List all files in the /data/reports directory")
# The agent will call mcp__filesystem__list_directory automatically
```

## Step 5: Disconnect When Done

Gracefully disconnect from all servers when your session is complete:

```python
await session.disconnect_interop()
```

## Checking Connection Status

You can check whether MCP is connected at any time:

```python
if session.mcp_connected:
    print("MCP servers are connected")

# Get the list of tools from a specific server
tools = session._mcp_client.get_tools_for_server("filesystem")
for tool in tools:
    print(f"  {tool.name}: {tool.description}")
```

## Multiple Servers

You can connect to as many MCP servers as you need. Each server's tools are namespaced independently:

```python
agent = engine.create_agent(
    name="power-assistant",
    system_prompt="You have access to files, databases, and web browsing.",
    mcp_servers=[
        MCPServerConfig(
            name="filesystem",
            transport="stdio",
            command="npx",
            args=["-y", "@modelcontextprotocol/server-filesystem", "/data"],
        ),
        MCPServerConfig(
            name="postgres",
            transport="stdio",
            command="npx",
            args=["-y", "@modelcontextprotocol/server-postgres"],
            env={"DATABASE_URL": "postgresql://localhost/mydb"},
        ),
        MCPServerConfig(
            name="browser",
            transport="sse",
            url="https://browser-mcp.internal:8080/sse",
        ),
    ],
)
```

## Manual Tool Registration

If the `mcp` pip package is not installed (or you want to register tools from a cached schema), use `register_tools()` on the MCP client:

```python
session._mcp_client.register_tools("filesystem", [
    {
        "name": "read_file",
        "description": "Read the contents of a file",
        "inputSchema": {
            "type": "object",
            "properties": {
                "path": {"type": "string", "description": "File path"}
            },
            "required": ["path"],
        },
    },
])
```

This is useful for testing, development, and environments where the MCP library cannot be installed.

## Error Handling

### Connection Failures

`connect_all()` connects servers in parallel and never raises on individual failures. Check the returned status dict:

```python
status = await session.connect_interop()
for server, connected in status["mcp"].items():
    if not connected:
        conn = session._mcp_client.get_connection(server)
        print(f"Server '{server}' failed: {conn.error}")
```

### Tool Execution Errors

MCP tool execution errors are raised as exceptions:

```python
try:
    result = await session._execute_interop_tool(
        "mcp__filesystem__read_file",
        {"path": "/nonexistent/file.txt"},
    )
except ValueError as e:
    print(f"Tool not found or server not connected: {e}")
except PermissionError as e:
    print(f"Capability check failed: {e}")
```

### Missing MCP Library

If the `mcp` pip package is not installed, servers are still registered but no tools are discovered automatically. Install it for full functionality:

```bash
pip install mcp
```

## Security Considerations

- MCP tools are subject to CapabilitySet checks -- configure capabilities to restrict which tools each tenant can access
- Use `headers` on SSE transport for authentication tokens
- Use `env` on stdio transport to pass credentials without hardcoding them
- Never expose MCP servers on public networks without authentication

## See Also

- [Protocol Interoperability](../../concepts/interop/index.md) -- Overview of MCP and A2A
- [MCP -- Model Context Protocol](../../concepts/interop/mcp.md) -- Concept deep dive
- [Delegate Tasks to A2A Agents](a2a-agents.md) -- The other interop protocol
