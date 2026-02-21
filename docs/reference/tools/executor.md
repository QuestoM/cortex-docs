# Tool Executor API Reference

## Module: `corteX.tools.executor`

Safe tool execution engine with timeout enforcement, error wrapping, and latency tracking. Integration hooks for the Weight Engine and Prediction system are defined in the class but not yet wired (planned for a future release).

**Security principle**: Only tools explicitly registered by the developer can be executed. The agent never discovers or calls tools on its own initiative. This is enforced at the executor level.

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `DEFAULT_TIMEOUT_SECONDS` | `30` | Default execution timeout per tool call |

---

## Classes

### `ToolExecutionResult`

Result of a tool execution.

#### Constructor

```python
ToolExecutionResult(
    tool_name: str,
    success: bool,
    output: str,
    latency_ms: float,
    error: Optional[str] = None,
)
```

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `tool_name` | `str` | Name of the executed tool |
| `success` | `bool` | Whether execution succeeded |
| `output` | `str` | Tool output (empty string on failure) |
| `latency_ms` | `float` | Execution time in milliseconds |
| `error` | `Optional[str]` | Error message (if failed) |

#### Methods

##### `to_dict`

```python
def to_dict(self) -> Dict[str, Any]
```

Serialize to dict. Output is truncated to 2000 characters for context window management.

---

### `ToolExecutor`

Executes tools safely with timeout enforcement, error wrapping, and latency tracking.

#### Constructor

```python
ToolExecutor(
    tools: Optional[List[ToolWrapper]] = None,
    timeout: int = DEFAULT_TIMEOUT_SECONDS,
)
```

**Parameters**:

- `tools` (`Optional[List[ToolWrapper]]`): Initial tools to register
- `timeout` (int, default=30): Maximum execution time in seconds per tool call

#### Methods

##### `register`

```python
def register(self, tool: ToolWrapper) -> None
```

Register a tool for execution.

##### `get_definitions`

```python
def get_definitions(self) -> List[ToolDefinition]
```

Get all tool definitions for passing to LLM providers. Returns a list of `ToolDefinition` objects.

##### `execute`

```python
async def execute(
    self,
    tool_name: str,
    arguments: Dict[str, Any],
) -> ToolExecutionResult
```

Execute a tool by name with given arguments. Enforces timeout and captures errors.

**Security**: Only tools explicitly registered by the developer can be executed. Attempts to call unregistered tools are blocked with a security warning logged.

**Parameters**:

- `tool_name` (str): Name of the tool to execute
- `arguments` (`Dict[str, Any]`): Keyword arguments for the tool function

**Returns**: `ToolExecutionResult` with success status, output, latency, and optional error.

**Example**:

```python
from corteX.tools.decorator import ToolWrapper
from corteX.tools.executor import ToolExecutor

def add(a: int, b: int) -> str:
    return str(a + b)

wrapper = ToolWrapper(func=add, name="add", description="Add two numbers")
executor = ToolExecutor(tools=[wrapper], timeout=10)

result = await executor.execute("add", {"a": 5, "b": 3})
print(result.success)     # True
print(result.output)      # "8"
print(result.latency_ms)  # 0.1

# Blocked: unregistered tool
result = await executor.execute("rm_rf", {"path": "/"})
print(result.success)  # False
print(result.error)    # "Tool not registered: rm_rf..."
```

##### `execute_tool_call`

```python
async def execute_tool_call(self, tool_call: Dict[str, Any]) -> ToolExecutionResult
```

Execute from an LLM tool call response. Parses the function name and JSON arguments from the standard tool call format:

```python
tool_call = {
    "function": {
        "name": "get_weather",
        "arguments": '{"city": "London"}',
    }
}
result = await executor.execute_tool_call(tool_call)
```

Handles JSON parsing errors gracefully, returning a failed `ToolExecutionResult`.

---

## Error Handling

The executor handles three categories of errors:

| Error Type | Behavior | Result |
|-----------|----------|--------|
| **Unregistered tool** | Blocked immediately, security warning logged | `success=False`, error explains restriction |
| **Timeout** | `asyncio.wait_for` cancels after `timeout` seconds | `success=False`, error shows timeout |
| **Exception** | Caught and wrapped | `success=False`, error contains exception message |

No exceptions propagate from `execute()` or `execute_tool_call()` -- all errors are captured in the `ToolExecutionResult`.

---

## See Also

- [Tool Decorator API](./decorator.md) - Tool registration framework
- [Custom Tools Guide](../../guides/tools/custom-tools.md)
- [Tool Reputation Guide](../../guides/tools/tool-reputation.md)
