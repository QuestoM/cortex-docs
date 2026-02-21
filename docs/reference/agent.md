# Agent API Reference

## Module: `corteX.sdk.Agent`

The `Agent` is a configured agent template. It is **stateless** -- it defines the agent's personality, tools, and behavior settings but holds no conversation state. To begin a conversation, call `start_session()` which returns a stateful `Session`.

Think of `Agent` as a blueprint: you configure it once, then create as many concurrent sessions as you need for different users.

## Class

### `Agent`

A configured agent template. Stateless -- creates sessions for live conversations.

#### Constructor

```python
Agent(
    engine: Engine,
    name: str,
    system_prompt: str = "",
    tools: Optional[List[ToolWrapper]] = None,
    goal_tracking: bool = True,
    loop_prevention: bool = True,
    weight_config: Optional[WeightConfig] = None,
    enterprise_config: Optional[EnterpriseConfig] = None,
    context_config: Optional[ContextManagementConfig] = None,
    enable_session_recording: bool = False,
    mcp_servers: Optional[List] = None,
    a2a_agents: Optional[List] = None,
)
```

!!! note "Prefer `engine.create_agent()`"
    You typically do not instantiate `Agent` directly. Use `engine.create_agent()` instead, which passes the engine reference automatically.

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `engine` | `Engine` | (required) | The parent engine that provides LLM routing and plugin registry. |
| `name` | `str` | (required) | Agent name for identification and logging. |
| `system_prompt` | `str` | `""` | System prompt defining agent persona and instructions. |
| `tools` | `Optional[List[ToolWrapper]]` | `None` | Tools available to the agent during sessions. |
| `goal_tracking` | `bool` | `True` | Enable goal progress tracking with drift detection. |
| `loop_prevention` | `bool` | `True` | Enable state-hash-based loop detection. |
| `weight_config` | `Optional[WeightConfig]` | `None` | Behavioral weight initial values and learning rates. |
| `enterprise_config` | `Optional[EnterpriseConfig]` | `None` | Enterprise settings (safety, audit, compliance). |
| `context_config` | `Optional[ContextManagementConfig]` | `None` | Cortical Context Engine configuration (token budgets, profiles). |
| `enable_session_recording` | `bool` | `False` | Enable digital twin session recording for replay and what-if. |
| `mcp_servers` | `Optional[List]` | `None` | List of MCP (Model Context Protocol) server configurations for external tool integration. Each entry is an `MCPServerConfig` specifying the server endpoint and capabilities. |
| `a2a_agents` | `Optional[List]` | `None` | List of A2A (Agent-to-Agent) agent configurations for inter-agent communication. Each entry is an `A2AAgentConfig` specifying the remote agent endpoint and protocol. |

---

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `engine` | `Engine` | Reference to the parent engine. |
| `name` | `str` | Agent identifier. |
| `system_prompt` | `str` | The system prompt injected at session start. |
| `tools` | `List[ToolWrapper]` | List of registered tools (empty list if none). |
| `goal_tracking` | `bool` | Whether goal tracking is enabled. |
| `loop_prevention` | `bool` | Whether loop detection is enabled. |
| `weight_config` | `Optional[WeightConfig]` | Behavioral weight configuration. |
| `enterprise_config` | `Optional[EnterpriseConfig]` | Enterprise configuration. |
| `context_config` | `Optional[ContextManagementConfig]` | Context management configuration. |
| `enable_session_recording` | `bool` | Whether session recording is enabled. |
| `mcp_servers` | `List` | MCP server configurations (empty list if none). |
| `a2a_agents` | `List` | A2A agent configurations (empty list if none). |

---

#### Methods

##### `start_session`

```python
def start_session(
    self,
    user_id: str = "default",
    session_id: Optional[str] = None,
) -> Session
```

Create a new conversation session for a specific user.

Each session is fully independent -- it initializes its own brain components (weights, feedback, prediction, plasticity, memory, etc.) and maintains its own conversation history. Multiple sessions can run concurrently for the same or different users.

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `user_id` | `str` | `"default"` | Identifier for the user. Used for weight personalization, memory isolation, and audit logging. |
| `session_id` | `Optional[str]` | `None` | Custom session ID. If not provided, a unique ID is auto-generated (`sess_<hex>`). |

**Returns**: `Session` - A new stateful conversation session with all brain components initialized.

**Example**:

```python
# Create a session for a specific user
session = agent.start_session(user_id="user_123")

# Create a session with a custom ID (useful for resumption tracking)
session = agent.start_session(
    user_id="user_123",
    session_id="support_ticket_456",
)
```

---

## Usage Patterns

### Basic Agent with Tools

```python
import cortex
from corteX.tools.decorator import tool

@tool(name="check_inventory", description="Check product inventory")
def check_inventory(product_id: str) -> str:
    # Your inventory logic here
    return f"Product {product_id}: 42 units in stock"

@tool(name="create_ticket", description="Create a support ticket")
def create_ticket(subject: str, description: str) -> str:
    return f"Ticket created: {subject}"

engine = cortex.Engine(providers={"openai": {"api_key": "sk-..."}})

agent = engine.create_agent(
    name="support",
    system_prompt=(
        "You are a support agent for an e-commerce platform. "
        "Help customers with orders, inventory, and tickets."
    ),
    tools=[check_inventory, create_ticket],
)
```

### Agent with Custom Weights

```python
from cortex import WeightConfig

# A concise, autonomous agent
coding_agent = engine.create_agent(
    name="code_reviewer",
    system_prompt="Review code for bugs, performance, and style.",
    weight_config=WeightConfig(
        verbosity=-0.3,       # Concise responses
        formality=0.2,        # Slightly formal
        autonomy=0.8,         # High autonomy
        initiative=0.6,       # Proactive suggestions
        speed_vs_quality=0.3, # Favor quality
    ),
)
```

### Agent with Context Management

```python
from cortex import ContextManagementConfig

# Agent optimized for long coding sessions (10,000+ steps)
long_session_agent = engine.create_agent(
    name="dev_assistant",
    system_prompt="You are a senior developer assistant.",
    context_config=ContextManagementConfig(
        enabled=True,
        token_budget_ratio=0.85,
        hot_ratio=0.45,
        warm_ratio=0.30,
        cold_ratio=0.25,
        summarize_every_n_steps=15,
        profile="coding",
    ),
)
```

### Enterprise Agent with Safety Controls

```python
from cortex import EnterpriseConfig

# Strict enterprise agent with audit trail
enterprise_agent = engine.create_agent(
    name="hr_assistant",
    system_prompt="You help employees with HR policies.",
    enterprise_config=EnterpriseConfig(
        safety_level="strict",
        audit_log=True,
        blocked_topics=["salary_negotiation", "legal_advice"],
        max_tokens_per_session=30000,
        compliance=["SOC2", "GDPR"],
    ),
    enable_session_recording=True,
)
```

### Multiple Concurrent Sessions

```python
import asyncio

async def handle_user(agent, user_id, question):
    session = agent.start_session(user_id=user_id)
    response = await session.run(question)
    await session.close()
    return response.content

async def main():
    # Handle multiple users concurrently
    results = await asyncio.gather(
        handle_user(agent, "alice", "Track order #100"),
        handle_user(agent, "bob", "Return policy?"),
        handle_user(agent, "carol", "Product availability?"),
    )
    for result in results:
        print(result)
```

---

## Design Notes

- **Stateless Design**: The Agent holds configuration only. All conversation state, brain state, and learned weights live in the Session. This means you can safely share one Agent across many sessions.

- **Tool Binding**: Tools are bound at the Agent level, not the Session level. All sessions spawned from an agent share the same tool definitions. This is intentional -- tools represent capabilities, not state.

- **Config Inheritance**: If `enterprise_config` is not set on the Agent, it inherits the Engine's enterprise config. Agent-level config always takes priority.

---

## See Also

- [Engine API Reference](./engine.md) - Engine creation and provider setup
- [Session API Reference](./session.md) - Session execution methods
- [Tool Decorator Reference](./tools/decorator.md) - Creating tools
- [Weight Tuning Guide](../guides/config/weight-tuning.md) - Tuning behavioral weights
- [Your First Agent Tutorial](../getting-started/first-agent.md) - Step-by-step guide
