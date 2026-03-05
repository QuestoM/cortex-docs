# Context Pyramid

The `ContextPyramid` implements multi-resolution context management, storing the same information at four detail levels and dynamically selecting the optimal resolution per item within a token budget. This allows the agent to keep more items in context by compressing less-important ones, rather than dropping them entirely.

## How It Works

When content is registered via `add_item()`, the pyramid generates four resolution levels using heuristic compression (no LLM calls required):

| Level | Name | Compression | Description |
|-------|------|-------------|-------------|
| R0 | Full | 1x tokens | Complete verbatim content |
| R1 | Standard | ~0.3x tokens | Key sentences containing entities, decisions, and code references |
| R2 | Compact | ~0.1x tokens | Entity-relationship pairs (e.g., `user_service.py: referenced`) |
| R3 | Micro | ~0.02x tokens | Single-line semantic fingerprint (first 8 words + "...") |

### Priority-Based Resolution Assignment

During the `resolve()` phase, items are sorted by priority and assigned to tiers:

- **Top 10%** of items get R0 (full verbatim content)
- **Next 20%** get R1 (key sentences)
- **Next 30%** get R2 (entity-relationship summary)
- **Bottom 40%** get R3 (single-line fingerprint)

If an item at its assigned resolution would exceed the remaining token budget, it is automatically downgraded to the next resolution level until it fits or reaches R3.

### Compression Heuristics

All compression is regex-based with zero external dependencies:

- **R1 extraction**: Scans for sentences containing decision verbs (`decided`, `chose`, `created`, `error`, `failed`, etc.), file paths, API endpoints, or Python identifiers. Keeps up to 20 key sentences.
- **R2 extraction**: Lists extracted entities with `referenced` annotations, joined by `|`.
- **R3 extraction**: Takes the first 8 words of the content, appended with `...`.

## Key Features

- **Zero LLM calls** - All compression is heuristic-based, keeping the system fast and deterministic
- **Dynamic budget fitting** - Automatically downgrades items that exceed remaining token budget
- **Per-item resolution tracking** - Each `MultiResolutionItem` records its current resolution level
- **On-demand regeneration** via `generate_resolutions()` - Regenerates R1/R2/R3 from updated R0 content
- **Direct resolution access** via `get_at_resolution()` - Retrieve any item at any specific level
- **Capacity management** - Trims oldest items when exceeding `max_items` (default 500)
- **Goal proximity awareness** - Items store a `goal_proximity` score for priority-aware resolution selection

## Integration

The `ContextPyramid` is used by the `CognitiveContextPipeline` during the resolution selection phase. After items are scored and prioritized, the pyramid assigns resolution levels within the token budget. Items with higher entanglement scores or goal relevance naturally receive higher priority and thus higher resolution.

The `ResolvedItem` output feeds into the final context assembly, where each item's content is the pyramid-selected version rather than always the full text.

## Usage Example

```python
from corteX.engine.cognitive import (
    ContextPyramid, ResolutionLevel, ResolvedItem
)

pyramid = ContextPyramid(max_items=500)

# Register content - all 4 resolutions generated automatically
item = pyramid.add_item(
    item_id="deploy_log",
    full_content="The deployment of user-service to production completed "
                 "successfully. All 142 tests passed. The service is now "
                 "running on port 8080 with health checks enabled.",
    goal_embedding_sim=0.8,
)

print(item.r0_tokens)  # Full content tokens
print(item.r1_tokens)  # ~30% of R0
print(item.r2_tokens)  # ~10% of R0
print(item.r3_tokens)  # ~2% of R0

# Resolve items within a budget
candidates = [
    ResolvedItem(item_id="deploy_log", content="", tokens=0,
                 resolution=0, priority=0.9),
    ResolvedItem(item_id="old_config", content="", tokens=0,
                 resolution=0, priority=0.2),
]
resolved = pyramid.resolve(candidates, budget_tokens=200, goal="deploy")
# High-priority items get R0, low-priority items get R2 or R3

# Access a specific resolution
r2_content = pyramid.get_at_resolution(
    "deploy_log", ResolutionLevel.R2_COMPACT
)
# "user_service: referenced | port: referenced | 8080: referenced"

# Check statistics
stats = pyramid.get_stats()
# {"total_items": 2, "by_resolution": {0: 1, 1: 0, 2: 1, 3: 0}}
```

## See Also

- [Density Optimizer](density-optimizer.md) - Applies additional compression within each resolution level
- [Entanglement Graph](entanglement.md) - Influences priority scores that drive resolution assignment
- [Context Quality Engine](quality-engine.md) - Measures information density of the assembled output
- [Predictive Pre-Loader](predictive-loader.md) - Pre-loads items at R2 resolution by default
