# Protocol Interoperability

corteX supports two industry-standard protocols for agent interoperability:

| Protocol | Purpose | Standard Body |
|----------|---------|--------------|
| **MCP** (Model Context Protocol) | Connect agents to external tools and data sources | Linux Foundation (Anthropic) |
| **A2A** (Agent-to-Agent) | Enable cross-agent task delegation | Linux Foundation (Google) |

Both are optional -- install with `pip install cortex-ai[mcp]` or `pip install cortex-ai[mcp,a2a]`.

## When to Use Each

**Use MCP when** your agent needs to access external tools: file systems, databases, APIs, or any MCP-compliant server. MCP tools appear as native corteX tools -- zero changes to your agent logic.

**Use A2A when** your agent needs to delegate tasks to other AI agents: a research agent, a code review agent, or any A2A-compliant endpoint. Tasks flow through the existing SubAgentManager with budget and concurrency tracking.

## Architecture

```
Your Application
    |
    v
corteX Engine
    |
    +---> SessionInteropMixin
    |         |
    |         +---> MCPClientManager ---> MCP Servers (tools, resources)
    |         |
    |         +---> A2AClientManager ---> A2A Agents (task delegation)
    |
    +---> ToolExecutor (MCP tools integrated here)
    +---> SubAgentManager (A2A tasks integrated here)
```

## Security

All interop connections are governed by the existing `CapabilitySet` security model:

- MCP tools require explicit `mcp:connect` capability
- A2A delegation requires `a2a:delegate` capability
- Per-tenant isolation: each session has its own interop connections
- No external calls unless the tenant explicitly enables them

## Quick Start

```python
from corteX.interop.types import MCPServerConfig, A2AAgentConfig

engine = cortex.Engine(providers={"openai": {"api_key": "sk-..."}})

agent = engine.create_agent(
    name="assistant",
    mcp_servers=[
        MCPServerConfig(
            name="filesystem",
            transport="stdio",
            command="npx",
            args=["@modelcontextprotocol/server-filesystem", "/data"],
        ),
    ],
    a2a_agents=[
        A2AAgentConfig(
            name="researcher",
            url="https://research-agent.example.com",
        ),
    ],
)

session = agent.start_session(user_id="u1")
await session.connect_interop()
response = await session.run("List files in /data")
```

## See Also

- [MCP Deep Dive](mcp.md) -- How MCP integration works
- [A2A Deep Dive](a2a.md) -- How A2A delegation works
- [Connect to MCP Servers](../../guides/interop/mcp-servers.md) -- Step-by-step guide
- [Delegate to A2A Agents](../../guides/interop/a2a-agents.md) -- Step-by-step guide
