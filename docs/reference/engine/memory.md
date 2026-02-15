# Memory

`corteX.memory.manager` -- Context Broker and production memory management.

---

## Overview

The **ContextBroker** is the "Librarian" of the corteX system. It provides a unified interface
for storing and retrieving knowledge, session state, and execution trajectories. In production,
it uses Gemini's File Search API for RAG and Context Caching for efficient session management.
For development, an in-memory fallback is available.

Each `ContextBroker` instance is independent (no shared state). The module-level `broker`
variable exists for backward compatibility with Langflow integrations but is deprecated --
new code should create a `ContextBroker()` per tenant or session.

---

## Class: ContextBroker

```python
from corteX.memory.manager import ContextBroker
```

The main memory interface that integrates RAG document retrieval, trajectory storage,
knowledge graph slicing, and session management.

### Constructor

```python
ContextBroker(use_gemini_backend: bool = True)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `use_gemini_backend` | `bool` | `True` | When `True`, initializes Vertex AI Search or Gemini FileSearch drivers. When `False`, uses in-memory-only storage. |

### Attributes

| Name | Type | Description |
|------|------|-------------|
| `use_gemini` | `bool` | Whether Gemini-backed drivers are active. |
| `_short_term_memory` | `Dict[str, Any]` | In-memory key-value session store. |
| `_trajectories` | `List[Trajectory]` | Stored execution trajectories. |
| `_graph_nodes` | `Dict[str, EntityNode]` | Knowledge graph entity nodes. |
| `_file_search` | `Optional[IMemoryDriver]` | The active RAG search driver (Vertex or FileSearch). |
| `_context_cache` | `Optional[ContextCacheDriver]` | Context caching driver for long sessions. |

### Methods

#### `get_relevant_context`

```python
async def get_relevant_context(self, task_description: str) -> Dict[str, Any]
```

Retrieve relevant context for a task using real RAG. Queries the FileSearch driver
for documents and trajectories, then assembles a context bundle.

| Parameter | Type | Description |
|-----------|------|-------------|
| `task_description` | `str` | Natural language description of the current task. |

**Returns:** `Dict[str, Any]` -- Bundle containing keys: `trajectories`, `knowledge_graph`,
`session`, `retrieved_docs`.

#### `commit_trajectory`

```python
async def commit_trajectory(self, trajectory: Trajectory) -> None
```

Save a successful execution trajectory for future learning. Stores both in-memory
and in the FileSearch index for cross-session persistence.

| Parameter | Type | Description |
|-----------|------|-------------|
| `trajectory` | `Trajectory` | The completed execution trajectory to persist. |

#### `store_knowledge`

```python
async def store_knowledge(self, doc_id: str, content: str, metadata: Optional[Dict] = None) -> bool
```

Store a knowledge document for RAG retrieval.

| Parameter | Type | Description |
|-----------|------|-------------|
| `doc_id` | `str` | Unique identifier for the document. |
| `content` | `str` | Document text content. |
| `metadata` | `Optional[Dict]` | Optional metadata tags. |

**Returns:** `bool` -- `True` if stored successfully, `False` if no backend is available.

#### `create_session_cache`

```python
async def create_session_cache(self, session_id: str, history: List[str]) -> Optional[str]
```

Create a context cache for a long conversation session. Useful for maintaining context
without re-sending full history on every LLM call. Default TTL is 3600 seconds (1 hour).

| Parameter | Type | Description |
|-----------|------|-------------|
| `session_id` | `str` | Unique session identifier. |
| `history` | `List[str]` | Conversation history strings to cache. |

**Returns:** `Optional[str]` -- The cache ID, or `None` if caching is unavailable.

#### `update_session`

```python
def update_session(self, key: str, value: Any) -> None
```

Update a key-value pair in short-term session memory.

#### `get_session`

```python
def get_session(self, key: str, default: Any = None) -> Any
```

Get a value from session memory. Returns `default` if the key does not exist.

#### `clear_session`

```python
def clear_session(self) -> None
```

Clear all short-term session memory.

#### `add_graph_node`

```python
def add_graph_node(self, node: EntityNode) -> None
```

Add an entity node to the in-memory knowledge graph.

---

## Deprecated: Module-level `broker`

```python
from corteX.memory.manager import broker  # DEPRECATED
```

A lazy singleton that creates a `ContextBroker(use_gemini_backend=False)` on first
attribute access. Logs a `DeprecationWarning`. New code should instantiate
`ContextBroker()` directly.

---

## Backend Selection

The `ContextBroker` automatically selects the best available backend at initialization:

1. **Vertex AI Search** -- Used when `config.get_vertex_search_datastore()` returns a value.
   Best for production deployments with Google Cloud.
2. **Gemini FileSearch** -- Fallback when Vertex is not configured. Uses the Gemini File
   Search API directly.
3. **In-memory** -- Used when `use_gemini_backend=False`. No persistence; suitable for
   tests and development.

---

## Example

```python
import asyncio
from corteX.memory.manager import ContextBroker

async def main():
    broker = ContextBroker(use_gemini_backend=False)

    # Store session data
    broker.update_session("user_preference", "verbose")
    pref = broker.get_session("user_preference")
    # pref == "verbose"

    # Retrieve context for a task
    context = await broker.get_relevant_context("Fix the login endpoint")
    print(context["retrieved_docs"])  # [] in memory-only mode

    # Store knowledge for RAG
    await broker.store_knowledge(
        doc_id="auth_guide",
        content="OAuth2 requires a client_id and client_secret...",
        metadata={"category": "authentication"},
    )

asyncio.run(main())
```

```python
# Using session caching for long conversations
async def long_session():
    broker = ContextBroker()
    cache_id = await broker.create_session_cache(
        session_id="session_42",
        history=["User: Hello", "Agent: Hi!", "User: Help me debug auth"],
    )
    # cache_id can be passed to the LLM provider for context reuse
```
