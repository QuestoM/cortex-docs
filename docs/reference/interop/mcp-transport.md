# MCP Transport API Reference

## Module: `corteX.interop.mcp.client_transport`

Low-level transport layer for MCP communication. Provides `StdioTransport` (subprocess stdin/stdout) and `SSETransport` (HTTP+SSE) implementations over JSON-RPC 2.0.

---

## Classes

### `TransportMessage`

**Type**: `@dataclass`

A JSON-RPC 2.0 message used by the MCP protocol.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `jsonrpc` | `str` | `"2.0"` | JSON-RPC version |
| `method` | `Optional[str]` | `None` | RPC method name (present in requests) |
| `params` | `Optional[Dict[str, Any]]` | `None` | Request parameters |
| `result` | `Optional[Any]` | `None` | Success response payload |
| `error` | `Optional[Dict[str, Any]]` | `None` | Error response with `code` and `message` |
| `id` | `Optional[str]` | `None` | Request/response correlation ID |

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `is_response` | `bool` | `True` when message has `result` or `error` |
| `is_request` | `bool` | `True` when message has `method` and `id` |

#### Methods

##### `to_json`

```python
def to_json(self) -> str
```

Serialize to a JSON string, omitting `None` fields.

##### `from_json` *(classmethod)*

```python
@classmethod
def from_json(cls, raw: str) -> TransportMessage
```

Deserialize from a JSON string.

##### `request` *(classmethod)*

```python
@classmethod
def request(cls, method: str, params: Optional[Dict[str, Any]] = None) -> TransportMessage
```

Create a JSON-RPC request with a unique UUID id.

##### `response` *(classmethod)*

```python
@classmethod
def response(cls, request_id: str, result: Any) -> TransportMessage
```

Create a JSON-RPC success response.

##### `error_response` *(classmethod)*

```python
@classmethod
def error_response(cls, request_id: str, code: int, message: str) -> TransportMessage
```

Create a JSON-RPC error response.

#### Example

```python
from corteX.interop.mcp.client_transport import TransportMessage

# Create a request
req = TransportMessage.request("tools/list")
print(req.to_json())
# {"jsonrpc": "2.0", "method": "tools/list", "params": {}, "id": "abc-123"}

# Create a response
resp = TransportMessage.response(req.id, {"tools": []})
assert resp.is_response is True

# Parse from JSON
parsed = TransportMessage.from_json('{"jsonrpc":"2.0","result":{"ok":true},"id":"1"}')
assert parsed.result == {"ok": True}
```

---

### `MCPTransport`

**Type**: Abstract base class (`abc.ABC`)

Abstract base class for MCP transports.

#### Constructor

```python
MCPTransport(server_name: str, timeout: float = 30.0)
```

**Parameters**:

- `server_name` (`str`): Logical name of the MCP server.
- `timeout` (`float`): Connection/request timeout in seconds.

#### Abstract Methods

##### `connect`

```python
@abc.abstractmethod
async def connect(self) -> None
```

Establish the connection to the MCP server.

##### `disconnect`

```python
@abc.abstractmethod
async def disconnect(self) -> None
```

Gracefully shut down the connection.

##### `send_request`

```python
@abc.abstractmethod
async def send_request(
    self,
    method: str,
    params: Optional[Dict[str, Any]] = None,
) -> TransportMessage
```

Send a JSON-RPC request and return the response.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `connected` | `bool` | Whether the transport is currently connected |
| `server_name` | `str` | Logical name of the MCP server |

---

### `StdioTransport`

**Extends**: `MCPTransport`

Launch an MCP server as a child process and communicate via stdin/stdout. Uses newline-delimited JSON-RPC over the process's standard I/O streams.

#### Constructor

```python
StdioTransport(
    server_name: str,
    command: str,
    args: Optional[List[str]] = None,
    env: Optional[Dict[str, str]] = None,
    timeout: float = 30.0,
)
```

**Parameters**:

- `server_name` (`str`): Logical server name.
- `command` (`str`): Command to launch the MCP server (e.g. `"npx"`, `"python"`).
- `args` (`Optional[List[str]]`): Additional command arguments.
- `env` (`Optional[Dict[str, str]]`): Environment variables for the subprocess.
- `timeout` (`float`): Request timeout in seconds.

#### Methods

##### `connect`

```python
async def connect(self) -> None
```

Start the subprocess and send the MCP `initialize` handshake. The handshake sends protocol version `2024-11-05` and client info `{"name": "corteX", "version": "1.0.0"}`. After receiving the initialize response, sends a `notifications/initialized` notification.

Spawns a background reader task to dispatch responses from stdout.

**Raises**: `Exception` -- If the initialize handshake fails (disconnects automatically).

##### `disconnect`

```python
async def disconnect(self) -> None
```

Cancel pending futures, cancel the reader task, and terminate the subprocess. Waits up to 5 seconds for graceful termination before killing the process.

##### `send_request`

```python
async def send_request(
    self,
    method: str,
    params: Optional[Dict[str, Any]] = None,
) -> TransportMessage
```

Write a JSON-RPC request to stdin and wait for the matching response from stdout.

**Parameters**:

- `method` (`str`): JSON-RPC method name.
- `params` (`Optional[Dict[str, Any]]`): Method parameters.

**Returns**: `TransportMessage` -- The matching response.

**Raises**:

- `RuntimeError` -- If transport is not connected.
- `TimeoutError` -- If no response arrives within the timeout period.

#### Example

```python
from corteX.interop.mcp.client_transport import StdioTransport

transport = StdioTransport(
    server_name="filesystem",
    command="npx",
    args=["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
    timeout=15.0,
)

await transport.connect()

# List available tools
resp = await transport.send_request("tools/list")
tools = resp.result.get("tools", [])

# Call a tool
resp = await transport.send_request("tools/call", {
    "name": "read_file",
    "arguments": {"path": "/tmp/data.txt"},
})

await transport.disconnect()
```

---

### `SSETransport`

**Extends**: `MCPTransport`

Connect to an MCP server over HTTP + Server-Sent Events (SSE).

Requires `aiohttp` to be installed (`pip install aiohttp`).

#### Constructor

```python
SSETransport(
    server_name: str,
    url: str,
    headers: Optional[Dict[str, str]] = None,
    timeout: float = 30.0,
)
```

**Parameters**:

- `server_name` (`str`): Logical server name.
- `url` (`str`): SSE endpoint URL (e.g. `https://mcp.example.com/sse`).
- `headers` (`Optional[Dict[str, str]]`): HTTP headers (e.g. auth tokens).
- `timeout` (`float`): Request timeout in seconds.

#### Methods

##### `connect`

```python
async def connect(self) -> None
```

Create an `aiohttp.ClientSession` and perform the MCP initialize handshake. Derives the POST message endpoint from the SSE URL:

- URLs ending in `/sse` use `/message` instead
- Other URLs append `/message`

**Raises**:

- `ImportError` -- If `aiohttp` is not installed.
- `Exception` -- If the initialize handshake fails.

##### `disconnect`

```python
async def disconnect(self) -> None
```

Close the HTTP session.

##### `send_request`

```python
async def send_request(
    self,
    method: str,
    params: Optional[Dict[str, Any]] = None,
) -> TransportMessage
```

POST a JSON-RPC request to the message endpoint and return the response.

**Parameters**:

- `method` (`str`): JSON-RPC method name.
- `params` (`Optional[Dict[str, Any]]`): Method parameters.

**Returns**: `TransportMessage` -- Parsed response. HTTP errors (status >= 400) are returned as `TransportMessage` error responses.

**Raises**:

- `RuntimeError` -- If transport is not connected.
- `TimeoutError` -- If the request times out.

#### Example

```python
from corteX.interop.mcp.client_transport import SSETransport

transport = SSETransport(
    server_name="api-tools",
    url="https://mcp.example.com/sse",
    headers={"Authorization": "Bearer sk-..."},
    timeout=30.0,
)

await transport.connect()

resp = await transport.send_request("tools/list")
tools = resp.result.get("tools", [])

await transport.disconnect()
```

---

## See Also

- [MCP Client](./mcp-client.md) -- High-level client that uses transports internally
- [MCP Tool Bridge](./mcp-tool-bridge.md) -- Tool schema conversion
- [Types](./types.md) -- `MCPServerConfig` with transport selection
