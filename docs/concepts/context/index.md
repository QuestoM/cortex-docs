# Context and Memory

AI agents that run for thousands of steps face a fundamental challenge: the context window is finite, but the history of decisions, tool outputs, and user instructions grows without bound. The corteX Context and Memory system solves this with brain-inspired hierarchical memory management.

## Architecture Overview

```
              +-----------------+
              | Working Memory  |  Hot: 40% budget
              | (Prefrontal     |  Recent turns, active tools
              |  Cortex)        |  Volatile, fast access
              +-----------------+
                      |
              +-----------------+
              | Episodic Memory |  Warm: 35% budget
              | (Hippocampus)   |  Compressed history, key decisions
              |                 |  Cross-session, pattern matching
              +-----------------+
                      |
              +-----------------+
              | Semantic Memory |  Cold: 25% budget
              | (Neocortex)     |  Domain knowledge, facts
              |                 |  Long-term, stable
              +-----------------+
```

## Three Subsystems

| Subsystem | Page | What It Does |
|-----------|------|--------------|
| [Context Engine](context-engine.md) | 3-temperature hierarchy | Manages what occupies the LLM context window at each step |
| [Memory Fabric](memory.md) | Pluggable memory stores | Provides working, episodic, and semantic memory with pluggable backends |
| [Cross-Modal Associations](cross-modal.md) | Hebbian binding | Connects information across modalities (code, errors, docs, preferences) |

## Design Principles

**1. Progressive compression.** Not all history is equally valuable. Recent turns are kept verbatim (L0). Older tool outputs are masked (L1). Distant history is summarized (L2) or digested (L3). This follows the JetBrains NeurIPS 2025 finding that observation masking outperforms LLM summarization.

**2. Pluggable backends.** Every memory store works with a pluggable backend interface. The default is in-memory. For persistence, use `FileBackend`. For scale, implement `MemoryBackend` against Redis, SQLite, or any storage system. All backends work fully offline.

**3. Sleep-like consolidation.** At session boundaries, important working memories are "replayed" and promoted to episodic or semantic memory -- just as the brain consolidates memories during sleep by replaying hippocampal traces in the neocortex.

**4. Primacy and recency bias.** The context packer places stable context early in the window (primacy bias) and recent turns last (recency bias). This follows Chroma's 2025 research showing that LLMs recall information better at the beginning and end of the context.

**5. No cloud dependency.** All memory operations are local. `FileBackend` uses JSON files on disk. No database server required. Fully air-gapped compatible.

## Quick Start

```python
from corteX.engine.memory import MemoryFabric, FileBackend, EpisodicMemory
from corteX.engine.context import CorticalContextEngine, ContextConfig

# Memory Fabric with persistent storage
fabric = MemoryFabric(
    working_backend=None,  # In-memory (volatile)
    episodic_backend=FileBackend("/data/cortex/episodic"),
    semantic_backend=FileBackend("/data/cortex/semantic"),
)

# Context Engine for LLM context management
cce = CorticalContextEngine(config=ContextConfig())
cce.set_goal("Build a REST API with authentication")

# On each step
cce.add_user_message(step=1, content="Implement the login endpoint")
cce.add_tool_result(step=1, tool_name="file_write", content="...")

# Get optimally packed context for LLM call
packed = cce.get_context_window(model_context_window=200000)

# At session end: consolidate and persist
fabric.new_session()
```
