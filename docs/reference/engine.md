# Engine API Reference

## Module: `corteX.sdk.Engine`

The `Engine` is the root entry point of the corteX SDK. It manages LLM provider connections, global configuration, tenant-isolated plugin registries, and agent creation. Every corteX application starts by instantiating an Engine.

The Engine is designed to be created once per application (or once per tenant in multi-tenant SaaS) and shared across all agents.

## Class

### `Engine`

The root corteX engine. Manages providers, plugin isolation, and global config.

#### Constructor

```python
Engine(
    providers: Optional[Dict[str, Dict[str, Any]]] = None,
    enterprise_config: Optional[EnterpriseConfig] = None,
    orchestrator_model: Optional[str] = None,
    worker_model: Optional[str] = None,
    weight_persistence_path: Optional[str] = None,
    tenant_id: Optional[str] = None,
    allowed_plugin_roots: Optional[List[str]] = None,
)
```

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `providers` | `Optional[Dict[str, Dict[str, Any]]]` | `None` | Provider configurations keyed by provider name. Each value is a dict with `api_key`, optional `base_url`, `default_model`, and `organization`. |
| `enterprise_config` | `Optional[EnterpriseConfig]` | `None` | Global enterprise-level settings (safety, audit, compliance). Applied to all agents unless overridden at agent level. |
| `orchestrator_model` | `Optional[str]` | `None` | Default model for orchestrator/reasoning tasks (e.g., `"gpt-4o"`, `"gemini-3-pro-preview"`). |
| `worker_model` | `Optional[str]` | `None` | Default model for worker/tool-execution tasks (e.g., `"gpt-4o-mini"`, `"gemini-3-flash-preview"`). |
| `weight_persistence_path` | `Optional[str]` | `None` | File path for persisting learned behavioral weights across sessions. |
| `tenant_id` | `Optional[str]` | `None` | Unique identifier for tenant isolation. Each tenant gets an isolated plugin registry. |
| `allowed_plugin_roots` | `Optional[List[str]]` | `None` | List of filesystem directories from which dynamic plugin loading is permitted. Security control to prevent arbitrary code loading. |

#### Example: Basic Setup

```python
import cortex

engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
    }
)
```

#### Example: Multi-Provider with Model Selection

```python
import cortex

engine = cortex.Engine(
    providers={
        "openai": {
            "api_key": "sk-...",
            "default_model": "gpt-4o",
        },
        "gemini": {
            "api_key": "AIza...",
        },
    },
    orchestrator_model="gpt-4o",
    worker_model="gemini-3-flash-preview",
)
```

#### Example: Enterprise Multi-Tenant

```python
import cortex
from cortex import EnterpriseConfig

engine = cortex.Engine(
    providers={"openai": {"api_key": tenant_api_key}},
    enterprise_config=EnterpriseConfig(
        safety_level="strict",
        audit_log=True,
        blocked_topics=["competitor_data"],
        max_tokens_per_session=50000,
    ),
    tenant_id="tenant_acme_corp",
    allowed_plugin_roots=["/opt/plugins/acme"],
)
```

---

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `router` | `LLMRouter` | The LLM routing layer that manages provider connections, model selection, and failover. |
| `enterprise_config` | `Optional[EnterpriseConfig]` | Global enterprise configuration, inherited by agents that do not specify their own. |
| `weight_persistence_path` | `Optional[str]` | Path for persisting learned weights to disk. |
| `plugin_registry` | `PluginRegistry` | Tenant-isolated plugin registry. Pre-loaded with built-in plugins; additional plugins can be registered per-engine. |

---

#### Methods

##### `create_agent`

```python
def create_agent(
    self,
    name: str,
    system_prompt: str = "",
    tools: Optional[List[ToolWrapper]] = None,
    goal_tracking: bool = True,
    loop_prevention: bool = True,
    weight_config: Optional[WeightConfig] = None,
    enterprise_config: Optional[EnterpriseConfig] = None,
    context_config: Optional[ContextManagementConfig] = None,
    enable_session_recording: bool = False,
) -> Agent
```

Create a new agent template bound to this engine.

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | (required) | Unique name for the agent (e.g., `"support"`, `"code_reviewer"`). Used in logging and identification. |
| `system_prompt` | `str` | `""` | System prompt that defines the agent's persona and behavior. Injected at the start of every session. |
| `tools` | `Optional[List[ToolWrapper]]` | `None` | List of tools available to the agent. Created via the `@tool` decorator or `ToolWrapper` directly. |
| `goal_tracking` | `bool` | `True` | Enable goal progress tracking and drift detection. |
| `loop_prevention` | `bool` | `True` | Enable loop detection via state hashing and drift scoring. |
| `weight_config` | `Optional[WeightConfig]` | `None` | Initial behavioral weight configuration. Controls verbosity, formality, autonomy, and learning rates. |
| `enterprise_config` | `Optional[EnterpriseConfig]` | `None` | Agent-level enterprise config. Overrides the engine-level config if provided. |
| `context_config` | `Optional[ContextManagementConfig]` | `None` | Configuration for the Cortical Context Engine (token budgets, summarization intervals). |
| `enable_session_recording` | `bool` | `False` | Enable session recording for digital twin replay and what-if simulation. |

**Returns**: `Agent` - A configured agent template ready to start sessions.

**Example**:

```python
from corteX.tools.decorator import tool

@tool(name="search_docs", description="Search the documentation")
def search_docs(query: str) -> str:
    return f"Results for: {query}"

agent = engine.create_agent(
    name="support",
    system_prompt="You are a helpful customer support agent.",
    tools=[search_docs],
    goal_tracking=True,
    weight_config=cortex.WeightConfig(
        verbosity=0.3,
        autonomy=0.7,
    ),
)
```

---

#### Provider Resolution

The engine automatically resolves provider names to internal types:

| Provider Name | Aliases | Internal Type |
|---------------|---------|---------------|
| `"openai"` | `"azure"` | `ProviderType.OPENAI` |
| `"gemini"` | `"google"` | `ProviderType.GEMINI` |
| `"anthropic"` | `"claude"` | `ProviderType.ANTHROPIC` |
| `"local"` | `"ollama"`, `"vllm"` | `ProviderType.LOCAL` |

Unrecognized provider names default to `ProviderType.OPENAI` (OpenAI-compatible API format).

---

## Complete Example

```python
import asyncio
import cortex
from corteX.tools.decorator import tool

# 1. Define tools
@tool(name="lookup_order", description="Look up order by ID")
def lookup_order(order_id: str) -> str:
    return f"Order {order_id}: Shipped, arriving tomorrow"

# 2. Create engine with providers
engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
    },
    orchestrator_model="gpt-4o",
)

# 3. Create agent
agent = engine.create_agent(
    name="support",
    system_prompt="You help customers track their orders.",
    tools=[lookup_order],
)

# 4. Start session and run
async def main():
    session = agent.start_session(user_id="customer_42")
    response = await session.run("Where is my order #12345?")
    print(response.content)
    print(f"Latency: {response.metadata.latency_ms:.0f}ms")
    await session.close()

asyncio.run(main())
```

---

## See Also

- [Agent API Reference](./agent.md) - Agent template configuration
- [Session API Reference](./session.md) - Live conversation sessions
- [WeightConfig / EnterpriseConfig](./response.md) - Configuration classes
- [Quick Start Guide](../getting-started/quickstart.md) - Getting started tutorial
- [Multi-Provider Failover Tutorial](../tutorials/multi-provider.md) - Advanced provider setup
