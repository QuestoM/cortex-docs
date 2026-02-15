# Cognitive Context Pipeline API Reference

## Module: `corteX.engine.cognitive.cognitive_context`

8-phase unified context compilation pipeline. Phases: Score, Resolve, Entangle, Density, Assemble, Quality, Prefetch, Version. Replaces dual ContextCompiler + CorticalContextEngine. Includes adaptive zone budgets, quality gates, and causal version tracking.

## Classes

### `ScoredContextItem`

**Type**: `@dataclass`

Context item with computed priority scores.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `item_id` | `str` | Unique item identifier |
| `content` | `str` | Text content |
| `role` | `str` | Message role (`system`, `user`, `assistant`, `tool`) |
| `tokens` | `int` | Estimated token count |
| `raw_importance` | `float` | Base importance score |
| `goal_proximity` | `float` | Relevance to current goal (0.0-1.0) |
| `recency_factor` | `float` | Exponential decay by position |
| `priority` | `float` | Final priority: `0.35*goal + 0.30*recency + 0.35*importance` |
| `zone` | `str` | Target zone: `"system"`, `"persistent"`, `"working"`, `"recent"` |
| `metadata` | `Dict[str, Any]` | Additional metadata |

---

### `AssembledZone`

**Type**: `@dataclass`

A single zone in the assembled context.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Zone name |
| `messages` | `List[Dict[str, str]]` | Messages in this zone |
| `tokens` | `int` | Total tokens in zone |
| `budget` | `int` | Token budget for this zone |
| `utilization` | `float` | `tokens / budget` ratio |

---

### `CognitiveCompiledContext`

**Type**: `@dataclass`

Final output of the cognitive compilation pipeline.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `messages` | `List[Dict[str, str]]` | Assembled messages for the LLM |
| `total_tokens` | `int` | Total token count |
| `zone_usage` | `Dict[str, int]` | Tokens used per zone |
| `quality` | `ContextQualityReport` | Quality evaluation result |
| `context_version` | `str` | Version ID for this compilation |
| `phase_timings` | `Dict[str, float]` | Time (seconds) per phase |
| `items_scored` | `int` | Number of items scored in Phase 1 |
| `items_included` | `int` | Number of items included after filtering |
| `density_savings` | `int` | Tokens saved by density optimization |

---

### `ContextVersionSnapshot`

**Type**: `@dataclass`

Record of context state at a decision point.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `version_id` | `str` | Format: `v_{step}_{hash}` |
| `step_number` | `int` | Execution step |
| `context_hash` | `str` | SHA-256 content hash (16 chars) |
| `item_count` | `int` | Number of messages |
| `total_tokens` | `int` | Total tokens |
| `quality_snapshot` | `Dict[str, float]` | Quality scores at this point |
| `decision` | `str` | Decision made |
| `outcome` | `str` | Outcome text |
| `success` | `Optional[bool]` | Whether the outcome was successful |
| `timestamp` | `float` | Unix timestamp |

---

### `CognitiveContextPipeline`

Unified 8-phase cognitive context compilation pipeline.

#### Constructor

```python
CognitiveContextPipeline(
    max_context_tokens: int = 128_000,
    zone_budgets: Optional[Dict[str, float]] = None,
    quality_engine: Optional[ContextQualityEngine] = None,
    state_manager: Optional[StateFileManager] = None,
    enable_adaptive_zones: bool = True
)
```

**Parameters**:

- `max_context_tokens` (int, default=128,000): Maximum total tokens for assembled context.
- `zone_budgets` (`Optional[Dict[str, float]]`): Fractional budgets per zone. Defaults to:

| Zone | Budget Fraction | Purpose |
|------|----------------|---------|
| `system` | 0.12 | System prompt |
| `persistent` | 0.08 | Goal, brain state, externalized state |
| `working` | 0.40 | Working context items |
| `recent` | 0.40 | Recent conversation + goal reminder |

- `quality_engine` (`Optional[ContextQualityEngine]`): Custom quality engine. Auto-created if not provided.
- `state_manager` (`Optional[StateFileManager]`): State file manager for persistent context.
- `enable_adaptive_zones` (bool, default=True): Whether to adapt zone budgets dynamically.

#### Methods

##### `compile`

```python
async def compile(
    self, goal: str, system_prompt: str,
    messages: List[Dict[str, str]],
    brain_state: Optional[Dict[str, Any]] = None,
    step_number: int = 0,
    plan_state: Optional[Dict[str, Any]] = None
) -> CognitiveCompiledContext
```

Run the full 8-phase compilation pipeline.

**Parameters**:

- `goal` (`str`): Current task goal.
- `system_prompt` (`str`): System prompt text.
- `messages` (`List[Dict[str, str]]`): Input messages to compile.
- `brain_state` (`Optional[Dict[str, Any]]`): Brain state dict with confidence, momentum, focus, fatigue.
- `step_number` (`int`): Current execution step.
- `plan_state` (`Optional[Dict[str, Any]]`): Plan state for prefetch hints.

**Returns**: `CognitiveCompiledContext` with assembled messages, quality report, version ID, and phase timings.

**Pipeline phases**:

| Phase | Name | Description |
|-------|------|-------------|
| P1 | Score | Compute priority per message (goal proximity, recency, importance) |
| P2 | Resolve | Select items within token budget (priority-ordered) |
| P3 | Entangle | Boost co-referenced items (entity overlap detection) |
| P4 | Density | Apply domain abbreviations for token savings |
| P5 | Assemble | Build 4 zones (system, persistent, working, recent) |
| P6 | Quality | Evaluate all 6 quality dimensions |
| P7 | Prefetch | Log prefetch hints from plan state |
| P8 | Version | Create version snapshot with hash chain |

##### `record_outcome`

```python
async def record_outcome(
    self, step_number: int, decision: str,
    outcome: str, success: bool
) -> None
```

Record the outcome of a decision for version tracking and state file updates.

##### `diagnose_failure`

```python
def diagnose_failure(self, failure_step: int) -> Optional[Dict[str, Any]]
```

Auto-diagnose a failure by comparing against the most recent success.

**Returns**: `Optional[Dict[str, Any]]` with keys `success_version`, `failure_version`, `quality_delta`, `success_tokens`, `failure_tokens`, `success_items`, `failure_items`.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: Dict with keys `total_compiles`, `total_density_savings`, `versions_tracked`, `decisions_logged`, `quality_healthy`, `quality_history_count`.

---

## Priority Scoring Formula

```
priority = 0.35 * goal_proximity + 0.30 * recency_factor + 0.35 * importance
```

Where:
- `goal_proximity` = word overlap with goal, multiplied by 2, capped at 1.0
- `recency_factor` = `exp(-0.02 * distance_from_end)`
- `importance` = base importance, boosted for system (0.9) and tool (0.6) roles

---

## Example

```python
from corteX.engine.cognitive.cognitive_context import CognitiveContextPipeline

pipeline = CognitiveContextPipeline(max_context_tokens=128_000)

result = await pipeline.compile(
    goal="Build a REST API for user management",
    system_prompt="You are a helpful coding assistant.",
    messages=[
        {"role": "user", "content": "Create a User model with name and email"},
        {"role": "assistant", "content": "I will create the User model in models.py"},
        {"role": "tool", "content": "[Tool] Created models.py with User class"},
    ],
    brain_state={"confidence": 0.8, "momentum": 0.6},
    step_number=3,
)

print(f"Total tokens: {result.total_tokens}")
print(f"Quality: {result.quality.overall_health:.2f} ({result.quality.health_label})")
print(f"Zone usage: {result.zone_usage}")
print(f"Phase timings: {result.phase_timings}")
print(f"Density savings: {result.density_savings} tokens")

# Record outcome
await pipeline.record_outcome(
    step_number=3,
    decision="Create User model",
    outcome="Model created successfully",
    success=True,
)
```

---

## See Also

- [Context Quality Engine](./context-quality.md) -- Quality scoring (Phase 6)
- [State File Manager](./state-files.md) -- Persistent state injection
- [Density Optimizer](./density-optimizer.md) -- Token compression (Phase 4)
- [Entanglement Graph](./entanglement.md) -- Co-reference boosting (Phase 3)
- [Context Versioner](./versioner.md) -- Version tracking (Phase 8)
