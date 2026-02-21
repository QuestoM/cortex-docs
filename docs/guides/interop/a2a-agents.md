# Delegate Tasks to A2A Agents

This guide walks you through connecting a corteX agent to external A2A (Agent-to-Agent) agents so it can delegate complex sub-tasks to specialized peers.

## Prerequisites

- corteX SDK installed: `pip install cortex-ai`
- An HTTP client library: `pip install aiohttp` or `pip install httpx`
- One or more A2A-compliant agents running and accessible over HTTP

## Step 1: Configure an A2A Agent

Create an `A2AAgentConfig` for each external agent you want to delegate to:

```python
from corteX.interop.types import A2AAgentConfig

researcher = A2AAgentConfig(
    name="researcher",
    url="https://research-agent.internal:8080",
    description="Deep research and market analysis agent",
    skills=["market_analysis", "literature_review"],
    auth_token="bearer-token-...",
    timeout=60.0,
)
```

| Parameter | Required | Description |
|---|---|---|
| `name` | Yes | Agent identifier (used for delegation calls) |
| `url` | Yes | Base URL of the A2A agent |
| `description` | No | Human-readable description of the agent's purpose |
| `skills` | No | List of skill IDs this agent provides |
| `auth_token` | No | Bearer token for authentication |
| `headers` | No | Additional HTTP headers |
| `timeout` | No | Request timeout in seconds (default: 60) |
| `max_retries` | No | Max retry attempts on failure (default: 2) |

## Step 2: Create an Agent with A2A Peers

Pass your agent configs to `create_agent()`:

```python
import cortex

engine = cortex.Engine(providers={"openai": {"api_key": "sk-..."}})

agent = engine.create_agent(
    name="orchestrator",
    system_prompt="You coordinate work across specialized agents.",
    a2a_agents=[researcher],
)
```

## Step 3: Discover Agents

When you start a session, call `connect_interop()` to discover all configured A2A agents by fetching their Agent Cards:

```python
session = agent.start_session(user_id="user_123")

status = await session.connect_interop()
print(status)
# {"mcp": {}, "a2a": {"researcher": True}}
```

During discovery, corteX fetches `{url}/.well-known/agent.json` for each agent and parses the Agent Card to learn about available skills and capabilities.

## Step 4: Send a Task

Delegate a task to a discovered agent:

```python
result = await session._a2a_client.send_task(
    agent_name="researcher",
    goal="Analyze the enterprise AI agent market in 2026",
    context="Focus on pricing models and enterprise adoption rates",
)
print(f"Task ID: {result.task_id}")
print(f"Status: {result.status}")  # "submitted" or "working"
```

## Step 5: Poll for Results

For long-running tasks, poll the status until completion:

```python
import asyncio

while True:
    status = await session._a2a_client.get_task_status(
        agent_name="researcher",
        task_id=result.task_id,
    )
    if status.status in ("completed", "failed", "canceled"):
        break
    await asyncio.sleep(2)  # Poll every 2 seconds

if status.status == "completed":
    # Extract text from artifacts
    from corteX.interop.a2a.task_bridge import A2ATaskBridge
    bridge = A2ATaskBridge()
    text = bridge.get_result_text(status)
    print(text)
elif status.status == "failed":
    print(f"Task failed: {status.error}")
```

## Step 6: Cancel a Task

If you need to cancel an in-progress task:

```python
cancelled = await session._a2a_client.cancel_task(
    agent_name="researcher",
    task_id=result.task_id,
)
print(f"Cancelled: {cancelled}")  # True or False
```

## Step 7: Disconnect

Clean up when done:

```python
await session.disconnect_interop()
```

## Working with Agent Cards

### Inspecting a Discovered Agent

After discovery, you can inspect an agent's card:

```python
card = session._a2a_client.get_agent("researcher")
print(f"Name: {card.name}")
print(f"Description: {card.description}")
print(f"Version: {card.version}")
for skill in card.skills:
    print(f"  Skill: {skill.name} -- {skill.description}")
    print(f"    Tags: {skill.tags}")
    print(f"    Examples: {skill.examples}")
```

### Listing All Discovered Agents

```python
agents = session._a2a_client.get_available_agents()
for agent_card in agents:
    print(f"{agent_card.name}: {agent_card.description}")
```

## Multiple A2A Agents

You can configure as many A2A agents as needed. Each is discovered independently:

```python
agent = engine.create_agent(
    name="orchestrator",
    system_prompt="You coordinate research, translation, and review tasks.",
    a2a_agents=[
        A2AAgentConfig(
            name="researcher",
            url="https://research.internal:8080",
            description="Market research and analysis",
        ),
        A2AAgentConfig(
            name="translator",
            url="https://translate.internal:8080",
            description="Multi-language translation",
        ),
        A2AAgentConfig(
            name="reviewer",
            url="https://review.internal:8080",
            description="Document and code review",
        ),
    ],
)
```

## Combining MCP and A2A

For maximum capability, use both protocols together. MCP provides tools, A2A provides agents:

```python
from corteX.interop.types import MCPServerConfig, A2AAgentConfig

agent = engine.create_agent(
    name="full-stack-assistant",
    system_prompt="You have file access and can delegate research tasks.",
    mcp_servers=[
        MCPServerConfig(
            name="filesystem",
            transport="stdio",
            command="npx",
            args=["-y", "@modelcontextprotocol/server-filesystem", "/data"],
        ),
    ],
    a2a_agents=[
        A2AAgentConfig(
            name="researcher",
            url="https://research.internal:8080",
            description="Deep research agent",
        ),
    ],
)

session = agent.start_session(user_id="user_123")
status = await session.connect_interop()
# status: {"mcp": {"filesystem": True}, "a2a": {"researcher": True}}
```

## Error Handling

### Discovery Failures

If an agent cannot be discovered (network error, invalid card), `discover_all()` logs a warning and marks that agent as not discovered:

```python
status = await session.connect_interop()
for agent_name, discovered in status["a2a"].items():
    if not discovered:
        conn = session._a2a_client._connections[agent_name]
        print(f"Agent '{agent_name}' failed: {conn.error}")
```

### Task Failures

Task operations raise `ValueError` for configuration issues and `PermissionError` for capability violations:

```python
try:
    result = await session._a2a_client.send_task(
        agent_name="unknown",
        goal="Do something",
    )
except ValueError as e:
    print(f"Agent not configured or not discovered: {e}")
except PermissionError as e:
    print(f"Capability check failed: {e}")
```

### Missing HTTP Library

If neither `aiohttp` nor `httpx` is installed:

```python
# This will raise ImportError with installation instructions
pip install aiohttp  # or: pip install httpx
```

## Publishing Your Agent as an A2A Server

To make your corteX agent available to other A2A agents, build and serve an Agent Card:

```python
from corteX.interop.a2a.agent_card import AgentCardBuilder, AgentSkill

card_json = AgentCardBuilder.build_card(
    name="Support Agent",
    description="Handles customer support tickets for Barvaz Security",
    url="https://support.barvaz.com",
    skills=[
        AgentSkill(
            id="ticket_resolution",
            name="Ticket Resolution",
            description="Resolve customer support tickets",
            tags=["support", "helpdesk"],
        ),
    ],
)

# Serve as JSON at /.well-known/agent.json
# (use your preferred web framework -- FastAPI, Flask, etc.)
```

## Security Considerations

- Use `auth_token` to authenticate with remote agents
- A2A delegation is subject to CapabilitySet checks (`a2a:{agent}` with `delegate` action)
- `to_dict()` on `A2AAgentConfig` excludes `auth_token` to prevent accidental serialization of secrets
- Always use HTTPS for production A2A connections
- Validate Agent Cards from untrusted sources before trusting their skill claims

## See Also

- [Protocol Interoperability](../../concepts/interop/index.md) -- Overview of MCP and A2A
- [A2A -- Agent-to-Agent Protocol](../../concepts/interop/a2a.md) -- Concept deep dive
- [Connect to MCP Servers](mcp-servers.md) -- The other interop protocol
