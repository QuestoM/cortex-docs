# Predictive Pre-Loader

The `PredictivePreLoader` implements CPU-cache-inspired context prefetching, predicting what context items will be needed 2-3 turns ahead and pre-loading them into a buffer. By anticipating future needs, the agent avoids cold-start delays where critical context must be assembled from scratch.

## How It Works

The pre-loader uses three prediction signals - none of which require LLM calls:

### Signal 1: Plan Step Lookahead

If the agent has an execution plan with discrete steps, the pre-loader extracts entities from the next 2-3 upcoming steps and generates prefetch candidates. Relevance decays with distance: the next step gets 0.7 relevance, two steps ahead gets 0.6, and three steps ahead gets 0.5.

### Signal 2: Entity Co-Occurrence

The pre-loader maintains a co-occurrence index mapping entities to other entities that have historically appeared alongside them. When current context mentions entity A, and entity B has frequently co-occurred with A in the past, B is predicted as a prefetch candidate with 0.5 relevance.

### Signal 3: Error Pattern Matching

Known error patterns and their resolutions are recorded via `record_error_resolution()`. When current context contains error keywords (`error`, `fail`) that match a known signature, the resolution context is predicted as needed with 0.7 relevance.

### Self-Tuning Threshold

The pre-loader automatically adjusts its minimum relevance threshold based on observed hit/miss rates:

- If hit rate exceeds 70%, the threshold is lowered by 0.02 (prefetch more aggressively)
- If hit rate drops below 30%, the threshold is raised by 0.02 (be more selective)
- The threshold is bounded between a floor of 0.3 and a ceiling of 0.9
- Adjustment only begins after at least 10 predictions (to avoid premature tuning)

### Buffer Management

Prefetched items are stored in a bounded buffer (default 30 items). Items are evicted when they are 3+ steps past their predicted need step or when they have been promoted to active context.

## Key Features

- **Three complementary prediction signals** - Plan lookahead, entity co-occurrence, and error patterns
- **Self-tuning relevance threshold** that adapts based on prediction accuracy
- **Hit/miss tracking** for monitoring prefetch effectiveness
- **Bounded prefetch buffer** with automatic stale entry eviction
- **Content registration** - Pipeline populates actual content for buffered items via `register_content()`
- **Promotion tracking** via `promote()` and `check_hits()` - Distinguishes between predicted-and-used vs predicted-and-wasted
- **Zero LLM calls** - All prediction is lightweight regex and lookup-based

## Integration

The `PredictivePreLoader` is called by the `CognitiveContextPipeline` at the beginning of each compilation cycle. The `prefetch()` method generates candidates based on the current execution step, plan state, and recent context messages. The pipeline then populates content for buffered items and checks which previous predictions were correct via `check_hits()`.

Items in the prefetch buffer are loaded at R2 (compact) resolution by default, balancing memory cost against readiness. When an item is actually needed, the `ContextPyramid` can upgrade it to R0 or R1.

The co-occurrence index is fed by the `EntanglementGraph`'s entity extraction, and error patterns are recorded from the `StateFileManager`'s error journal.

## Usage Example

```python
from corteX.engine.cognitive import PredictivePreLoader

loader = PredictivePreLoader(
    buffer_size=30,
    lookahead_steps=3,
    min_relevance=0.3,
)

# Record known error patterns for Signal 3
loader.record_error_resolution(
    error_signature="ConnectionRefusedError",
    resolution="Check if the service is running on the expected port",
)

# Update co-occurrence index for Signal 2
loader.update_cooccurrence(["user_service.py", "database.py", "config.yaml"])

# Run prefetch at each step
candidates = loader.prefetch(
    step_number=5,
    plan_state={
        "steps": [
            {"description": "Build user_service"},
            {"description": "Run tests"},
            {"description": "Deploy to staging"},    # step 6
            {"description": "Run smoke tests"},      # step 7
            {"description": "Deploy to production"}, # step 8
        ],
        "current_step_index": 2,
    },
    current_context=[
        {"role": "tool", "content": "Build succeeded for user_service.py"},
    ],
)

# Pipeline populates content for buffered items
for candidate in candidates:
    loader.register_content(candidate.item_id, "actual content here")

# Check which predictions were correct
hits = loader.check_hits(["plan_deploy_staging_6"])

# Monitor performance
status = loader.get_status()
print(f"Hit rate: {status['hit_rate']:.0%}")
print(f"Buffer: {status['buffer_size']} items")
print(f"Threshold: {status['min_relevance_threshold']:.2f}")
```

## See Also

- [Entanglement Graph](entanglement.md) - Provides entity relationships used for co-occurrence prediction
- [Context Pyramid](pyramid.md) - Manages resolution levels for prefetched items
- [State File Manager](state-file-manager.md) - Supplies error journal for error pattern matching
- [Active Forgetting](active-forgetting.md) - Handles eviction of items the pre-loader no longer needs
