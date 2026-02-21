# A2A Client Manager API Reference

## Module: `corteX.interop.a2a.client`

Discovers and interacts with external A2A agents. Manages agent discovery (via Agent Card), task delegation, and status tracking across multiple A2A-compliant agents.

---

## Classes

### `A2AAgentConnection`

**Type**: `@dataclass`

Tracks connection to an external A2A agent.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `config` | `A2AAgentConfig` | *(required)* | Agent configuration |
| `card` | `Optional[AgentCardInfo]` | `None` | Parsed Agent Card (populated after discovery) |
| `discovered` | `bool` | `False` | Whether discovery succeeded |
| `error` | `Optional[str]` | `None` | Error message if discovery failed |

---

### `A2AClientManager`

Discovers and interacts with external A2A agents. Manages agent discovery, task sending, status polling, and task cancellation.

#### Constructor

```python
A2AClientManager(
    configs: Optional[List[A2AAgentConfig]] = None,
    capability_set: Optional[CapabilitySet] = None,
)
```

**Parameters**:

- `configs` (`Optional[List[A2AAgentConfig]]`): List of agent configurations.
- `capability_set` (`Optional[CapabilitySet]`): Optional capability set for permission checks. When provided, `send_task` verifies `a2a:{agent_name}` has the `delegate` permission.

#### Methods

##### `discover_all`

```python
async def discover_all(self) -> Dict[str, bool]
```

Discover all configured agents in parallel. Fetches each agent's Agent Card from `{url}/.well-known/agent.json`.

**Returns**: `Dict[str, bool]` -- Mapping of `{agent_name: success}`.

**Example**:

```python
from corteX.interop.types import A2AAgentConfig
from corteX.interop.a2a.client import A2AClientManager

configs = [
    A2AAgentConfig(name="summarizer", url="https://summarizer.example.com"),
    A2AAgentConfig(name="translator", url="https://translator.example.com"),
]

manager = A2AClientManager(configs)
results = await manager.discover_all()
# {"summarizer": True, "translator": True}
```

##### `discover_agent`

```python
async def discover_agent(self, config: A2AAgentConfig) -> bool
```

Discover a single agent by fetching its Agent Card.

**Parameters**:

- `config` (`A2AAgentConfig`): Agent configuration with URL.

**Returns**: `bool` -- `True` if discovery succeeded.

##### `get_available_agents`

```python
def get_available_agents(self) -> List[AgentCardInfo]
```

Return `AgentCardInfo` list for all discovered agents.

**Returns**: `List[AgentCardInfo]` -- Cards for all successfully discovered agents.

##### `get_agent`

```python
def get_agent(self, name: str) -> Optional[AgentCardInfo]
```

Get a specific agent's card by name.

**Parameters**:

- `name` (`str`): Agent name.

**Returns**: `Optional[AgentCardInfo]` -- Agent card if discovered, `None` otherwise.

##### `send_task`

```python
async def send_task(
    self,
    agent_name: str,
    goal: str,
    context: str = "",
) -> A2ATaskResult
```

Send a task to an A2A agent. Builds a JSON-RPC 2.0 `tasks/send` request, posts it to the agent's URL, and parses the response.

**Parameters**:

- `agent_name` (`str`): Name of the target agent.
- `goal` (`str`): Task goal / instruction.
- `context` (`str`): Optional additional context.

**Returns**: `A2ATaskResult` -- Parsed result from the remote agent.

**Raises**:

- `ValueError` -- If agent is not configured or not yet discovered.
- `PermissionError` -- If capability check fails (no `delegate` on `a2a:{agent_name}`).

**Example**:

```python
result = await manager.send_task(
    agent_name="summarizer",
    goal="Summarize the Q4 financial report",
    context="Focus on revenue growth and key metrics",
)

print(result.task_id)   # UUID
print(result.status)    # "completed", "working", etc.
```

##### `get_task_status`

```python
async def get_task_status(
    self,
    agent_name: str,
    task_id: str,
) -> A2ATaskResult
```

Get the current status of a task from an A2A agent.

**Parameters**:

- `agent_name` (`str`): Name of the target agent.
- `task_id` (`str`): Task ID to query.

**Returns**: `A2ATaskResult` -- Current task status and results.

**Raises**: `ValueError` -- If agent is not configured or not discovered.

**Example**:

```python
status = await manager.get_task_status("summarizer", result.task_id)
print(status.status)  # "completed"
```

##### `cancel_task`

```python
async def cancel_task(
    self,
    agent_name: str,
    task_id: str,
) -> bool
```

Cancel a task on an A2A agent.

**Parameters**:

- `agent_name` (`str`): Name of the target agent.
- `task_id` (`str`): Task ID to cancel.

**Returns**: `bool` -- `True` if the cancel request succeeded (status became `canceled`/`cancelled`).

**Raises**: `ValueError` -- If agent is not configured or not discovered.

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `discovered_agents` | `List[str]` | Names of all successfully discovered agents |

---

## HTTP Provider

The A2AClientManager automatically selects an available HTTP library:

1. **aiohttp** (preferred) -- `pip install aiohttp`
2. **httpx** (fallback) -- `pip install httpx`

If neither is installed, `ImportError` is raised with installation instructions.

---

## Capability Enforcement

When a `CapabilitySet` is provided, `send_task` checks:

```python
resource = f"a2a:{agent_name}"
capability_set.has(resource, "delegate")
```

If the check fails, `PermissionError` is raised before any HTTP request.

```python
from corteX.security.capabilities import CapabilitySet

caps = CapabilitySet()
caps.grant("a2a:summarizer", "delegate")
# Only summarizer delegation is allowed

manager = A2AClientManager(configs, capability_set=caps)
```

---

## See Also

- [Types](./types.md) -- `A2AAgentConfig` dataclass
- [A2A Agent Card](./a2a-agent-card.md) -- Agent Card builder and parser
- [A2A Task Bridge](./a2a-task-bridge.md) -- Task lifecycle mapping
