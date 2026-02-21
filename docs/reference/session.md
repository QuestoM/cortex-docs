# Session API Reference

## Module: `corteX.session.Session`

The `Session` is a **stateful** live conversation. It manages the full brain-inspired AI pipeline: synaptic weights, goal tracking, loop prevention, dual-process routing, context management, prediction/surprise, feedback learning, memory consolidation, and 20+ cognitive subsystems.

Sessions are created via `agent.start_session()`. Each session is fully independent -- concurrent sessions never share state.

## Class

### `Session`

A live conversation session with full brain integration.

The Session class is assembled from 20+ mixin modules via cooperative multiple inheritance. This page documents only the **public API** that SaaS developers use directly.

#### Constructor

```python
Session(
    agent: Agent,
    user_id: str,
    session_id: Optional[str] = None,
)
```

!!! note "Use `agent.start_session()` instead"
    Do not instantiate `Session` directly. Use `agent.start_session(user_id=...)` which passes the agent reference automatically.

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `agent` | `Agent` | (required) | The parent agent providing config, tools, and engine access. |
| `user_id` | `str` | (required) | User identifier for personalization and isolation. |
| `session_id` | `Optional[str]` | `None` | Custom session ID, or auto-generated as `sess_<hex>`. |

---

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `agent` | `Agent` | Reference to the parent agent. |
| `user_id` | `str` | The user this session belongs to. |
| `session_id` | `str` | Unique session identifier. |
| `weights` | `WeightEngine` | The synaptic weight engine (Bayesian posteriors + prospect theory). |
| `feedback` | `FeedbackEngine` | 4-tier feedback processing (direct, user insights, enterprise, global). |
| `prediction` | `PredictionEngine` | Prediction/surprise detection for learning. |
| `plasticity` | `PlasticityManager` | Controls learning rate adaptation. |
| `memory` | `MemoryFabric` | Unified memory across working, short-term, long-term, and episodic layers. |
| `context_engine` | `CorticalContextEngine` | Token-aware context management for 10,000+ step sessions. |
| `dual_process` | `DualProcessRouter` | System 1 (fast) / System 2 (slow) routing. |
| `reputation` | `ReputationSystem` | Tool trust scoring with quarantine for failing tools. |
| `calibration` | `ContinuousCalibrationEngine` | Continuous confidence calibration tracking. |
| `columns` | `ColumnManager` | Cortical column specialization and competition. |
| `attention` | `AttentionSystem` | Subconscious filtering and change detection. |
| `concepts` | `ConceptGraphManager` | Distributed concept representation and co-occurrence learning. |
| `audit` | `AuditLogger` | Enterprise audit trail logger. |
| `proactive` | `ProactivePredictionEngine` | Next-action prediction engine (P0). |
| `cross_modal` | `ContextEnricher` | Cross-modal association enricher (P1). |
| `resource_map` | `ResourceHomunculus` | Non-uniform resource allocation map (P2). |
| `reorganizer` | `CorticalMapReorganizer` | Cortical territory redistribution (P3). |
| `modulator` | `TargetedModulator` | Optogenetic-inspired activation/silencing (P3). |
| `simulator` | `ComponentSimulator` | Digital twin and what-if analysis (P3). |
| `adaptation` | `AdaptationFilter` | Sensory adaptation filtering. |
| `quality_estimator` | `PopulationQualityEstimator` | Multi-response quality estimation. |

---

## Execution Methods

### `run`

```python
async def run(
    self,
    message: str,
    attachments: Optional[List[Any]] = None,
) -> Response
```

Run the agent with a user message. This is the **primary execution method** that runs the full brain-integrated pipeline:

1. Feedback processing and sensory adaptation
2. Goal tracking initialization
3. Dual-process routing (System 1/2 decision)
4. Context engine management (CCE token budgets)
5. Brain state injection into LLM prompt
6. LLM generation (fast or slow path)
7. Tool execution with reputation tracking (up to 5 rounds)
8. Quality estimation and plasticity updates
9. Memory consolidation

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `message` | `str` | (required) | The user's message to process. |
| `attachments` | `Optional[List[Any]]` | `None` | Optional file attachments (reserved for future use). |

**Returns**: `Response` - The agent's response with content and metadata.

**Example**:

```python
session = agent.start_session(user_id="user_123")

response = await session.run("What's the status of my order #456?")
print(response.content)
print(f"Goal progress: {response.metadata.goal_progress:.0%}")
print(f"Model: {response.metadata.model_used}")
print(f"Tools called: {response.metadata.tools_called}")
```

---

### `run_stream`

```python
async def run_stream(
    self,
    message: str,
) -> AsyncIterator[StreamChunk]
```

Stream the agent's response with full tool execution support.

Streams text chunks from the LLM in real-time. If the LLM requests tool calls, executes them (yielding status chunks), then generates a follow-up response. Loops up to 5 tool rounds. After streaming completes, runs the full learning/feedback pipeline.

Uses the same brain-integrated preparation pipeline as `run()`.

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `message` | `str` | (required) | The user's message to process. |

**Returns**: `AsyncIterator[StreamChunk]` - An async iterator yielding `StreamChunk` objects.

**StreamChunk Attributes**:

| Attribute | Type | Description |
|-----------|------|-------------|
| `content` | `str` | Text content of this chunk. |
| `finish_reason` | `Optional[str]` | `"stop"`, `"error"`, `"policy_block"`, or `None` while streaming. |
| `model` | `Optional[str]` | Model name (available on final chunk). |
| `chunk_type` | `str` | `"content"`, `"tool_execution"`, or `"tool_result"`. |
| `tool_call_delta` | `Optional[Dict]` | Tool call information when `chunk_type` is tool-related. |

**Example**:

```python
session = agent.start_session(user_id="user_123")

async for chunk in session.run_stream("Explain quantum computing"):
    if chunk.chunk_type == "content" and chunk.content:
        print(chunk.content, end="", flush=True)
    elif chunk.chunk_type == "tool_execution":
        print(f"\n[Executing tools: {chunk.tool_call_delta['tools']}]")
    elif chunk.chunk_type == "tool_result":
        print(f"\n[Tool {chunk.tool_call_delta['tool_name']}: "
              f"{'OK' if chunk.tool_call_delta['success'] else 'Failed'}]")
print()  # Final newline
```

---

### `run_agentic`

```python
async def run_agentic(
    self,
    goal: str,
    max_steps: int = 100,
    interaction_callback: Optional[Callable] = None,
) -> Dict[str, Any]
```

Run a goal-driven, multi-step agent loop. The agent plans, executes, reflects, and iterates until the goal is achieved or the step budget is exhausted.

Uses AgentLoop (if available) or falls back to a planning-based loop with PlanningEngine + ReflectionEngine. Supports sub-agent delegation for complex subtasks.

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `goal` | `str` | (required) | The high-level goal to achieve (e.g., `"Build a REST API for user management"`). |
| `max_steps` | `int` | `100` | Maximum number of loop iterations before stopping. |
| `interaction_callback` | `Optional[Callable]` | `None` | Async callback for user interactions during execution (e.g., asking for clarification). |

**Returns**: `Dict[str, Any]` - A result dictionary with keys:

| Key | Type | Description |
|-----|------|-------------|
| `"goal"` | `str` | The original goal. |
| `"steps_taken"` | `int` | Number of loop iterations executed. |
| `"final_response"` | `str` | The agent's final output. |
| `"stats"` | `Dict` | Execution statistics (latency, tokens, tools used). |

**Example**:

```python
session = agent.start_session(user_id="dev_42")

result = await session.run_agentic(
    goal="Research the top 3 Python web frameworks and create a comparison table",
    max_steps=20,
)

print(f"Completed in {result['steps_taken']} steps")
print(result["final_response"])
```

---

## Inspection Methods

### `get_goal_progress`

```python
def get_goal_progress(self) -> Dict[str, Any]
```

Get current goal tracking state including progress percentage and drift score.

**Returns**: `Dict` with keys `"progress"` (float, 0.0-1.0) and `"drift"` (float, 0.0-1.0).

### `get_weights`

```python
def get_weights(self) -> Dict[str, Any]
```

Get a snapshot of the current behavioral weights (verbosity, formality, autonomy, etc.).

**Returns**: `Dict[str, Any]` - Full weight state including behavioral, tool, and model weights.

### `override_weight`

```python
def override_weight(self, key: str, value: float) -> None
```

Manually override a behavioral weight. This is a direct user override that takes effect immediately on the next turn.

**Parameters**:

- `key` (str): Weight name (e.g., `"verbosity"`, `"formality"`, `"autonomy"`).
- `value` (float): New weight value, typically in [-1.0, 1.0].

**Example**:

```python
# Make the agent more verbose mid-conversation
session.override_weight("verbosity", 0.8)
# Make it more formal
session.override_weight("formality", 0.5)
```

### `get_context_stats`

```python
def get_context_stats(self) -> Dict[str, Any]
```

Get Cortical Context Engine statistics (token usage, hot/warm/cold distribution, items tracked).

### `get_dual_process_stats`

```python
def get_dual_process_stats(self) -> Dict[str, Any]
```

Get System 1 (fast) / System 2 (slow) routing decision statistics.

### `get_reputation_stats`

```python
def get_reputation_stats(self) -> Dict[str, Any]
```

Get tool reputation and trust scores including quarantine status.

### `get_calibration_report`

```python
def get_calibration_report(self) -> Dict[str, Any]
```

Get calibration health report with ECE (Expected Calibration Error) scores by domain.

### `get_column_stats`

```python
def get_column_stats(self) -> Dict[str, Any]
```

Get functional column specialization statistics.

### `get_concept_graph_stats`

```python
def get_concept_graph_stats(self) -> Dict[str, Any]
```

Get concept graph statistics including node count and co-occurrence patterns.

### `get_concept_recommendations`

```python
def get_concept_recommendations(self, items: List[str]) -> Dict[str, Any]
```

Get concept-based recommendations for given items based on learned co-occurrence patterns.

### `get_proactive_stats`

```python
def get_proactive_stats(self) -> Dict[str, Any]
```

Get proactive prediction statistics (P0). Returns metrics on next-action predictions including hit rate and prediction counts.

### `get_cross_modal_stats`

```python
def get_cross_modal_stats(self) -> Dict[str, Any]
```

Get cross-modal association statistics (P1). Returns metrics on context enrichment from cross-modal associations.

### `get_resource_map`

```python
def get_resource_map(self) -> Dict[str, Any]
```

Get resource allocation map statistics (P2). Returns the current non-uniform resource allocation across domains.

### `get_attention_stats`

```python
def get_attention_stats(self) -> Dict[str, Any]
```

Get attentional filter statistics (P2). Returns metrics on subconscious filtering and change detection.

### `get_territory_map`

```python
def get_territory_map(self) -> Dict[str, Any]
```

Get cortical map reorganizer territory map (P3). Returns the current territory distribution across specialized regions.

### `get_active_modulations`

```python
def get_active_modulations(self) -> List[Dict[str, Any]]
```

Get all active modulations (P3). Returns a list of currently active tool activation/silencing modulations.

### `get_decision_log`

```python
def get_decision_log(self) -> Any
```

Get the decision log for audit and compliance. Returns the `DecisionLog` instance containing all recorded decisions from the session.

### `get_goal_tree`

```python
def get_goal_tree(self) -> Any
```

Get the goal tree for hierarchical progress tracking. Returns the `GoalTree` instance showing goal decomposition and sub-goal status.

### `get_nash_routing_stats`

```python
def get_nash_routing_stats(self) -> Dict[str, Any]
```

Get Nash routing optimizer statistics. Returns game-theoretic routing metrics including equilibrium convergence data.

### `get_shapley_stats`

```python
def get_shapley_stats(self) -> Dict[str, Any]
```

Get Shapley attribution statistics. Returns fair contribution attribution across tools and components.

### `get_summarization_stats`

```python
def get_summarization_stats(self) -> Dict[str, Any]
```

Get L2/L3 summarization pipeline statistics. Returns metrics on context summarization including compression ratios and summary counts.

### `get_simulator`

```python
def get_simulator(self) -> ComponentSimulator
```

Get the component simulator for what-if analysis (P3). Returns the `ComponentSimulator` instance which supports forking state, running simulations, and session recording/replay.

### `get_progress_estimate`

```python
def get_progress_estimate(self) -> Dict[str, Any]
```

Get the current progress estimate including estimated steps remaining and velocity.

**Returns**: Dict with keys `"steps_remaining"`, `"velocity"`, `"status"`, `"trend"`, `"current_progress"`.

---

## Modulation Methods

### `activate_tool`

```python
def activate_tool(
    self,
    tool_name: str,
    turns: int = 5,
    reason: str = "",
) -> str
```

Force-activate a tool for N turns via targeted modulation. The agent will prefer using this tool during the activation period.

**Returns**: `str` - Modulation ID for tracking.

### `silence_tool`

```python
def silence_tool(
    self,
    tool_name: str,
    turns: int = 5,
    reason: str = "",
) -> str
```

Force-silence a tool for N turns. The agent will avoid using this tool during the silence period.

**Returns**: `str` - Modulation ID for tracking.

### `simulate_what_if`

```python
def simulate_what_if(self, change: Dict[str, Any]) -> Dict[str, Any]
```

Run a what-if simulation against the current session state. Fork the weight state, apply the proposed change, and predict the impact without affecting the live session.

**Example**:

```python
result = session.simulate_what_if({
    "weight_overrides": {"verbosity": 0.9},
})
print(f"Predicted impact: {result}")
```

---

## GDPR Property

### `gdpr`

```python
@property
def gdpr(self) -> GDPRManager
```

Lazy-initialized GDPR DSAR (Data Subject Access Request) manager. Provides access to GDPR Articles 15-22 operations: export, rectify, erase, restrict, port, object, and transparency.

The `GDPRManager` is only instantiated on first access, keeping session initialization lightweight when GDPR features are not needed.

**Returns**: `GDPRManager` - The GDPR operations manager bound to this session's memory, weights, and audit logger.

**Example**:

```python
# Export all personal data (Article 15)
data = session.gdpr.export(user_id="user_123")

# Erase personal data (Article 17 - Right to Erasure)
result = session.gdpr.erase(user_id="user_123")

# Data portability (Article 20)
portable = session.gdpr.port(user_id="user_123", format="json")
```

---

## Lifecycle Methods

### `close`

```python
async def close(self) -> Dict[str, Any]
```

Close the session and perform final consolidation:

1. Stop session recording (if active)
2. Consolidate learned weights
3. Consolidate important memories
4. Persist state files
5. Collect comprehensive statistics from all 30+ brain components

**Returns**: `Dict[str, Any]` - Comprehensive session statistics with 40+ keys from all brain components:

| Key | Type | Description |
|-----|------|-------------|
| `"session_id"` | `str` | Session identifier. |
| `"turns"` | `int` | Total conversation turns. |
| `"total_tokens"` | `int` | Total tokens consumed. |
| `"duration_seconds"` | `float` | Session wall-clock duration. |
| `"final_weights"` | `Dict` | Final weight state snapshot. |
| `"memory_stats"` | `Dict` | Memory layer statistics. |
| `"context_stats"` | `Dict` | Context engine statistics. |
| `"dual_process_stats"` | `Dict` | System 1/2 routing statistics. |
| `"reputation_stats"` | `Dict` | Tool reputation and trust scores. |
| `"proactive_stats"` | `Dict` | Proactive prediction statistics. |
| `"cross_modal_stats"` | `Dict` | Cross-modal association statistics. |
| `"calibration_stats"` | `Dict` | Calibration health report. |
| `"column_stats"` | `Dict` | Functional column statistics. |
| `"resource_map_stats"` | `Dict` | Resource allocation statistics. |
| `"attention_stats"` | `Dict` | Attentional filter statistics. |
| `"concept_graph_stats"` | `Dict` | Concept graph statistics. |
| `"reorganizer_stats"` | `Dict` | Cortical map reorganizer statistics. |
| `"modulator_stats"` | `Dict` | Targeted modulator statistics. |
| `"simulator_stats"` | `Dict` | Component simulator statistics. |
| `"feedback_stats"` | `Dict` | Feedback signal summary. |
| `"goal_tracker_stats"` | `Dict` | Goal tracking summary. |
| `"plasticity_stats"` | `Dict` | Plasticity manager statistics. |
| `"prediction_stats"` | `Dict` | Prediction engine statistics. |
| `"adaptation_stats"` | `Dict` | Adaptation filter statistics. |
| `"quality_estimator_stats"` | `Dict` | Quality estimator last agreement. |
| `"param_resolver_stats"` | `Dict` | Brain parameter resolver statistics. |
| `"nash_routing_stats"` | `Dict` | Nash routing optimizer statistics. |
| `"shapley_attribution_stats"` | `Dict` | Shapley attribution statistics. |
| `"summarization_stats"` | `Dict` | L2/L3 summarization pipeline statistics. |
| `"semantic_scorer_stats"` | `Dict` | Semantic importance scorer statistics. |
| `"context_compiler_stats"` | `Dict` | Context compiler statistics. |
| `"planner_stats"` | `Dict` | Planning engine availability. |
| `"reflector_stats"` | `Dict` | Reflection engine statistics. |
| `"recovery_stats"` | `Dict` | Recovery engine statistics. |
| `"interaction_stats"` | `Dict` | Interaction manager statistics. |
| `"policy_stats"` | `Dict` | Policy engine statistics. |
| `"sub_agent_stats"` | `Dict` | Sub-agent manager statistics. |
| `"decision_log_stats"` | `Dict` | Decision log pattern analysis. |
| `"progress_estimator_stats"` | `Dict` | Progress estimator statistics. |
| `"speculative_executor_stats"` | `Dict` | Speculative executor statistics. |
| `"model_mosaic_stats"` | `Dict` | Model mosaic statistics. |
| `"provider_health_stats"` | `Dict` | Provider health monitor statistics. |
| `"decision_tracer_stats"` | `Dict` | Decision tracer statistics. |
| `"goal_tree_stats"` | `Dict` | Goal tree summary. |
| `"active_forgetting_stats"` | `Dict` | Active forgetting statistics. |
| `"tenant_dna_stats"` | `Dict` | Tenant DNA profile statistics. |
| `"quota_tracker_stats"` | `Dict` | Quota usage statistics. |
| `"audit_stream_stats"` | `Dict` | Audit stream statistics. |
| `"metrics_collector_stats"` | `Dict` | Metrics collector statistics. |

**Example**:

```python
stats = await session.close()
print(f"Session {stats['session_id']} completed:")
print(f"  Turns: {stats['turns']}")
print(f"  Tokens: {stats['total_tokens']}")
print(f"  Duration: {stats['duration_seconds']:.1f}s")
```

---

## Brain Components Initialized at Session Start

When `start_session()` is called, the Session initializes the full brain pipeline:

| Component | Class | Purpose |
|-----------|-------|---------|
| Weights | `WeightEngine` | Bayesian posteriors with prospect theory |
| Feedback | `FeedbackEngine` | 4-tier feedback processing |
| Prediction | `PredictionEngine` | Surprise detection for adaptive learning |
| Plasticity | `PlasticityManager` | Learning rate adaptation |
| Memory | `MemoryFabric` | Unified multi-layer memory |
| Context | `CorticalContextEngine` | Token-aware 10K+ step context |
| Dual Process | `DualProcessRouter` | System 1/2 fast-slow routing |
| Reputation | `ReputationSystem` | Tool trust with quarantine |
| Calibration | `ContinuousCalibrationEngine` | Confidence calibration tracking |
| Columns | `ColumnManager` | Cortical column specialization |
| Attention | `AttentionSystem` | Subconscious filtering |
| Concepts | `ConceptGraphManager` | Concept co-occurrence learning |
| Proactive | `ProactivePredictionEngine` | Next-action prediction |
| Cross-Modal | `ContextEnricher` | Cross-modal association |
| Resource Map | `ResourceHomunculus` | Non-uniform resource allocation |
| Reorganizer | `CorticalMapReorganizer` | Territory redistribution |
| Modulator | `TargetedModulator` | Optogenetic-inspired activation/silencing |
| Simulator | `ComponentSimulator` | Digital twin and what-if analysis |

All components degrade gracefully -- if any initialization fails, the session continues without that component.

---

## See Also

- [Engine API Reference](./engine.md) - Engine and provider setup
- [Agent API Reference](./agent.md) - Agent configuration
- [Response API Reference](./response.md) - Response and metadata types
- [Streaming Guide](../getting-started/streaming.md) - Streaming responses tutorial
- [Brain-LLM Bridge Concept](../concepts/brain/brain-llm-bridge.md) - How brain state reaches the LLM
