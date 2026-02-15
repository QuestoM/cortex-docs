# Cognitive Context Pipeline

The Cognitive Context Pipeline manages context quality, state persistence, and intelligent assembly across unlimited agent execution cycles. It replaces the dual ContextCompiler + CorticalContextEngine architecture with a unified 8-phase pipeline that ensures high-quality context even after thousands of compaction cycles.

## What It Does

The cognitive context system provides three core capabilities:

1. **State File Management**: Externalized 3-layer state that survives unlimited compactions
2. **Context Quality Scoring**: 6-dimensional quality metrics with automatic degradation detection
3. **Unified Context Pipeline**: 8-phase intelligent context assembly with prefetching and versioning

Together, these components ensure that agents maintain high-quality context and goal retention even in ultra-long workflows (10,000+ steps).

---

## StateFileManager

**corteX Innovation**: Externalized state persistence with atomic updates and tenant isolation.

The StateFileManager externalizes agent state into three JSON files that persist independently of conversation history compression:

| Layer | File | Purpose |
|-------|------|---------|
| **Crystallized** | `{tenant_id}/{session_id}_crystallized.json` | Facts, decisions, commitments that must survive forever |
| **Fluid** | `{tenant_id}/{session_id}_fluid.json` | Working state, current tasks, recent decisions |
| **Insights** | `{tenant_id}/{session_id}_insights.json` | Learned patterns, mistakes, optimization notes |

### Why: The Memory Persistence Problem

When agents compress conversation history to fit context windows, they risk losing critical state:

- User decisions from turn 47 that affect turn 1,247
- Mistakes identified at turn 203 that should never repeat
- Architectural decisions from the first 10 turns that constrain all future work

State files solve this by **externalizing critical information** outside the conversation history. Even after 100 compression cycles, the agent can reload its commitments, learnings, and working state.

### How It Works

```python
from corteX.engine.cognitive import StateFileManager, CrystallizedState, FluidState, InsightState

# Initialize manager
manager = StateFileManager(
    tenant_id="acme_corp",
    session_id="cs-2024-001",
    base_dir="C:\\agent_state"
)

# Initialize crystallized facts (immutable commitments)
await manager.initialize(
    goal="Build a REST API with JWT authentication",
    constraints=[
        "Never hardcode API keys",
        "Follow PEP-8 for Python modules"
    ],
    success_criteria=[
        "JWT authentication working",
        "PostgreSQL database connected"
    ],
    user_identity={
        "preference": "TypeScript for frontend",
        "style": "Detailed comments"
    },
    project_context="Building production SaaS application"
)

# Update fluid working state (changes frequently)
await manager.update_fluid(
    progress=0.4,
    current_sub_goal="Build login endpoint",
    sub_goals=[
        {"goal": "Set up FastAPI", "status": "complete"},
        {"goal": "Build login endpoint", "status": "active"}
    ],
    entities={
        "last_test_file": "tests/test_auth.py",
        "current_file": "api/auth.py"
    },
    questions=["Should we use RS256 or HS256 for JWT?"],
    step_count=47
)

# Save insights (learned patterns)
insights = InsightState(
    decision_log=[
        {"step": "15", "decision": "Added __init__.py", "rationale": "Package initialization required"}
    ],
    error_journal=[
        {"step": "42", "error": "Used mutable default argument", "resolution": "Use None + conditional initialization", "status": "resolved"}
    ],
    learned_constraints=[
        "Always create __init__.py in new packages",
        "Never use mutable default arguments"
    ]
)
await manager.add_learned_constraint("Always create __init__.py in new packages")

# Load state (e.g., after compression)
crystallized = manager.get_crystallized()
fluid = manager.get_fluid()
insights = manager.get_insights()
# Returns: CrystallizedState, FluidState, InsightState

# Check if state exists
if manager.exists("crystallized"):
    crystallized = manager.load_crystallized()
```

### Atomic Updates

All writes use atomic JSON updates with temp files:

1. Write to `{file}.tmp`
2. Flush to disk
3. Atomic rename to replace original

This prevents corruption if the process crashes mid-write.

### Tenant Isolation

State files are stored in tenant-specific directories:

```
C:\agent_state\
  acme_corp\
    cs-2024-001_crystallized.json
    cs-2024-001_fluid.json
    cs-2024-001_insights.json
  beta_inc\
    cs-2024-002_crystallized.json
    cs-2024-002_fluid.json
```

No tenant can access another tenant's state files.

---

## ContextQualityEngine

**corteX Innovation**: 6-dimensional context quality scoring with automatic degradation detection.

The ContextQualityEngine measures context quality across six dimensions and detects when compression has degraded quality below acceptable thresholds.

### Six Quality Dimensions

| Dimension | Weight | What It Measures |
|-----------|--------|------------------|
| **Goal Retention** | 30% | Is the original goal still present in context? |
| **Information Density** | 20% | How much useful information per token? |
| **Entanglement Completeness** | 15% | Are all critical cross-references preserved? |
| **Temporal Coherence** | 15% | Does the timeline make sense? |
| **Decision Preservation** | 10% | Are key decisions still documented? |
| **Anti-Hallucination** | 10% | Are there contradictions or invented facts? |

### How It Works

```python
from corteX.engine.cognitive import ContextQualityEngine

engine = ContextQualityEngine()

# Evaluate context quality
context_messages = [
    {"role": "user", "content": "Build a REST API with JWT auth"},
    {"role": "assistant", "content": "I'll build a FastAPI server..."},
    # ... 500 more turns ...
]

goal = "Build a REST API with JWT auth"
total_tokens = 125000
decision_log = [
    {"step": "3", "decision": "Use FastAPI", "rationale": "Modern async framework"}
]

quality_report = engine.evaluate(
    context_messages=context_messages,
    goal=goal,
    total_tokens=total_tokens,
    decision_log=decision_log,
    entangled_pairs_total=100,
    entangled_pairs_complete=85
)

print(f"Overall Quality: {quality_report.overall_health:.2%}")
# 0.847 (84.7%)

print(quality_report.dimension_scores)
# {
#   "grs": 0.95,
#   "idi": 0.82,
#   "ec": 0.78,
#   "tc": 0.91,
#   "dpr": 0.85,
#   "ahs": 0.73
# }

# Check if quality is acceptable
if quality_report.overall_health < 0.70:
    print("WARNING: Context quality degraded - consider checkpoint restore")
    weakest = engine.get_weakest_dimension()
    print(f"Weakest dimension: {weakest[0]} ({weakest[1]:.2f})")
    # "ahs (0.73)"
```

### Scoring Algorithm

The overall score uses a **weighted harmonic mean** to penalize any single dimension falling too low:

```
overall_score = 1 / Σ(weight_i / score_i)
```

This means:
- If anti-hallucination drops to 0.3, it pulls down the overall score significantly
- You can't compensate for a critical weakness by having high scores elsewhere
- All dimensions must stay above minimum thresholds

### When It Activates

- **Before every compression cycle**: Score quality to decide if compression is safe
- **After compression**: Score again to verify quality didn't degrade too much
- **Every 100 turns**: Background quality audit
- **On user request**: When the user asks "how is context quality?"

---

## CognitiveContextPipeline

**corteX Innovation**: Unified 8-phase context assembly pipeline with intelligent prefetching and versioning.

The CognitiveContextPipeline replaces the dual ContextCompiler + CorticalContextEngine with a single unified pipeline that handles scoring, resolution, entanglement, density optimization, assembly, quality checks, prefetching, and versioning.

### Eight Pipeline Phases

```text
Phase 1: Score Quality
   |
   v
Phase 2: Resolve References (goal, tools, state files)
   |
   v
Phase 3: Entangle Cross-Links (decisions → outcomes, tools → results)
   |
   v
Phase 4: Optimize Density (remove redundancy, compress repetition)
   |
   v
Phase 5: Assemble KV-Cache-Aware Context (4-zone allocation)
   |
   v
Phase 6: Quality Gate (verify assembled context meets thresholds)
   |
   v
Phase 7: Prefetch Next-Turn Resources (predictive loading)
   |
   v
Phase 8: Version Checkpoint (snapshot for rollback)
   |
   v
Compiled Context (ready for LLM)
```

### How It Works

```python
from corteX.engine.cognitive import CognitiveContextPipeline, ContextQualityEngine
from corteX.engine.cognitive import StateFileManager

# Initialize pipeline
state_manager = StateFileManager(tenant_id="acme", session_id="s1")
quality_engine = ContextQualityEngine()

pipeline = CognitiveContextPipeline(
    max_context_tokens=128000,
    quality_engine=quality_engine,
    state_manager=state_manager,
    enable_adaptive_zones=True
)

# Run full pipeline
result = await pipeline.compile(
    goal="Build REST API with JWT auth",
    system_prompt="You are a helpful assistant...",
    messages=conversation_history,
    brain_state={"confidence": 0.8, "momentum": 0.6},
    step_number=502
)

# Use compiled context in LLM call
llm_messages = result.messages
quality_score = result.quality.overall_health  # 0.847
phase_timings = result.phase_timings
# {
#   "score": 12,
#   "resolve": 8,
#   "entanglement": 15,
#   ...
# }

# Check zone usage
print(result.zone_usage)
# {"system": 15000, "persistent": 10000, "working": 50000, "recent": 53000}
```

### Phase Details

#### Phase 1: Score Quality

Runs `ContextQualityEngine.score_quality()` on the current context to establish a baseline.

#### Phase 2: Resolve References

Resolves three types of references:
- **Goal references**: "As we decided earlier..." → exact turn number and decision text
- **Tool references**: "Using the same tool..." → specific tool name and previous result
- **State references**: "The architecture we chose..." → crystallized state entry

#### Phase 3: Entangle Cross-Links

Creates explicit links between:
- Decisions and their downstream outcomes
- Tool calls and their results in later turns
- Promises made and promises fulfilled
- Errors detected and fixes applied

Example:
```
Turn 15: "I'll use PostgreSQL for the database"
  ↓ (entangled)
Turn 47: "Creating PostgreSQL migration script"
  ↓ (entangled)
Turn 203: "PostgreSQL connection pool configured"
```

#### Phase 4: Optimize Density

Removes redundancy:
- Collapse repetitive status updates: "Building... Building... Building..." → "Built (3 attempts)"
- Merge similar tool calls: `ls /foo`, `ls /foo` → `ls /foo (checked 2x)`
- Compress verbose outputs: 500-line stack trace → first 50 + last 50 lines

#### Phase 5: Assemble KV-Cache-Aware Context

Allocates context window into 4 zones (same as ContextCompiler):

| Zone | Allocation | Content |
|------|------------|---------|
| **System** | 12% | Brain state, safety rules, tool definitions |
| **Persistent** | 8% | Original goal, crystallized commitments, key decisions |
| **Working** | 40% | Current task state, recent decisions, active errors |
| **Recent** | 40% | Last N turns of conversation history |

#### Phase 6: Quality Gate

Re-scores the assembled context to verify quality didn't degrade during assembly:

```python
if result.quality_score < config.min_quality_score:
    raise ContextQualityError(f"Quality {result.quality_score:.2%} below threshold {config.min_quality_score:.2%}")
```

#### Phase 7: Prefetch Next-Turn Resources

Predicts what the user/agent will do next and preloads:
- Tool schemas that are likely to be needed
- Relevant memory chunks from long-term storage
- State file sections that will be referenced

This reduces latency for the next turn.

#### Phase 8: Version Checkpoint

Saves a snapshot of the assembled context for rollback:

```python
checkpoint = pipeline.save_checkpoint()
# Later, if something goes wrong:
pipeline.restore_checkpoint(checkpoint)
```

### Integration with Agent Loop

The pipeline is called automatically in the agent loop:

```python
# In Session.run() or Session.run_agentic()
pipeline_result = self.cognitive_pipeline.run(
    conversation_history=self.history,
    original_goal=self.goal_tracker.original_goal,
    state_files_manager=self.state_manager,
    turn_number=self.turn_number,
)

# Pass compiled context to LLM
response = self.llm_router.generate(
    messages=pipeline_result.compiled_context,
    # ... other params ...
)
```

---

## When It Activates

| Component | Activation Point |
|-----------|------------------|
| **StateFileManager** | Every 25 turns (save), after compression (save+load), on session start (load) |
| **ContextQualityEngine** | Before compression (score), after compression (verify), every 100 turns (audit) |
| **CognitiveContextPipeline** | Every LLM call in both `run()` and `run_agentic()` modes |

---

## Configuration

```python
from corteX.engine.cognitive import (
    StateFileManager,
    ContextQualityEngine,
    CognitiveContextPipeline,
    QualityThresholds
)

# State file configuration
state_manager = StateFileManager(
    base_path="/var/cortex/state",
    tenant_id="acme_corp",
    session_id="cs-001"
)

# Quality engine thresholds
thresholds = QualityThresholds(
    grs=0.3,  # Goal retention score minimum
    idi=0.4,  # Information density minimum
    ec=0.9,   # Entanglement completeness minimum
    tc=0.7,   # Temporal coherence minimum
    dpr=0.8,  # Decision preservation rate minimum
    ahs=0.5   # Anti-hallucination score minimum
)

quality_engine = ContextQualityEngine(thresholds=thresholds)

# Pipeline configuration
pipeline = CognitiveContextPipeline(
    max_context_tokens=128000,
    zone_budgets={"system": 0.12, "persistent": 0.08, "working": 0.40, "recent": 0.40},
    quality_engine=quality_engine,
    state_manager=state_manager,
    enable_adaptive_zones=True
)
```

---

## API Reference

```python
from corteX.engine.cognitive import (
    StateFileManager,
    CrystallizedState,
    FluidState,
    InsightState,
    ContextQualityEngine,
    ContextQualityReport,
    QualityThresholds,
    CognitiveContextPipeline,
    CognitiveCompiledContext,
    AssembledZone,
    ScoredContextItem,
    ContextVersionSnapshot,
)
```

See also:
- [Goal Intelligence](goal-intelligence.md) - GoalDNA, GoalReminderInjector, GoalTree
- [Anti-Drift](anti-drift.md) - LoopDetector, DriftEngine, AdaptiveBudget
- [Architecture Overview](architecture.md) - Full 14-step pipeline
