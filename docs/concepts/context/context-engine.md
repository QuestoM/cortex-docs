# Cortical Context Engine

The Cortical Context Engine (CCE) manages what information occupies the LLM context window at each step of a long-running agent workflow. It implements a three-temperature memory hierarchy with progressive compression, inspired by CPU cache architecture and the brain's hierarchical memory systems.

## What It Does

For workflows that span thousands of steps, the CCE:

1. **Classifies context items** into hot, warm, and cold tiers based on recency and importance
2. **Progressively compresses** old items through four levels (verbatim -> masked -> summarized -> digested)
3. **Packs the optimal context window** by filling each tier's budget with the most valuable items
4. **Checkpoints state** for fault-tolerant recovery from corrupted context

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Hierarchical Memory Systems"
    The brain maintains multiple memory systems operating at different timescales:

    - **Working memory** (prefrontal cortex): Holds ~4 items in active focus. Fast access, limited capacity, volatile.
    - **Short-term buffer** (hippocampus): Bridges working memory and long-term storage. Intermediate capacity, hours to days.
    - **Long-term memory** (neocortex): Vast capacity, slow formation, stable for years.

    Information flows through these systems via consolidation: important working memories are replayed during sleep and transferred to long-term storage. Unimportant memories fade.

    The CCE implements the same hierarchy: hot memory is the prefrontal "workspace," warm memory is the hippocampal buffer, and cold memory is the neocortical archive. The `compress()` method implements consolidation.

## How It Works

### Three-Temperature Hierarchy

```python
from corteX.engine.context import CorticalContextEngine, ContextConfig

config = ContextConfig(
    token_budget_ratio=0.80,       # Use 80% of model's context window
    hot_ratio=0.40,                # Recent turns: 40% of budget
    warm_ratio=0.35,               # Compressed history: 35%
    cold_ratio=0.25,               # Retrieved archives: 25%
    output_reservation=4096,       # Reserve tokens for model response
)

cce = CorticalContextEngine(config=config)
```

| Tier | Budget | Contains | Compression |
|------|--------|----------|-------------|
| **Hot** | 40% | Recent turns, active tool results | L0 (verbatim) |
| **Warm** | 35% | Compressed history, key decisions, task state | L1-L2 |
| **Cold** | 25% | Archived history, retrieved by relevance | L2-L3 |

### Progressive Compression

Four compression levels, each reducing token usage more aggressively:

| Level | Age | Ratio | Method |
|-------|-----|-------|--------|
| **L0 Verbatim** | 0-10 steps | 1:1 | Full content |
| **L1 Condensed** | 11-50 steps | 3:1 to 5:1 | Observation masking (tool outputs replaced with placeholders) |
| **L2 Summary** | 51-200 steps | 10:1 to 20:1 | LLM-generated summary of decision points |
| **L3 Digest** | 200+ steps | 50:1 to 100:1 | Structured digest (goals + lessons only) |

The key research insight (JetBrains NeurIPS 2025): L1 observation masking achieves 50%+ cost reduction without the trajectory elongation (+13-15% more steps) caused by LLM summarization.

```python
# Compression runs automatically or manually
compressed_count = cce.compress()
```

### Importance Scoring

Every context item receives a composite importance score:

```
importance = w_r * recency(item)
           + w_v * relevance(item, current_goal)
           + w_c * causal_weight(item)
           + w_f * reference_frequency(item)
           + w_s * success_correlation(item)
           + w_d * domain_weight(item)
```

Configurable weights with sensible defaults:

```python
from corteX.engine.context import ImportanceWeights

weights = ImportanceWeights(
    recency=0.25,
    relevance=0.25,
    causal=0.20,
    reference_frequency=0.10,
    success_correlation=0.10,
    domain=0.10,
)
```

### Context Window Packing

The packer assembles items from all three tiers into the optimal context window:

```python
# Add items during the workflow
cce.add_user_message(step=42, content="Now add JWT validation")
cce.add_assistant_message(step=42, content="I'll implement JWT...")
cce.add_tool_result(step=42, tool_name="file_write", content="...")

# Pack the context for the next LLM call
packed = cce.get_context_window(model_context_window=200000)

# Ordering: system_prompt -> task_state -> warm -> cold -> hot
# Stable context early (primacy bias), recent turns last (recency bias)
```

### Task State

The CCE maintains a structured `TaskState` that persists through compression:

```python
from corteX.engine.context import TaskState

# Updated automatically during compression
state = cce.task_state
state.current_goal        # "Build a REST API"
state.sub_goals           # [{"goal": "Login endpoint", "status": "done"}, ...]
state.decisions_made      # [{"decision": "Use JWT", "rationale": "...", "step": "15"}]
state.active_entities     # {"auth_handler.py": "implemented", ...}
state.progress_percentage # 65.0
state.error_patterns      # [{"error": "401 on /api", "resolution": "Fixed token validation"}]
```

### Domain Profiles

Compression is domain-aware. Built-in profiles for coding and research:

```python
from corteX.engine.context import CODING_PROFILE, RESEARCH_PROFILE

# Coding profile preserves code snippets but aggressively compresses pip output
cce = CorticalContextEngine(profile=CODING_PROFILE)

# Research profile preserves citations but compresses navigation steps
cce = CorticalContextEngine(profile=RESEARCH_PROFILE)
```

### Checkpointing

Periodic checkpoints enable recovery from corrupted context:

```python
# Manual checkpoint
cp = cce.create_checkpoint()

# Automatic (every 50 steps by default)
# Configured via ContextConfig.checkpoint_every_n_steps

# Get stats
budget = cce.get_token_budget_status(model_context_window=200000)
# {
#   "total_budget": 160000,
#   "used": 95000,
#   "available": 65000,
#   "utilization": 0.59,
# }
```

## When It Activates

- **Before every LLM call (all modes)**: `get_context_window()` packs the optimal context. This applies to `session.run()` (chat), `session.run_agentic()` (agentic), and `session.run_stream()` (streaming). The 4-zone `ContextCompiler` is wired into all three execution paths.
- **Every N steps** (configurable): `compress()` runs progressive compression. L2 summaries are generated by sending prompts to the LLM, and L3 structured digests are built from those summaries.
- **Every M steps**: Checkpoints are created for recovery
- **On item addition**: Items are scored for importance and assigned to the hot tier
- **During compression**: Items flow from hot -> warm -> cold based on age

## API Reference

```python
from corteX.engine.context import (
    CorticalContextEngine,
    ContextConfig,
    ContextItem,
    ContextItemType,
    CompressionLevel,
    CompressionProfile,
    TaskState,
    TokenCounter,
    ImportanceScorer,
    ImportanceWeights,
    ObservationMasker,
    ContextWindowPacker,
    ContextCheckpointer,
    CODING_PROFILE,
    RESEARCH_PROFILE,
)
```
