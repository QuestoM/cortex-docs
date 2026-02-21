# A2A -- Agent-to-Agent Protocol

The Agent-to-Agent Protocol (A2A) is an open standard for inter-agent communication. corteX's A2A integration lets your agent delegate complex sub-tasks to external agents that have their own reasoning capabilities, tools, and domain expertise.

## How A2A Works

Unlike MCP (which provides *tools*), A2A provides *agents*. You send a goal to a remote agent, it works on that goal autonomously, and returns results. The remote agent may use its own tools, LLMs, and memory to accomplish the task.

```
corteX Agent (your agent)
    |
    | send_task("Analyze Q4 revenue trends")
    v
A2AClientManager
    |
    +-- A2AAgentConnection ("analyst")
    |       card: AgentCardInfo (skills, capabilities)
    |       |
    |       +-- POST /tasks/send  (JSON-RPC 2.0)
    |       +-- POST /tasks/get   (poll status)
    |       +-- POST /tasks/cancel
    |
    +-- A2AAgentConnection ("translator")
            card: AgentCardInfo (skills, capabilities)
```

## Agent Card Discovery

Every A2A agent publishes an **Agent Card** at `/.well-known/agent.json`. This card describes the agent's identity, capabilities, and skills. corteX fetches this card during discovery to understand what the remote agent can do.

```json
{
    "name": "Research Agent",
    "description": "Deep research and analysis agent",
    "url": "https://research-agent.internal:8080",
    "version": "1.0",
    "skills": [
        {
            "id": "market_analysis",
            "name": "Market Analysis",
            "description": "Analyze market trends and competitive landscape",
            "tags": ["research", "market"],
            "examples": ["Analyze the CRM market in 2026"]
        },
        {
            "id": "literature_review",
            "name": "Literature Review",
            "description": "Review academic and industry publications",
            "tags": ["research", "academic"],
            "examples": ["Review recent papers on transformer architectures"]
        }
    ],
    "capabilities": {
        "input_modes": ["text"],
        "output_modes": ["text"]
    }
}
```

corteX parses this into an `AgentCardInfo` dataclass with typed `AgentSkill` entries.

## Task Lifecycle

A2A tasks follow a well-defined lifecycle with status transitions:

```
    send_task()
        |
        v
  +-----------+
  | submitted |
  +-----+-----+
        |
        v
  +-----------+
  |  working  | <-- agent is actively processing
  +-----+-----+
        |
   +----+----+
   |         |
   v         v
+------+ +--------+
|failed| |completed|
+------+ +--------+

At any point:
  cancel_task() --> canceled
```

### Status Mapping

corteX maps A2A statuses to its internal sub-agent lifecycle:

| A2A Status | corteX Status | Meaning |
|---|---|---|
| `submitted` | `pending` | Task received, queued |
| `working` | `running` | Agent actively processing |
| `completed` | `completed` | Task finished successfully |
| `failed` | `failed` | Task encountered an error |
| `canceled` | `cancelled` | Task was cancelled |

## Task Operations

### send_task()

Sends a goal to a remote agent. Returns an `A2ATaskResult` with the initial status (usually `submitted` or `working`).

```python
result = await a2a_client.send_task(
    agent_name="researcher",
    goal="Analyze the AI agent SDK market in 2026",
    context="Focus on enterprise adoption and pricing models",
)
print(f"Task {result.task_id}: {result.status}")
```

The request is a JSON-RPC 2.0 `tasks/send` call:

```json
{
    "jsonrpc": "2.0",
    "id": "uuid-...",
    "method": "tasks/send",
    "params": {
        "id": "uuid-...",
        "message": {
            "role": "user",
            "parts": [
                {"type": "text", "text": "Analyze the AI agent SDK market..."},
                {"type": "text", "text": "Focus on enterprise adoption..."}
            ]
        }
    }
}
```

### get_task_status()

Polls the remote agent for the current task status:

```python
status = await a2a_client.get_task_status(
    agent_name="researcher",
    task_id=result.task_id,
)
if status.status == "completed":
    for artifact in status.artifacts:
        print(artifact.name, artifact.parts)
```

### cancel_task()

Requests cancellation of an in-progress task:

```python
cancelled = await a2a_client.cancel_task(
    agent_name="researcher",
    task_id=result.task_id,
)
```

## Artifacts and Messages

A2A responses can contain **artifacts** (produced outputs) and **messages** (conversation history):

```python
from corteX.interop.a2a.task_bridge import A2ATaskBridge

bridge = A2ATaskBridge()
result = bridge.parse_response(response)

# Extract all text content
text = bridge.get_result_text(result)

# Access individual artifacts
for artifact in result.artifacts:
    print(f"Artifact: {artifact.name}")
    for part in artifact.parts:
        if part["type"] == "text":
            print(part["text"])
```

## Security via CapabilitySet

A2A delegation is gated by corteX's capability system. The resource pattern is `a2a:{agent_name}` with the `delegate` action:

```python
from corteX.security.capabilities import CapabilitySet

caps = CapabilitySet()
caps.grant("a2a:researcher", "delegate")
# a2a:translator is NOT granted -- delegation blocked
```

If the agent tries to delegate to an agent not in the capability set, a `PermissionError` is raised.

## Building Your Own Agent Card

If you want to expose a corteX agent *as* an A2A server (so other agents can delegate to it), use `AgentCardBuilder`:

```python
from corteX.interop.a2a.agent_card import AgentCardBuilder, AgentSkill

card = AgentCardBuilder.build_card(
    name="Support Agent",
    description="Customer support agent for Barvaz Security",
    url="https://support-agent.barvaz.com",
    version="1.0",
    skills=[
        AgentSkill(
            id="ticket_resolution",
            name="Ticket Resolution",
            description="Resolve customer support tickets",
            tags=["support", "helpdesk"],
            examples=["Resolve ticket #1234"],
        ),
    ],
)
# Serve card at /.well-known/agent.json
```

You can also pass `tools` (a list of `ToolWrapper` instances) to automatically convert the agent's tools into skill entries on the card.

## HTTP Client

The `A2AClientManager` uses either `aiohttp` or `httpx` for HTTP communication. It auto-detects which library is available:

```
pip install aiohttp   # Preferred
# OR
pip install httpx     # Alternative
```

If neither is installed, A2A operations raise a clear `ImportError` with installation instructions.

## Key Classes

| Class | Module | Purpose |
|---|---|---|
| `A2AAgentConfig` | `corteX.interop.types` | Agent connection configuration |
| `A2AClientManager` | `corteX.interop.a2a.client` | Discovery + task management |
| `A2AAgentConnection` | `corteX.interop.a2a.client` | Per-agent state tracking |
| `AgentCardInfo` | `corteX.interop.a2a.agent_card` | Parsed Agent Card data |
| `AgentSkill` | `corteX.interop.a2a.agent_card` | Skill advertised by an agent |
| `AgentCardBuilder` | `corteX.interop.a2a.agent_card` | Build and parse Agent Cards |
| `A2ATaskBridge` | `corteX.interop.a2a.task_bridge` | JSON-RPC request/response mapping |
| `A2ATaskResult` | `corteX.interop.a2a.task_bridge` | Parsed task result |
| `SessionInteropMixin` | `corteX.session.interop` | Wires A2A into session lifecycle |

## See Also

- [Protocol Interoperability](index.md) -- Overview and comparison with MCP
- [Delegate Tasks to A2A Agents](../../guides/interop/a2a-agents.md) -- Step-by-step how-to guide
- [MCP -- Model Context Protocol](mcp.md) -- Tool server protocol
