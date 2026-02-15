# Tool Decorator API Reference

## Module: `corteX.tools.decorator`

Decorator-based tool registration framework with per-instance isolation. Provides the `@tool` decorator for the global registry and `ToolRegistry` for per-tenant isolated tool sets.

## Functions

### `tool`

```python
def tool(
    name: Optional[str] = None,
    description: Optional[str] = None,
    parameters: Optional[Dict[str, Any]] = None,
) -> Callable
```

Decorator to register a function as a corteX tool in the default global registry. Supports both sync and async functions. Automatically generates a JSON Schema from type hints.

**Parameters**:

- `name` (`Optional[str]`): Custom tool name. Defaults to the function name
- `description` (`Optional[str]`): Tool description for the LLM. Defaults to the function docstring
- `parameters` (`Optional[Dict[str, Any]]`): Explicit JSON Schema for parameters. If not provided, auto-generated from type hints

**Returns**: A `ToolWrapper` instance (the decorated function is wrapped).

**Example**:

```python
from corteX.tools.decorator import tool

@tool(name="get_weather", description="Get current weather for a city")
def get_weather(city: str, units: str = "celsius") -> str:
    """Get weather data.
    :param city: City name
    :param units: Temperature units (celsius or fahrenheit)
    """
    return f"Weather in {city}: 22{units[0].upper()}"

@tool()
async def search_database(query: str, limit: int = 10) -> str:
    """Search the internal database."""
    results = await db.search(query, limit=limit)
    return str(results)
```

### `get_registered_tools`

```python
def get_registered_tools() -> Dict[str, ToolWrapper]
```

Get all tools from the default global registry.

### `get_tool`

```python
def get_tool(name: str) -> Optional[ToolWrapper]
```

Get a single tool by name from the default registry. Returns `None` if not found.

### `clear_tools`

```python
def clear_tools() -> None
```

Clear all tools from the default registry.

---

## Classes

### `ToolWrapper`

Wraps a user-defined function as a corteX tool with metadata, JSON Schema, and execution capability.

#### Constructor

```python
ToolWrapper(
    func: Callable,
    name: str,
    description: str,
    parameters: Optional[Dict[str, Any]] = None,
)
```

**Parameters**:

- `func` (Callable): The underlying function (sync or async)
- `name` (str): Tool name
- `description` (str): Description for the LLM
- `parameters` (`Optional[Dict[str, Any]]`): JSON Schema. Auto-generated from function signature if not provided

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `func` | `Callable` | The underlying function |
| `name` | `str` | Tool name |
| `description` | `str` | Human-readable description |
| `parameters` | `Dict[str, Any]` | JSON Schema for parameters |
| `is_async` | `bool` | Whether the function is async |

#### Methods

##### `to_definition`

```python
def to_definition(self) -> ToolDefinition
```

Convert to a universal `ToolDefinition` for LLM providers.

##### `execute`

```python
async def execute(self, **kwargs) -> str
```

Execute the tool with given arguments. Handles both sync and async functions. Returns the result as a string. On error, returns an error message string (does not raise).

---

### `ToolRegistry`

Per-instance tool registry for tenant isolation. Each registry maintains its own independent set of tools.

#### Constructor

```python
ToolRegistry(*, name: Optional[str] = None)
```

**Parameters**:

- `name` (`Optional[str]`): Registry name for identification. Defaults to `"default"`

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `name` | `str` | Registry name |

#### Methods

##### `register`

```python
def register(
    self, func: Callable, name: Optional[str] = None,
    description: Optional[str] = None,
    parameters: Optional[Dict[str, Any]] = None,
) -> ToolWrapper
```

Register a callable as a tool in this registry.

##### `register_wrapper`

```python
def register_wrapper(self, wrapper: ToolWrapper) -> None
```

Register a pre-built `ToolWrapper`.

##### `get`

```python
def get(self, name: str) -> Optional[ToolWrapper]
```

Get a tool by name.

##### `get_all`

```python
def get_all(self) -> Dict[str, ToolWrapper]
```

Get all registered tools.

##### `clear`

```python
def clear(self) -> None
```

Remove all tools from this registry.

##### `import_from`

```python
def import_from(self, source: ToolRegistry) -> None
```

Import all tools from another registry into this one.

##### `__len__`

Returns the number of registered tools.

##### `__contains__`

Check if a tool name exists in the registry: `"tool_name" in registry`.

**Example**:

```python
from corteX.tools.decorator import ToolRegistry

# Per-tenant isolated registry
tenant_tools = ToolRegistry(name="acme-corp")

# Register tools for this tenant
def lookup_order(order_id: str) -> str:
    """Look up an order by ID."""
    return f"Order {order_id}: shipped"

tenant_tools.register(lookup_order)

# Check registration
assert "lookup_order" in tenant_tools
assert len(tenant_tools) == 1
```

---

## Auto-Schema Generation

The decorator framework automatically converts Python type hints to JSON Schema:

| Python Type | JSON Schema |
|------------|-------------|
| `str` | `{"type": "string"}` |
| `int` | `{"type": "integer"}` |
| `float` | `{"type": "number"}` |
| `bool` | `{"type": "boolean"}` |
| `List[str]` | `{"type": "array", "items": {"type": "string"}}` |
| `Dict[str, int]` | `{"type": "object", "additionalProperties": {"type": "integer"}}` |
| `Optional[str]` | `{"type": "string"}` |
| `Enum` subclass | `{"type": "string", "enum": [...]}` |
| Pydantic `BaseModel` | Full JSON Schema from `model_json_schema()` |
| `Annotated[T, ...]` | Unwraps to schema for `T` |
| `Tuple[str, int]` | `{"type": "array", "prefixItems": [...]}` |
| `Set[str]` | `{"type": "array", "uniqueItems": true}` |

Parameters without default values are marked as `required` in the schema.

---

## See Also

- [Tool Executor API](./executor.md) - Executes registered tools safely
- [Custom Tools Guide](../../guides/tools/custom-tools.md) - How-to guide
