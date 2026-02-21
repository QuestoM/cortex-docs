# Interop Types API Reference

## Module: `corteX.interop.types`

Configuration dataclasses for MCP and A2A interoperability. These are the developer-facing config objects passed to `Engine.create_agent()`.

---

## Classes

### `MCPServerConfig`

**Type**: `@dataclass`

Configuration for connecting to a single MCP server.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | *(required)* | Human-readable server identifier (used as tool namespace) |
| `transport` | `Literal["stdio", "sse"]` | `"stdio"` | Connection method |
| `command` | `Optional[str]` | `None` | For stdio transport: command to launch the server process |
| `args` | `List[str]` | `[]` | For stdio transport: additional command arguments |
| `url` | `Optional[str]` | `None` | For SSE transport: the HTTP+SSE endpoint URL |
| `env` | `Dict[str, str]` | `{}` | Environment variables for stdio subprocess |
| `headers` | `Dict[str, str]` | `{}` | HTTP headers for SSE transport (e.g. auth tokens) |
| `capabilities` | `Optional[Dict[str, Any]]` | `None` | Capability restrictions for this server's tools |
| `timeout` | `float` | `30.0` | Connection timeout in seconds |
| `reconnect` | `bool` | `True` | Whether to auto-reconnect on connection loss |
| `max_reconnect_attempts` | `int` | `3` | Max reconnect attempts |

#### Validation

- `transport="stdio"` requires `command` to be set -- raises `ValueError` otherwise
- `transport="sse"` requires `url` to be set -- raises `ValueError` otherwise

#### Methods

##### `to_dict`

```python
def to_dict(self) -> Dict[str, Any]
```

Serialize to a plain dict. Only includes non-empty optional fields.

**Returns**: `Dict[str, Any]` -- JSON-serializable dictionary.

##### `from_dict` *(classmethod)*

```python
@classmethod
def from_dict(cls, data: Dict[str, Any]) -> MCPServerConfig
```

Deserialize from a plain dict.

**Parameters**:

- `data` (`Dict[str, Any]`): Dictionary with config values. Only `name` is required; all others have defaults.

**Returns**: `MCPServerConfig`

#### Example

```python
from corteX.interop.types import MCPServerConfig

# Stdio server (e.g. filesystem tools)
fs_server = MCPServerConfig(
    name="filesystem",
    transport="stdio",
    command="npx",
    args=["-y", "@modelcontextprotocol/server-filesystem", "/data"],
    timeout=15.0,
)

# SSE server (e.g. remote API)
api_server = MCPServerConfig(
    name="api-tools",
    transport="sse",
    url="https://mcp.example.com/sse",
    headers={"Authorization": "Bearer sk-..."},
)

# Roundtrip serialization
data = fs_server.to_dict()
restored = MCPServerConfig.from_dict(data)
assert restored.name == "filesystem"
```

---

### `A2AAgentConfig`

**Type**: `@dataclass`

Configuration for connecting to an external A2A agent.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | *(required)* | Human-readable agent identifier (used as delegation target) |
| `url` | `str` | *(required)* | Base URL of the A2A agent (Agent Card at `/.well-known/agent.json`) |
| `description` | `str` | `""` | Description of what this agent does |
| `skills` | `List[str]` | `[]` | Skill IDs this agent provides |
| `auth_token` | `Optional[str]` | `None` | Bearer token for authentication |
| `headers` | `Dict[str, str]` | `{}` | Additional HTTP headers |
| `timeout` | `float` | `60.0` | Request timeout in seconds |
| `max_retries` | `int` | `2` | Max retry attempts on failure |

#### Validation

- `url` must be non-empty -- raises `ValueError` otherwise.

#### Methods

##### `get_auth_headers`

```python
def get_auth_headers(self) -> Dict[str, str]
```

Build HTTP headers including the bearer token if present.

**Returns**: `Dict[str, str]` -- Headers dict with `Authorization: Bearer <token>` added when `auth_token` is set.

##### `to_dict`

```python
def to_dict(self) -> Dict[str, Any]
```

Serialize to plain dict. **Excludes `auth_token` for safety** -- never serialized to prevent accidental credential exposure.

**Returns**: `Dict[str, Any]`

##### `from_dict` *(classmethod)*

```python
@classmethod
def from_dict(cls, data: Dict[str, Any]) -> A2AAgentConfig
```

Deserialize from a plain dict.

**Parameters**:

- `data` (`Dict[str, Any]`): Dictionary with config values. `name` and `url` are required.

**Returns**: `A2AAgentConfig`

#### Example

```python
from corteX.interop.types import A2AAgentConfig

agent = A2AAgentConfig(
    name="summarizer",
    url="https://summarizer.example.com",
    description="Summarizes long documents",
    skills=["summarize", "extract-key-points"],
    auth_token="sk-secret-token",
    timeout=120.0,
)

# Auth headers include the bearer token
headers = agent.get_auth_headers()
# {"Authorization": "Bearer sk-secret-token"}

# Serialization excludes auth_token
data = agent.to_dict()
assert "auth_token" not in data
```

---

### `InteropConfig`

**Type**: `@dataclass`

Combined interoperability configuration for an agent. This is the top-level config object passed to `Engine.create_agent()`.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `mcp_servers` | `List[MCPServerConfig]` | `[]` | MCP servers to connect to |
| `a2a_agents` | `List[A2AAgentConfig]` | `[]` | External A2A agents for delegation |
| `auto_connect` | `bool` | `True` | Whether to automatically connect on session start |

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `has_mcp` | `bool` | Whether any MCP servers are configured |
| `has_a2a` | `bool` | Whether any A2A agents are configured |
| `is_empty` | `bool` | Whether no interop is configured at all |

#### Methods

##### `to_dict`

```python
def to_dict(self) -> Dict[str, Any]
```

Serialize the full config tree to a plain dict.

**Returns**: `Dict[str, Any]`

##### `from_dict` *(classmethod)*

```python
@classmethod
def from_dict(cls, data: Dict[str, Any]) -> InteropConfig
```

Deserialize the full config tree from a plain dict. Automatically constructs nested `MCPServerConfig` and `A2AAgentConfig` objects.

**Parameters**:

- `data` (`Dict[str, Any]`): Dictionary with `mcp_servers`, `a2a_agents`, and `auto_connect` keys.

**Returns**: `InteropConfig`

#### Example

```python
from corteX.interop.types import InteropConfig, MCPServerConfig, A2AAgentConfig

config = InteropConfig(
    mcp_servers=[
        MCPServerConfig(name="fs", transport="stdio", command="npx",
                        args=["-y", "@modelcontextprotocol/server-filesystem", "/"]),
    ],
    a2a_agents=[
        A2AAgentConfig(name="writer", url="https://writer.example.com"),
    ],
    auto_connect=True,
)

assert config.has_mcp is True
assert config.has_a2a is True
assert config.is_empty is False

# Full roundtrip
data = config.to_dict()
restored = InteropConfig.from_dict(data)
assert len(restored.mcp_servers) == 1
assert len(restored.a2a_agents) == 1
```

---

## See Also

- [Interop Overview](./index.md) -- Package overview and architecture
- [MCP Client](./mcp-client.md) -- Uses `MCPServerConfig` to manage connections
- [A2A Client](./a2a-client.md) -- Uses `A2AAgentConfig` to manage agent discovery
