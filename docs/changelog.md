# Changelog

All notable changes to corteX are documented here.

---

## v3.0.0-alpha.2

*Agentic engine gap fixes -- 6,200 tests passing.*

### Agentic engine wiring (6 fixes)

- **ContextCompiler in chat mode**: `session.run()` now uses the 4-zone context compiler (was agentic-only).
- **L2/L3 summarization execution**: Summarization prompts are now sent to the LLM and results stored (was generating prompts without executing them).
- **Sub-agent delegation**: The agentic loop now delegates work to `SubAgentManager` when the LLM requests task delegation.
- **Memory retrieval injection**: `MemoryFabric.get_relevant_context()` is called before each LLM call, injecting relevant memories into context.
- **Brain params consistency**: All 14 `generate()` call sites now pass the full 7-parameter brain state bundle.
- **Streaming with tools**: `run_stream()` now supports tool execution (up to 5 rounds).

### StreamChunk update

- Added `model` field (which model generated the chunk).
- Added `chunk_type` field (`text`, `tool_call`, `tool_result`, `error`).

---

## v3.0.0-alpha

*Initial release.*

### Brain components (20)

**P0 -- Core:**

- **WeightEngine** -- Bayesian conjugate priors with Thompson Sampling and prospect-theory loss aversion.
- **GoalTracker** -- Goal progress tracking with drift detection and loop prevention.
- **FeedbackEngine** -- 4-tier feedback processing (direct, user insights, enterprise, global).
- **PredictionEngine** -- Predictive coding with Bayesian surprise signals.
- **PlasticityManager** -- Hebbian, homeostatic, and metaplasticity learning rules.
- **MemoryFabric** -- Working, episodic, and semantic memory with pluggable backends.
- **DualProcessRouter** -- System 1/2 routing (fast vs slow path) based on surprise, novelty, and safety.
- **ReputationSystem** -- Game-theoretic tool trust scoring with automatic quarantine.
- **AdaptationFilter** -- Sensory adaptation with habituation and novelty amplification.
- **PopulationQualityEstimator** -- Multi-perspective response quality estimation.

**P1 -- Context and calibration:**

- **CorticalContextEngine** -- Hot/warm/cold context tiers supporting 10,000+ step workflows with auto-summarization and checkpointing.
- **ProactivePredictionEngine** -- Predict next user action and pre-warm resources.
- **CrossModalAssociator** -- Hebbian co-occurrence learning across conversation, tools, and memory modalities.
- **ContinuousCalibrationEngine** -- Platt scaling, confidence adjustment, and metacognition monitoring.

**P2 -- Specialization and attention:**

- **ColumnManager** -- Cortical column specialization with inter-column competition.
- **ResourceHomunculus** -- Non-uniform resource allocation by task type.
- **AttentionSystem** -- Subconscious processing, change detection, and attentional priority routing.

**P3 -- Advanced plasticity:**

- **ConceptGraphManager** -- Distributed concept representation with spreading activation.
- **CorticalMapReorganizer** -- Territory merging and redistribution for tools and models.
- **TargetedModulator** -- Optogenetic-inspired force-activation and silencing of tools/components.
- **ComponentSimulator** -- Digital twin for what-if analysis and A/B testing.

### SDK

- `Engine`, `Agent`, `Session` three-layer public API.
- `@cortex.tool` decorator with automatic JSON schema generation from type hints.
- `run()` with full 14-step brain pipeline.
- `run_stream()` for token-by-token streaming.
- `Response` with `ResponseMetadata` (goal progress, drift, tokens, latency, tools called).
- Session introspection: `get_weights()`, `get_goal_progress()`, `get_dual_process_stats()`, `get_reputation_stats()`, and 10 more.
- `simulate_what_if()` for digital twin what-if analysis.
- `activate_tool()` / `silence_tool()` for targeted modulation overrides.

### LLM providers

- OpenAI (GPT-4o, GPT-4o-mini, and compatible models).
- Google Gemini (Gemini 2.0 Flash, Gemini 2.5 Pro).
- Local models via OpenAI-compatible API (Ollama, vLLM).
- Multi-provider LLM Router with role-based routing and failover.

### Enterprise

- `EnterpriseConfig` with safety levels (strict, moderate, permissive).
- Blocked topics, audit logging, compliance rules.
- Per-session token budgets.
- Data retention controls (none, session, persistent).

### Infrastructure

- Python 3.11+ required.
- `pydantic`, `httpx`, `numpy` core dependencies.
- Optional extras: `[openai]`, `[gemini]`, `[server]`, `[browser]`, `[all]`.
- MkDocs Material documentation system.
