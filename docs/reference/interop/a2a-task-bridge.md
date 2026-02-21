# A2A Task Bridge API Reference

## Module: `corteX.interop.a2a.task_bridge`

Maps A2A JSON-RPC tasks to the corteX sub-agent lifecycle. Handles building JSON-RPC 2.0 requests, parsing responses, and converting between A2A and corteX status values.

---

## Classes

### `A2AArtifact`

**Type**: `@dataclass`

An artifact produced by an A2A task.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `parts` | `List[Dict[str, Any]]` | `[]` | Content parts (e.g. `[{"type": "text", "text": "..."}]`) |
| `name` | `str` | `""` | Artifact name |
| `description` | `str` | `""` | Artifact description |

---

### `A2AMessage`

**Type**: `@dataclass`

A message exchanged during an A2A task.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `role` | `str` | *(required)* | Message role: `"user"` or `"agent"` |
| `parts` | `List[Dict[str, Any]]` | `[]` | Content parts |

---

### `A2ATaskResult`

**Type**: `@dataclass`

Parsed result from an A2A task response.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `task_id` | `str` | *(required)* | Unique task identifier |
| `status` | `str` | *(required)* | Task status: `"submitted"`, `"working"`, `"completed"`, `"failed"`, `"canceled"` |
| `artifacts` | `List[A2AArtifact]` | `[]` | Artifacts produced by the task |
| `messages` | `List[A2AMessage]` | `[]` | Messages exchanged during the task |
| `error` | `Optional[str]` | `None` | Error message if the task failed |

---

### `A2ATaskBridge`

Maps A2A JSON-RPC tasks to corteX sub-agent lifecycle.

#### Methods

##### `build_send_request`

```python
def build_send_request(
    self,
    goal: str,
    context: str = "",
    task_id: Optional[str] = None,
) -> Dict[str, Any]
```

Build a JSON-RPC 2.0 `tasks/send` request.

**Parameters**:

- `goal` (`str`): The task goal / user message.
- `context` (`str`): Optional additional context (added as a second text part).
- `task_id` (`Optional[str]`): Optional task ID. Generated as UUID if omitted.

**Returns**: `Dict[str, Any]` -- JSON-RPC 2.0 request dict.

**Example**:

```python
from corteX.interop.a2a.task_bridge import A2ATaskBridge

bridge = A2ATaskBridge()

request = bridge.build_send_request(
    goal="Summarize the Q4 report",
    context="Focus on revenue metrics",
)

# {
#     "jsonrpc": "2.0",
#     "id": "abc-123-...",
#     "method": "tasks/send",
#     "params": {
#         "id": "abc-123-...",
#         "message": {
#             "role": "user",
#             "parts": [
#                 {"type": "text", "text": "Summarize the Q4 report"},
#                 {"type": "text", "text": "Focus on revenue metrics"},
#             ],
#         },
#     },
# }
```

##### `build_get_request`

```python
def build_get_request(self, task_id: str) -> Dict[str, Any]
```

Build a JSON-RPC 2.0 `tasks/get` request.

**Parameters**:

- `task_id` (`str`): The task ID to query.

**Returns**: `Dict[str, Any]` -- JSON-RPC 2.0 request dict.

##### `build_cancel_request`

```python
def build_cancel_request(self, task_id: str) -> Dict[str, Any]
```

Build a JSON-RPC 2.0 `tasks/cancel` request.

**Parameters**:

- `task_id` (`str`): The task ID to cancel.

**Returns**: `Dict[str, Any]` -- JSON-RPC 2.0 request dict.

##### `parse_response`

```python
def parse_response(self, response: Dict[str, Any]) -> A2ATaskResult
```

Parse a JSON-RPC 2.0 response into `A2ATaskResult`.

Handles JSON-RPC errors (returns status `"failed"` with error message), and extracts artifacts and messages from the result payload.

**Parameters**:

- `response` (`Dict[str, Any]`): Raw JSON-RPC 2.0 response dict.

**Returns**: `A2ATaskResult` -- Parsed result.

**Example**:

```python
# Success response
response = {
    "jsonrpc": "2.0",
    "id": "task-123",
    "result": {
        "id": "task-123",
        "status": {"state": "completed"},
        "artifacts": [
            {
                "parts": [{"type": "text", "text": "Summary: Revenue grew 15%..."}],
                "name": "summary",
            },
        ],
    },
}

result = bridge.parse_response(response)
assert result.status == "completed"
assert len(result.artifacts) == 1

# Error response
error_response = {
    "jsonrpc": "2.0",
    "id": "task-456",
    "error": {"code": -32600, "message": "Invalid request"},
}

result = bridge.parse_response(error_response)
assert result.status == "failed"
assert result.error == "Invalid request"
```

##### `a2a_status_to_cortex`

```python
def a2a_status_to_cortex(self, status: str) -> str
```

Map A2A task status to corteX sub-agent status.

**Status Mapping**:

| A2A Status | corteX Status |
|------------|---------------|
| `submitted` | `pending` |
| `working` | `running` |
| `completed` | `completed` |
| `failed` | `failed` |
| `canceled` | `cancelled` |

**Parameters**:

- `status` (`str`): A2A status string.

**Returns**: `str` -- corteX sub-agent status string. Unknown statuses are returned as-is.

##### `cortex_status_to_a2a`

```python
def cortex_status_to_a2a(self, status: str) -> str
```

Map corteX sub-agent status to A2A task status.

**Status Mapping**:

| corteX Status | A2A Status |
|---------------|------------|
| `pending` | `submitted` |
| `running` | `working` |
| `completed` | `completed` |
| `failed` | `failed` |
| `cancelled` | `canceled` |

**Parameters**:

- `status` (`str`): corteX sub-agent status string.

**Returns**: `str` -- A2A status string. Unknown statuses are returned as-is.

##### `get_result_text`

```python
def get_result_text(self, result: A2ATaskResult) -> str
```

Extract all text content from a task result. Concatenates text from artifacts and messages into a single string, joined by newlines.

**Parameters**:

- `result` (`A2ATaskResult`): Parsed task result.

**Returns**: `str` -- Combined text content from all text parts.

**Example**:

```python
result = A2ATaskResult(
    task_id="task-123",
    status="completed",
    artifacts=[
        A2AArtifact(parts=[{"type": "text", "text": "Summary paragraph 1"}]),
        A2AArtifact(parts=[{"type": "text", "text": "Summary paragraph 2"}]),
    ],
)

text = bridge.get_result_text(result)
# "Summary paragraph 1\nSummary paragraph 2"
```

---

## Full Lifecycle Example

```python
from corteX.interop.a2a.task_bridge import A2ATaskBridge

bridge = A2ATaskBridge()

# 1. Build and send a task
request = bridge.build_send_request("Translate to French: Hello, world!")
# POST request to agent URL...
response = await post_to_agent(request)

# 2. Parse the response
result = bridge.parse_response(response)
print(f"Status: {result.status}")  # "working"

# 3. Poll for completion
get_req = bridge.build_get_request(result.task_id)
status_response = await post_to_agent(get_req)
final = bridge.parse_response(status_response)

# 4. Extract text
if final.status == "completed":
    text = bridge.get_result_text(final)
    print(text)  # "Bonjour, le monde!"

# 5. Or cancel if needed
cancel_req = bridge.build_cancel_request(result.task_id)
await post_to_agent(cancel_req)
```

---

## See Also

- [A2A Client](./a2a-client.md) -- High-level client that uses the task bridge
- [A2A Agent Card](./a2a-agent-card.md) -- Agent discovery
- [Types](./types.md) -- `A2AAgentConfig` dataclass
