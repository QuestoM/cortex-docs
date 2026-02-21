# MCP Tool Bridge API Reference

## Module: `corteX.interop.mcp.tool_bridge`

Converts between MCP tool schemas and corteX `ToolWrapper` instances. Handles name prefixing, schema normalization, and result formatting.

---

## Name Prefixing Scheme

All MCP tools are namespaced to prevent collisions across servers:

```
mcp__{server_name}__{tool_name}
```

For example, a tool named `read_file` on server `filesystem` becomes `mcp__filesystem__read_file`.

---

## Functions

### `make_tool_name`

```python
def make_tool_name(server_name: str, tool_name: str) -> str
```

Build a namespaced MCP tool name.

**Parameters**:

- `server_name` (`str`): Logical server name.
- `tool_name` (`str`): Original tool name from the MCP server.

**Returns**: `str` -- Namespaced name in the format `mcp__{server}__{tool}`.

**Example**:

```python
from corteX.interop.mcp.tool_bridge import make_tool_name

name = make_tool_name("filesystem", "read_file")
# "mcp__filesystem__read_file"
```

---

### `parse_tool_name`

```python
def parse_tool_name(prefixed_name: str) -> Tuple[str, str]
```

Parse `mcp__server__tool` into `(server_name, tool_name)`.

Returns `("", prefixed_name)` when the name is not an MCP tool (no `mcp__` prefix or invalid format).

**Parameters**:

- `prefixed_name` (`str`): The namespaced tool name to parse.

**Returns**: `Tuple[str, str]` -- `(server_name, tool_name)` or `("", original_name)`.

**Example**:

```python
from corteX.interop.mcp.tool_bridge import parse_tool_name

server, tool = parse_tool_name("mcp__filesystem__read_file")
# ("filesystem", "read_file")

server, tool = parse_tool_name("my_local_tool")
# ("", "my_local_tool")
```

---

### `is_mcp_tool`

```python
def is_mcp_tool(name: str) -> bool
```

Return `True` when the name uses the `mcp__` namespace prefix.

**Parameters**:

- `name` (`str`): Tool name to check.

**Returns**: `bool`

**Example**:

```python
from corteX.interop.mcp.tool_bridge import is_mcp_tool

is_mcp_tool("mcp__fs__read_file")   # True
is_mcp_tool("my_custom_tool")        # False
```

---

### `mcp_tool_to_wrapper`

```python
def mcp_tool_to_wrapper(
    tool_schema: Dict[str, Any],
    server_name: str,
    execute_fn: Optional[Callable] = None,
) -> ToolWrapper
```

Convert an MCP tool schema dict to a corteX `ToolWrapper`.

**Parameters**:

- `tool_schema` (`Dict[str, Any]`): MCP tool definition with `name`, `description`, and `inputSchema` keys.
- `server_name` (`str`): Logical MCP server name (used for namespacing).
- `execute_fn` (`Optional[Callable]`): Async callable `(**kwargs) -> str` that invokes the tool via MCP. When `None`, a placeholder that raises `RuntimeError` is used.

**Returns**: `ToolWrapper` -- A corteX tool wrapper with the namespaced name, description, and JSON Schema parameters.

**Schema Normalization**: The input schema is normalized to ensure it is a valid JSON Schema object:

- Missing `type` defaults to `"object"`
- Missing `properties` defaults to `{}`
- Empty schema becomes `{"type": "object", "properties": {}}`

**Example**:

```python
from corteX.interop.mcp.tool_bridge import mcp_tool_to_wrapper

schema = {
    "name": "read_file",
    "description": "Read contents of a file",
    "inputSchema": {
        "type": "object",
        "properties": {
            "path": {"type": "string", "description": "File path"},
        },
        "required": ["path"],
    },
}

async def execute_read(**kwargs):
    return f"Contents of {kwargs['path']}"

wrapper = mcp_tool_to_wrapper(schema, "fs", execute_read)
# wrapper.name == "mcp__fs__read_file"
# wrapper.description == "Read contents of a file"
```

---

### `wrapper_to_mcp_tool`

```python
def wrapper_to_mcp_tool(wrapper: ToolWrapper) -> Dict[str, Any]
```

Convert a corteX `ToolWrapper` back to MCP tool format. If the wrapper name uses the `mcp__server__tool` convention, the prefix is stripped so the returned dict contains the original tool name.

**Parameters**:

- `wrapper` (`ToolWrapper`): The corteX tool wrapper to convert.

**Returns**: `Dict[str, Any]` -- MCP tool dict with `name`, `description`, and `inputSchema` keys.

**Example**:

```python
from corteX.interop.mcp.tool_bridge import wrapper_to_mcp_tool

mcp_schema = wrapper_to_mcp_tool(wrapper)
# {"name": "read_file", "description": "...", "inputSchema": {...}}
```

---

### `mcp_result_to_string`

```python
def mcp_result_to_string(result: Any) -> str
```

Normalize an MCP `CallToolResult` (or similar) to a plain string. Handles several result shapes:

| Shape | Handling |
|-------|----------|
| `None` | Returns `""` |
| `str` | Returns as-is |
| `dict` with `"content"` list | Extracts `text` from content items, joins with newlines |
| `dict` with `"text"` key | Returns `str(result["text"])` |
| `dict` with `"result"` key | Returns `str(result["result"])` |
| Object with `.content` attribute | Extracts `.text` from content items |
| Anything else | Returns `str(result)` |

**Parameters**:

- `result` (`Any`): Raw MCP tool result in any supported shape.

**Returns**: `str` -- Normalized string result.

**Example**:

```python
from corteX.interop.mcp.tool_bridge import mcp_result_to_string

# Standard MCP format
result = {"content": [{"type": "text", "text": "Hello"}, {"type": "text", "text": "World"}]}
text = mcp_result_to_string(result)
# "Hello\nWorld"

# Simple string
text = mcp_result_to_string("plain text")
# "plain text"

# None
text = mcp_result_to_string(None)
# ""
```

---

## Constants

### `_MCP_PREFIX`

```python
_MCP_PREFIX: str = "mcp__"
```

The namespace prefix used for all MCP tool names.

---

## See Also

- [MCP Client](./mcp-client.md) -- Uses the tool bridge for discovery and execution
- [Types](./types.md) -- `MCPServerConfig` dataclass
- [MCP Transport](./mcp-transport.md) -- Low-level transport layer
