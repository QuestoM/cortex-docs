# Memory Fabric

The Memory Fabric provides three biologically-inspired memory systems -- working, episodic, and semantic -- unified under a single API with pluggable storage backends. All memory operations work fully offline with zero cloud dependency.

## What It Does

The Memory Fabric answers three distinct questions:

| Memory System | Brain Region | Question |
|--------------|-------------|----------|
| **Working Memory** | Prefrontal Cortex | "What is relevant right now?" |
| **Episodic Memory** | Hippocampus | "What happened when we tried this before?" |
| **Semantic Memory** | Neocortex | "What do we know about this topic?" |

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Three Memory Systems"
    "You wake up in the morning -- your brain needs to re-learn itself. You're not exactly the same person as before..." -- Prof. Idan Segev

    The brain's memory is not a single system but a collection of specialized systems:

    - **Working memory** (prefrontal cortex): Holds approximately 4 items in active focus. The "mental workspace" where reasoning happens. Limited capacity (about 7 +/- 2 items per Miller's Law), fast access, volatile -- lost when attention shifts.
    - **Episodic memory** (hippocampus): Records personal experiences with spatial and temporal context. "I remember debugging that 401 error last Tuesday." Supports pattern matching: "This situation is similar to that time when..."
    - **Semantic memory** (neocortex): Stores factual knowledge abstracted from specific episodes. "REST APIs use HTTP methods for CRUD operations." Stable, general, shared across contexts.

    During sleep, the hippocampus "replays" important episodic memories, transferring them to the neocortex for long-term semantic storage. This consolidation process is what the Memory Fabric's `consolidate()` method implements.

## How It Works

### Working Memory

Limited capacity, fast access, volatile. Used for the current session's state:

```python
from corteX.engine.memory import MemoryFabric

fabric = MemoryFabric(working_capacity=100)

# Store current session state
fabric.working.store("current_goal", "Build a REST API", importance=0.9)
fabric.working.store("active_file", "auth_handler.py", importance=0.7)
fabric.working.store("last_error", "ImportError: jwt", importance=0.8)

# Recall by key
goal = fabric.working.recall("current_goal")
# "Build a REST API"

# Search by content
results = fabric.working.search("error", max_results=3)
# Returns MemoryItems ranked by relevance * importance

# Automatic eviction when capacity exceeded
# Least important + least recently accessed items are evicted
```

### Episodic Memory

Records past experiences as trajectories with outcomes:

```python
from corteX.engine.memory import EpisodicMemory

# Record a completed episode
fabric.episodic.record(EpisodicMemory(
    episode_id="ep_001",
    goal="Implement JWT authentication",
    steps=[
        {"action": "Read docs", "tool": "web_search"},
        {"action": "Write handler", "tool": "code_interpreter"},
        {"action": "Run tests", "tool": "shell_exec"},
    ],
    outcome="JWT auth working with refresh tokens",
    success=True,
    quality=0.9,
    tags=["authentication", "jwt", "python"],
))

# Find similar past experiences
similar = fabric.episodic.recall_similar("Add OAuth2 authentication")
# Returns episodes with goals matching the query

# Find only successful episodes
successful = fabric.episodic.recall_successful("authentication")

# Check success rate for a domain
rate = fabric.episodic.get_success_rate(tag="authentication")
# 0.85
```

### Semantic Memory

Long-term domain knowledge that persists across sessions:

```python
from corteX.engine.memory import SemanticEntry

# Store domain knowledge
fabric.semantic.learn(SemanticEntry(
    entry_id="jwt_basics",
    topic="JWT Authentication",
    content="JSON Web Tokens use HMAC-SHA256 or RSA for signing. "
            "Access tokens are short-lived; refresh tokens are long-lived.",
    source="learned_from_episode_ep_001",
    confidence=0.9,
))

# Query knowledge
entries = fabric.semantic.query("token authentication")
# Returns SemanticEntry objects ranked by relevance

# Update existing knowledge
fabric.semantic.update("jwt_basics", content="...", confidence=0.95)
```

### Pluggable Backends

Every memory system accepts a pluggable backend:

```python
from corteX.engine.memory import (
    MemoryFabric,
    InMemoryBackend,
    FileBackend,
)

# In-memory (default): fast, volatile
fabric = MemoryFabric()

# File-based: persistent, on-prem compatible
fabric = MemoryFabric(
    working_backend=None,  # Working memory is volatile by design
    episodic_backend=FileBackend("/data/cortex/episodic"),
    semantic_backend=FileBackend("/data/cortex/semantic"),
)
```

The `FileBackend` stores each memory item as a JSON file:
- File names are MD5 hashes of the key (safe for all filesystems)
- Items are cached in memory after initial load
- Writes are synchronous (crash-safe)
- No database server required

To implement a custom backend (Redis, SQLite, Postgres), extend `MemoryBackend`:

```python
from corteX.engine.memory import MemoryBackend, MemoryItem

class RedisBackend(MemoryBackend):
    def get(self, key: str) -> Optional[MemoryItem]: ...
    def put(self, item: MemoryItem) -> None: ...
    def delete(self, key: str) -> bool: ...
    def search(self, query: str, max_results: int = 5) -> List[MemoryItem]: ...
    def list_all(self) -> List[MemoryItem]: ...
    def clear(self) -> None: ...
    def count(self) -> int: ...
```

### Consolidation

Sleep-like consolidation promotes important working memories to long-term storage:

```python
consolidated = fabric.consolidate()
# Items with importance >= 0.7 are promoted:
#   - Items tagged "knowledge" -> semantic memory
#   - Items tagged "experience" -> episodic memory

# At session end: consolidate then clear working memory
fabric.new_session()
```

### Cross-Memory Search

Search across all three memory systems at once:

```python
context = fabric.get_relevant_context("JWT authentication", max_items=10)
# {
#   "working": [{"key": "last_error", "content": "jwt import failed"}],
#   "episodic": [{"goal": "Implement JWT", "outcome": "...", "success": True}],
#   "semantic": [{"topic": "JWT Authentication", "content": "..."}],
# }
```

### Memory Statistics

```python
stats = fabric.get_stats()
# {
#   "working": {"items": 12, "capacity": 100},
#   "episodic": {"items": 45, "capacity": 500, "success_rate": 0.82},
#   "semantic": {"items": 128, "capacity": 1000},
# }
```

## When It Activates

- **Every step**: Working memory stores current step state (active file, goal, errors)
- **After task completion**: Episodic memory records the trajectory and outcome
- **During learning**: Semantic memory stores extracted domain knowledge
- **At session end**: Consolidation promotes important working memories, then clears
- **On context packing**: The Memory Fabric provides relevant context to the CCE

## API Reference

```python
from corteX.engine.memory import (
    MemoryFabric,
    WorkingMemory,
    EpisodicStore,
    SemanticStore,
    MemoryItem,
    EpisodicMemory,
    SemanticEntry,
    MemoryQuery,
    MemoryStats,
    MemoryBackend,
    InMemoryBackend,
    FileBackend,
)
```
