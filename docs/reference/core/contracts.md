# Contracts API Reference

## Module: `corteX.core.contracts`

This module defines the **core domain types and protocols** of the corteX SDK. These are the foundational building blocks -- the "language" that all components speak. Every plugin, agent, subsystem, and memory driver implements one of these contracts.

If you are building a custom plugin or extending corteX, this is the module you need to understand.

---

## Domain Enumerations

### `AgentRole`

**Type**: `str, Enum`

Roles that agents can play within the orchestration system.

| Value | String | Description |
|-------|--------|-------------|
| `STRATEGIST` | `"strategist"` | High-level planning and decision-making. |
| `WORKER` | `"worker"` | Task execution and tool operation. |
| `QA` | `"qa"` | Quality assurance and output validation. |
| `SUPERVISOR` | `"supervisor"` | Oversight and coordination of other agents. |

```python
from corteX.core.contracts import AgentRole

role = AgentRole.WORKER
assert role == "worker"
```

---

### `ArtifactType`

**Type**: `str, Enum`

Types of artifacts that agents produce and exchange.

| Value | String | Description |
|-------|--------|-------------|
| `CODE` | `"code"` | Source code output. |
| `DOCUMENT` | `"document"` | Text document or report. |
| `PLAN` | `"plan"` | Execution plan or strategy. |
| `WEB_RESOURCE` | `"web_resource"` | Web content (URLs, scraped data). |
| `ERROR` | `"error"` | Error report or failure description. |
| `DECISION` | `"decision"` | Autonomy or timer-based decision request. |
| `UI_COMPONENT` | `"ui_component"` | UI element for streaming injection. |
| `ANALYSIS` | `"analysis"` | Data analysis result. |

---

### `AutonomyLevel`

**Type**: `str, Enum`

Controls how much independence an agent has in making decisions.

| Value | String | Description |
|-------|--------|-------------|
| `COLLABORATIVE` | `"collaborative"` | Agent needs user approval before acting. |
| `HIGH` | `"high"` | Agent acts on a timer -- proceeds if no objection. |
| `AUTONOMOUS` | `"autonomous"` | Agent acts immediately without asking. |

```python
from corteX.core.contracts import AutonomyLevel

# Check if agent should ask for approval
if level == AutonomyLevel.COLLABORATIVE:
    await ask_user_for_confirmation()
```

---

## Data Models

### `Artifact`

**Type**: `pydantic.BaseModel`

The atomic unit of work exchanged between agents and subsystems. Artifacts represent the output of any agent step -- code, documents, plans, errors, or UI components.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `artifact_id` | `str` | (required) | Unique identifier for this artifact. |
| `type` | `ArtifactType` | (required) | The type of artifact. |
| `content` | `Any` | (required) | The artifact content (code string, document text, structured data). |
| `metadata` | `Dict[str, Any]` | `{}` | Additional metadata (language, version, source, etc.). |
| `visual_spec` | `Optional[Dict[str, Any]]` | `None` | UI rendering hint (e.g., `{"render_as": "table"}`, `{"render_as": "code", "language": "python"}`). |

#### Example

```python
from corteX.core.contracts import Artifact, ArtifactType

# Code artifact
code_artifact = Artifact(
    artifact_id="art_001",
    type=ArtifactType.CODE,
    content="def hello():\n    return 'world'",
    metadata={"language": "python", "lines": 2},
    visual_spec={"render_as": "code", "language": "python"},
)

# Document artifact
doc = Artifact(
    artifact_id="art_002",
    type=ArtifactType.DOCUMENT,
    content="# Analysis Report\n\nRevenue grew 15% YoY...",
    metadata={"format": "markdown"},
)
```

---

### `ContextSlice`

**Type**: `pydantic.BaseModel`

A partial view of the context relevant to a specific task. Passed to agents and subsystems to provide the information they need without exposing the full system state.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `task_id` | `str` | (required) | The task this context slice is prepared for. |
| `relevant_nodes` | `List[Dict[str, Any]]` | `[]` | Relevant nodes from the knowledge graph. |
| `history` | `List[Dict[str, Any]]` | `[]` | Relevant chat history entries. |
| `artifacts` | `List[Artifact]` | `[]` | Previously produced artifacts relevant to this task. |
| `session_memory` | `str` | `""` | Short-term session state summary. |
| `working_memory` | `Dict[str, Any]` | `{}` | Active working memory contents. |
| `retrieved_docs` | `List[str]` | `[]` | RAG retrieval results. |

#### Example

```python
from corteX.core.contracts import ContextSlice, Artifact, ArtifactType

context = ContextSlice(
    task_id="task_analyze_sales",
    history=[
        {"role": "user", "content": "Analyze Q4 sales"},
        {"role": "assistant", "content": "I'll pull the data..."},
    ],
    artifacts=[
        Artifact(
            artifact_id="prev_data",
            type=ArtifactType.ANALYSIS,
            content={"revenue": 1_500_000, "growth": 0.15},
        ),
    ],
    session_memory="User is the VP of Sales. Prefers visual summaries.",
    retrieved_docs=["Q4 2025 sales report excerpt..."],
)
```

---

## Core Protocols

These protocols define the interfaces that all extensible components must implement.

### `IPlugin`

**Type**: `Protocol`

Marker interface for all loadable plugins. Every agent, subsystem, and memory driver implements this.

```python
class IPlugin(Protocol):
    @property
    def name(self) -> str: ...

    @property
    def version(self) -> str: ...

    def initialize(self, config: Dict[str, Any]) -> None: ...
```

| Property/Method | Type | Description |
|-----------------|------|-------------|
| `name` | `str` (property) | Unique plugin name. |
| `version` | `str` (property) | Semantic version string. |
| `initialize(config)` | `None` | Called once after registration with plugin-specific configuration. |

---

### `ILLMProvider`

**Type**: `ABC`

Interface for raw LLM access. Implementations exist for OpenAI, Gemini, Anthropic, and local models.

```python
class ILLMProvider(ABC):
    @abstractmethod
    async def generate_text(
        self, prompt: str, system_message: str = ""
    ) -> str: ...

    @abstractmethod
    async def generate_structured(
        self, prompt: str, response_model: type[BaseModel]
    ) -> BaseModel: ...
```

| Method | Returns | Description |
|--------|---------|-------------|
| `generate_text(prompt, system_message)` | `str` | Generate free-form text from a prompt. |
| `generate_structured(prompt, response_model)` | `BaseModel` | Generate a structured Pydantic model response. |

---

### `IMemoryDriver`

**Type**: `ABC`

Interface for storage backends (vector databases, graph databases, Redis, etc.).

```python
class IMemoryDriver(ABC):
    @abstractmethod
    async def store(
        self, collection: str, key: str, value: Any,
        vectors: Optional[List[float]] = None,
    ) -> None: ...

    @abstractmethod
    async def retrieve(
        self, collection: str, query: str
    ) -> List[Any]: ...
```

| Method | Returns | Description |
|--------|---------|-------------|
| `store(collection, key, value, vectors)` | `None` | Store a value with optional embedding vectors. |
| `retrieve(collection, query)` | `List[Any]` | Retrieve relevant items by query. |

---

### `ISubsystem`

**Type**: `ABC, IPlugin`

An intelligent tool with its own internal reasoning loop. Unlike simple tools (which are stateless functions), a subsystem has minimal autonomy and can execute multi-step operations internally.

```python
class ISubsystem(ABC, IPlugin):
    @abstractmethod
    async def execute(
        self, intent: str, context: ContextSlice
    ) -> Artifact: ...
```

| Method | Returns | Description |
|--------|---------|-------------|
| `execute(intent, context)` | `Artifact` | Execute a high-level intent and return an artifact. |

**Example**: A web research subsystem receives `"Research Python web frameworks"` and returns an `Artifact(type=DOCUMENT, content="# Research Report\n...")` after internally performing multiple searches and synthesis steps.

---

### `IAgent`

**Type**: `ABC, IPlugin`

A domain agent capable of reasoning and delegation.

```python
class IAgent(ABC, IPlugin):
    role: AgentRole

    @abstractmethod
    async def step(
        self, input_signal: Union[str, Artifact], context: ContextSlice
    ) -> Artifact: ...
```

| Attribute/Method | Type | Description |
|------------------|------|-------------|
| `role` | `AgentRole` | The agent's role in the system. |
| `step(input_signal, context)` | `Artifact` | Process one reasoning step and produce an artifact. |

---

### `ILogInterceptor`

**Type**: `Protocol`

Hook for detecting silent errors in tool outputs.

```python
class ILogInterceptor(Protocol):
    def on_tool_output(
        self, tool_name: str, input_args: Dict, output: Any
    ) -> Optional[str]: ...
```

| Method | Returns | Description |
|--------|---------|-------------|
| `on_tool_output(tool_name, input_args, output)` | `Optional[str]` | Returns `None` if output is valid, or a correction string if a silent error was detected. |

---

## Implementing a Custom Subsystem

```python
from corteX.core.contracts import ISubsystem, ContextSlice, Artifact, ArtifactType
from corteX.core.registry import PluginRegistry
from typing import Any, Dict

@PluginRegistry.register
class MySearchSubsystem(ISubsystem):
    @property
    def name(self) -> str:
        return "my_search"

    @property
    def version(self) -> str:
        return "1.0.0"

    def initialize(self, config: Dict[str, Any]) -> None:
        self.api_key = config.get("api_key", "")

    async def execute(self, intent: str, context: ContextSlice) -> Artifact:
        # Your multi-step search logic here
        results = await self._search(intent)
        return Artifact(
            artifact_id=f"search_{context.task_id}",
            type=ArtifactType.DOCUMENT,
            content=results,
            metadata={"query": intent, "result_count": len(results)},
        )
```

---

## See Also

- [EventBus API Reference](./events.md) - Event system
- [Registry API Reference](./registry.md) - Plugin registration
- [Architecture Overview](../../concepts/architecture.md) - System design
- [Custom Tools Guide](../../guides/tools/custom-tools.md) - Building tools
