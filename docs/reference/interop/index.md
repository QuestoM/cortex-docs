# Interop API Reference

## Module: `corteX.interop`

The interop package provides standards-based integration with external tools and agents via two open protocols:

- **MCP (Model Context Protocol)** -- Connect to external tool servers using stdio or SSE transports
- **A2A (Agent-to-Agent)** -- Delegate tasks to external AI agents via JSON-RPC

All interop modules are opt-in. When no `InteropConfig` is provided, the agent operates with zero external dependencies.

---

## MCP Modules

| Module | Description |
|--------|-------------|
| [Types](./types.md) | `InteropConfig`, `MCPServerConfig`, `A2AAgentConfig` dataclasses |
| [MCP Client](./mcp-client.md) | `MCPClientManager` -- connect, discover tools, execute |
| [MCP Tool Bridge](./mcp-tool-bridge.md) | Convert between MCP tool schemas and corteX `ToolWrapper` |
| [MCP Transport](./mcp-transport.md) | `StdioTransport`, `SSETransport` -- low-level protocol layer |
| [MCP Resource Bridge](./mcp-resource-bridge.md) | `MCPResourceBridge` -- fetch MCP resources as LLM context |

## A2A Modules

| Module | Description |
|--------|-------------|
| [A2A Client](./a2a-client.md) | `A2AClientManager` -- discover agents, send/cancel tasks |
| [A2A Agent Card](./a2a-agent-card.md) | `AgentCardBuilder` -- build and parse Agent Card JSON |
| [A2A Task Bridge](./a2a-task-bridge.md) | `A2ATaskBridge` -- map A2A tasks to corteX sub-agent lifecycle |

---

## Quick Start

```python
from corteX.interop.types import InteropConfig, MCPServerConfig, A2AAgentConfig

config = InteropConfig(
    mcp_servers=[
        MCPServerConfig(
            name="filesystem",
            transport="stdio",
            command="npx",
            args=["-y", "@modelcontextprotocol/server-filesystem", "/tmp"],
        ),
    ],
    a2a_agents=[
        A2AAgentConfig(
            name="summarizer",
            url="https://summarizer.example.com",
            description="Summarizes documents",
        ),
    ],
)
```

Pass `InteropConfig` when creating an agent to enable interop:

```python
from corteX.sdk import Engine

engine = Engine(api_key="sk-...")
agent = engine.create_agent(
    system_prompt="You are a helpful assistant.",
    interop=config,
)
```

---

## Architecture

```
InteropConfig
  |
  +-- MCPServerConfig[] -----> MCPClientManager
  |                              |-- StdioTransport / SSETransport
  |                              |-- tool_bridge module (tool conversion)
  |                              +-- MCPResourceBridge (resource context)
  |
  +-- A2AAgentConfig[] ------> A2AClientManager
                                 |-- AgentCardBuilder (discovery)
                                 +-- A2ATaskBridge (task lifecycle)
```

---

## See Also

- [Interop Concepts Guide](../../concepts/interop/index.md) -- Conceptual overview of MCP and A2A
- [MCP Integration How-To](../../guides/interop/mcp-servers.md) -- Step-by-step setup guide
- [A2A Delegation How-To](../../guides/interop/a2a-agents.md) -- Agent delegation guide
