# State File Manager

The `StateFileManager` manages cognitive state as externalized files organized into three distinct layers. This three-layer architecture ensures that critical information - goals, progress, and accumulated wisdom - survives context window compactions, session restarts, and long-running multi-step tasks.

## How It Works

StateFileManager splits agent state into three dataclass-backed layers, each with different mutability rules:

- **Layer 1 - Crystallized (`CrystallizedState`)**: Immutable identity set once at initialization. Contains the goal, success criteria, constraints, user identity, and project context. Never changes after `initialize()` is called.

- **Layer 2 - Fluid (`FluidState`)**: Mutable working state updated throughout execution. Tracks current sub-goal, progress (0.0 to 1.0), sub-goals list, active entities, open questions, brain state digest, and step count.

- **Layer 3 - Insights (`InsightState`)**: Append-only accumulated wisdom. Stores the decision log, error journal (with open/resolved status), learned constraints, entity relationships, and pattern observations.

All three layers are persisted atomically to a single `state.json` file using a write-to-temp-then-rename strategy, ensuring no partial writes corrupt state. Files are organized by tenant and session: `{base_path}/{tenant_id}/{session_id}/state.json`.

## Key Features

- **Atomic persistence** - Uses temporary file + rename to prevent corruption during writes
- **Per-tenant/session isolation** - Each tenant and session gets its own state directory
- **Dirty-flag optimization** - Only writes to disk when state has actually changed
- **Token-aware context building** - `build_persistent_context()` generates a formatted string from all three layers, truncating to a configurable `max_tokens` limit (default 2000)
- **Error lifecycle tracking** - Errors are recorded with open/resolved status and can be resolved later via `resolve_error()`
- **Decision provenance** - Every decision is logged with step number, rationale, and timestamp
- **Deduplication** - Learned constraints, entity relationships, and pattern observations reject duplicates

## Integration

StateFileManager is a foundation component of the Cognitive Context Engine. The `CognitiveContextPipeline` uses it to inject persistent state into the context window at every step. The `build_persistent_context()` method produces a formatted markdown string that includes goals, constraints, progress, sub-goals, active entities, open questions, key decisions, unresolved errors, and learned constraints - all respecting a token budget.

The Orchestrator calls `initialize()` at session start, `update_fluid()` after each step, and the various `record_*` methods as decisions are made and errors encountered. The `load()` method restores state from disk when resuming a session.

## Usage Example

```python
from corteX.engine.cognitive import StateFileManager

# Create manager with tenant isolation
sfm = StateFileManager(
    base_path=".cortex_state",
    tenant_id="acme-corp",
    session_id="task-42",
)

# Initialize with goal (Layer 1 - set once)
await sfm.initialize(
    goal="Deploy the user service to production",
    constraints=["No downtime", "Must pass all tests"],
    success_criteria=["Service healthy on port 8080"],
)

# Update progress (Layer 2 - mutable)
await sfm.update_fluid(
    progress=0.5,
    current_sub_goal="Running integration tests",
    entities={"user-service": "building", "postgres": "ready"},
)

# Record decisions and errors (Layer 3 - append-only)
await sfm.record_decision(
    step=3, decision="Use blue-green deployment",
    rationale="Minimizes downtime risk",
)
await sfm.record_error(step=4, error="Test timeout on CI")
await sfm.resolve_error(step=4, resolution="Increased timeout to 60s")

# Build context string for LLM injection
context = sfm.build_persistent_context(max_tokens=1500)

# Resume a previous session
sfm2 = StateFileManager(
    base_path=".cortex_state",
    tenant_id="acme-corp",
    session_id="task-42",
)
restored = await sfm2.load()  # Returns True if state.json exists
```

## See Also

- [Context Quality Engine](quality-engine.md) - Measures quality of assembled context including decision preservation
- [Memory Crystallizer](crystallizer.md) - Extracts reusable patterns from the insights layer
- [Context Versioner](versioner.md) - Snapshots context state at each decision point
- [Cognitive Context Overview](../cognitive-context.md) - Architecture overview of the full pipeline
