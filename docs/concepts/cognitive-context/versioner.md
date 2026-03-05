# Context Versioner

The `ContextVersioner` records the state of assembled context at each decision point and provides causal diff analysis when failures occur. By maintaining a versioned history of what information was available when each decision was made, it enables the agent to diagnose exactly what context was missing or corrupted when something goes wrong.

## How It Works

### Version Recording

At each decision point, `record()` captures a `ContextVersion` snapshot containing:

- A version ID derived from the step number and a SHA-256 hash of the context content (truncated to 16 chars)
- The list of item IDs present in the assembled context
- Quality scores at the time of the decision (from the `ContextQualityEngine`)
- The decision text, total token count, and timestamp

Versions are indexed by both version ID and step number for efficient lookup. A capacity limit (default 500 versions) automatically evicts the oldest versions in FIFO order.

### Outcome Recording

After a decision plays out, `record_outcome()` annotates the version with the result and a success/failure flag. This enables retrospective analysis of which context compositions led to good vs bad outcomes.

### Causal Diff

The core diagnostic tool is `causal_diff()`, which compares two versions (typically a success and a failure) and produces a `CausalDiff` report:

- **Missing items** - Item IDs present during success but absent during failure
- **Extra items** - Item IDs present during failure but absent during success
- **Quality delta** - Per-dimension quality score differences (positive means success had higher quality)
- **Suggested boosts** - Items from the missing list that should be prioritized for recovery
- **Diagnosis** - Human-readable summary of the likely cause

### Auto-Diagnosis

The `diagnose_failure()` method automates the process: given a failure step, it searches backward through version history for the most recent success and automatically diffs against it.

## Key Features

- **Content-addressable versions** - Each version has a unique hash for integrity
- **Step-indexed lookup** via `get_version()` for direct access by step number
- **Automatic failure diagnosis** via `diagnose_failure()` - Finds nearest prior success and diffs
- **Quality trend tracking** via `get_quality_trend()` - Returns quality scores, success flags, and token counts over recent versions
- **Success rate computation** via `get_success_rate()` over a configurable window
- **FIFO capacity management** - Oldest versions are evicted when exceeding `max_versions`
- **Full version history** via `get_history()` for inspection and debugging

## Integration

The `ContextVersioner` is called by the `CognitiveContextPipeline` after each context compilation. The pipeline records a version with the assembled context messages and quality scores, then later annotates it with the outcome. When the Orchestrator detects a failure, it calls `diagnose_failure()` to determine what context items to boost or restore.

The versioner's quality trend data feeds back into the `ContextQualityEngine`'s degradation velocity computation, creating a feedback loop that detects sustained quality decline. The `CausalDiff.suggested_boosts` list is used by the pipeline to prioritize items in the next compilation cycle.

## Usage Example

```python
from corteX.engine.cognitive import ContextVersioner

versioner = ContextVersioner(max_versions=500)

# Record context at each decision point
version = versioner.record(
    step_number=10,
    assembled_context=[
        {"role": "system", "content": "Deploy agent", "_item_id": "sys_prompt"},
        {"role": "tool", "content": "Tests: 142/142 passed", "_item_id": "test_results"},
        {"role": "tool", "content": "Config: port=8080", "_item_id": "config"},
    ],
    quality={"grs": 0.9, "dpr": 0.85, "overall": 0.82},
    decision="Proceed with deployment",
)

# Record outcome
versioner.record_outcome(step_number=10, outcome="Deployed OK", success=True)

# Record a failure at a later step
versioner.record(
    step_number=15,
    assembled_context=[
        {"role": "system", "content": "Deploy agent", "_item_id": "sys_prompt"},
        # Note: test_results and config are missing
    ],
    quality={"grs": 0.5, "dpr": 0.3, "overall": 0.4},
    decision="Rollback deployment",
)
versioner.record_outcome(step_number=15, outcome="Rollback failed", success=False)

# Diagnose the failure automatically
diff = versioner.diagnose_failure(failure_step=15)
if diff:
    print(diff.missing_items)    # ["test_results", "config"]
    print(diff.quality_delta)    # {"grs": 0.4, "dpr": 0.55, "overall": 0.42}
    print(diff.suggested_boosts) # ["test_results", "config"]
    print(diff.diagnosis)
    # "2 items present in success but missing at failure.
    #  Goal retention was higher during success.
    #  Decision preservation was higher during success.
    #  Overall quality delta: 0.42."

# Track trends
trend = versioner.get_quality_trend(last_n=20)
success_rate = versioner.get_success_rate(last_n=20)  # 0.5

# Statistics
stats = versioner.get_stats()
# {"total_versions": 2, "with_outcome": 2, "successes": 1,
#  "success_rate": 0.5, "max_versions": 500}
```

## See Also

- [Context Quality Engine](quality-engine.md) - Provides the quality scores recorded in each version
- [State File Manager](state-file-manager.md) - Decision log provides the decisions annotated in versions
- [Active Forgetting](active-forgetting.md) - Causal diffs reveal items lost to aggressive forgetting
- [Entanglement Graph](entanglement.md) - Missing partners in diffs may indicate broken entanglement
