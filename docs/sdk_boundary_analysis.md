# corteX SDK Boundary Analysis

## Built-In vs. SDK-Configurable: Defining the Developer Surface

**Date:** February 2026
**Status:** Architecture Decision Record
**Scope:** All 14 corteX modules -- what developers see, what's hidden, and where the configuration boundary lies

---

## 1. Competitor Boundary Analysis

### 1.1 OpenAI Agents SDK

**Philosophy:** "Minimalism with power -- fewer abstractions, faster learning."

| Aspect | Built-In (Zero Config) | Configurable by Developer |
|--------|----------------------|--------------------------|
| **Core Primitives** | Agent, Handoff, Guardrail | All are developer-defined instances |
| **Tracing** | Automatic span collection for LLM calls, tool calls, handoffs | Custom spans, external exporters (Logfire, AgentOps, Braintrust) |
| **Session Memory** | Automatic conversation history persistence | Memory strategy selection |
| **Guardrails** | Parallel execution model, fail-fast behavior | Guard logic itself (input/output validators) |
| **Model Selection** | Default to OpenAI models | Provider-agnostic: 100+ LLMs supported |
| **Tool Framework** | Python function wrapping with type inference | Developer writes the tool functions |
| **Multi-Agent** | Handoff protocol between agents | Topology, delegation rules, agent definitions |

**Key Insight:** OpenAI's SDK succeeds by being opinionated about *infrastructure* (tracing always on, guardrails run in parallel) while being flexible about *business logic* (what the agents do, what tools exist, what guardrails check). The developer never configures how tracing works -- only where traces go.

**What corteX should learn:** Tracing, safety execution patterns, and lifecycle management should be invisible infrastructure. Business logic surfaces (tools, prompts, guardrails) should be developer-owned.

### 1.2 LangChain / LangGraph

**Philosophy:** "Abstraction for everything" (LangChain) evolving toward "Low-level control" (LangGraph).

| Aspect | Built-In | Configurable | Extension Ecosystem |
|--------|----------|-------------|-------------------|
| **Chains/Graphs** | Execution engine | Node/edge definitions | 100+ pre-built chains |
| **Memory** | Multiple memory types | Which memory, capacity, backend | Community memory integrations |
| **LLM Providers** | Routing abstractions | Provider configs, model selection | 100+ provider adapters |
| **Retrieval** | RAG pipeline primitives | Embedding models, vector stores, chunking | Dozens of connector packages |
| **Evaluation** | LangSmith platform (paid) | Eval criteria, datasets | Custom evaluators |
| **Output Parsers** | Built-in JSON, Pydantic parsers | Schema definitions | Custom parser classes |

**Key Failure Pattern:** LangChain's original sin was abstracting at the wrong level. Developers complained about "the same feature done three different ways," deeply nested class hierarchies that made debugging impossible, and dependency bloat that pulled dozens of transitive dependencies into every project. The GitHub discussion thread "Is LangChain becoming too complex/bloated?" captures the community sentiment: the framework tries to provide an abstraction for everything, but this one-size-fits-all approach makes simple things complex.

**LangGraph Correction:** LangGraph represents LangChain's acknowledgment of this failure. It drops to a lower level: nodes and edges are just Python functions, usable with or without LangChain. This is the market signaling that developers want *less* abstraction, not more.

**What corteX should learn:** Never abstract business logic the developer understands better than the framework. Abstract infrastructure the developer should never need to think about. The boundary between the two is the critical design decision.

### 1.3 CrewAI

**Philosophy:** "Role-based teams that feel like hiring a crew."

| Aspect | Built-In (Opinionated) | Configurable | Not Possible |
|--------|----------------------|-------------|-------------|
| **Agent Model** | Role + Goal + Backstory | All three fields | Non-role-based agents |
| **Task Model** | Description + Expected Output | Task parameters | Non-sequential custom flows (until Flows) |
| **Process** | Sequential, Hierarchical | Process type selection | Custom process topologies (until recently) |
| **Memory** | Basic conversation | Memory type | Custom memory backends |
| **LLM** | OpenAI-centric | Model override per agent | Full provider-agnostic routing |
| **Delegation** | Built-in delegation protocol | Delegation rules | Custom delegation logic |

**Key Failure Pattern:** CrewAI is criticized for being too rigid. A Reddit user noted it "lacks flexibility" when you try anything outside its intended patterns. The role+goal+backstory triplet is elegant for demos but constraining for production systems that need dynamic agent composition.

**Key Success Pattern:** CrewAI's simplicity is also its strength for onboarding. Defining an agent in three lines (role, goal, backstory) is a "pit of success" -- the developer cannot create a malformed agent. The framework forces good practices.

**What corteX should learn:** Provide a simple "happy path" that is hard to misuse (like CrewAI), but do not close the door on advanced customization (unlike CrewAI). The `cortex.Engine` -> `create_agent` -> `start_session` -> `session.run()` chain achieves this.

### 1.4 Anthropic Claude SDK + MCP

**Philosophy:** "The SDK handles the model; MCP handles the world."

| Aspect | Built-In (SDK) | MCP Protocol (Extensible) | Developer Responsibility |
|--------|---------------|--------------------------|------------------------|
| **LLM Interface** | Claude API, streaming, tool use | N/A | Prompt engineering |
| **Built-In Tools** | Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch | MCP tool servers | Custom tool logic |
| **Agent Skills** | PowerPoint, Excel, Word, PDF processing | MCP Apps (interactive UI) | Domain-specific skills |
| **Memory** | Session-based context management | External memory via MCP | Long-term storage strategy |
| **Safety** | Constitutional AI, content filtering | N/A | Application-level guardrails |
| **Observability** | Basic logging | External via MCP integration | Monitoring infrastructure |

**Key Architectural Insight:** Anthropic drew the cleanest boundary in the industry. The SDK owns the *model interaction* (how you talk to Claude). MCP owns *world interaction* (how Claude talks to everything else). This separation means the SDK stays small while MCP enables infinite extensibility through a standard protocol.

**What corteX should learn:** The SDK should own the brain (weights, plasticity, prediction, feedback, adaptation) as opaque infrastructure. The protocol layer (tools, memory backends, LLM providers) should follow the MCP model of standard interfaces with pluggable implementations.

### 1.5 Google ADK (Agent Development Kit)

**Philosophy:** "Code-first with Gemini optimization."

| Aspect | Built-In | Developer-Extended |
|--------|----------|--------------------|
| **Agent Types** | LlmAgent, SequentialAgent, ParallelAgent, LoopAgent | Custom agents via BaseAgent |
| **Tools** | Search, Code Execution | Python functions with type hints |
| **Orchestration** | Workflow agents for predictable pipelines | LLM-driven dynamic routing |
| **Evaluation** | Built-in eval framework (response quality + trajectory) | Custom test cases |
| **Streaming** | Bidirectional audio/video | Custom stream handlers |
| **Debugging** | CLI + visual Web UI for step-by-step inspection | Custom event handlers |

**Key Insight:** ADK provides the strongest built-in evaluation story in the market. Systematic assessment of both "final response quality" and "step-by-step execution trajectory" against predefined test cases is a first-class primitive, not an afterthought.

**What corteX should learn:** Evaluation and debugging tools should be built-in, not bolted on. The GoalTracker, PredictionEngine comparison metrics, and PopulationCoding quality estimates already provide the raw data -- corteX needs to surface this as a developer-facing eval framework.

---

## 2. Developer Experience Principles

Based on competitor analysis and the Stack Overflow 2025 Developer Survey data (66% of developers frustrated by "AI solutions that are almost right, but not quite"), the following principles should govern corteX's boundary decisions.

### 2.1 The "Pit of Success" Principle

> Make the right thing easy and the wrong thing hard.

**Applied to corteX:**
- `cortex.Engine(providers={"openai": {"api_key": "..."}})` should immediately give you a working adaptive agent with zero brain configuration.
- Misconfiguring the brain should be *impossible* through the public API. You can tune it, but you cannot break it.
- The default configuration should be production-viable, not just demo-viable.

### 2.2 The "Infrastructure vs. Business Logic" Boundary

> Infrastructure is what the framework knows better than the developer.
> Business logic is what the developer knows better than the framework.

| Infrastructure (Framework Owns) | Business Logic (Developer Owns) |
|-------------------------------|--------------------------------|
| Weight system dynamics | What the agent does |
| Plasticity rules | System prompts |
| Feedback signal detection | Tool implementations |
| Prediction calibration | Domain-specific guardrails |
| Memory consolidation | When to use which agent |
| Adaptation filtering | Enterprise business rules |
| Population coding aggregation | Data models and schemas |
| Loop/drift detection | User-facing UX |

### 2.3 The "Inspect, Don't Build" Principle

> Developers should be able to inspect internal state but should never need to build it.

The Weight Engine, Plasticity Manager, and Prediction Engine operate autonomously. Developers should see their output (via `response.metadata` and `session.get_weights()`) but should not need to construct weight update rules or plasticity functions. The analogy: you can read your car's dashboard, but you do not rebuild the engine.

### 2.4 The "Progressive Disclosure" Principle

> Level 1: It works with zero config.
> Level 2: I can tune the knobs.
> Level 3: I can swap implementations.
> Level 4: I can extend the framework.

Each corteX module should support all four levels, but *most developers should never go beyond Level 2*.

### 2.5 The "Enterprise Layer" Principle

> Enterprise configuration constrains; it does not implement.

Enterprise config (safety policies, model policies, tool policies, audit) acts as a *ceiling* on what agents can do, never as a *floor*. An enterprise admin blocks topics, caps autonomy, requires approval -- but never implements agent logic. This keeps the enterprise layer orthogonal to the brain engine.

### 2.6 Anti-Pattern: The "Configuration Treadmill"

The Stack Overflow survey reveals that developers increasingly feel that "instead of building, people spend more time configuring the act of building." The response from the Kubernetes ecosystem was not to make every developer an expert but to introduce platform teams and abstractions that absorbed complexity.

**Applied to corteX:** The brain engine IS the platform that absorbs complexity. SaaS developers should not become neuroscience experts to use corteX. The framework does the thinking about thinking.

---

## 3. Module-by-Module Boundary Recommendations

### Category Definitions

| Category | Description | Developer Interaction | Example |
|----------|-------------|----------------------|---------|
| **A: Fully Built-In** | Zero config, invisible, always active | None -- developer does not know it exists | Loop detection |
| **B: Built-In with Defaults** | Works out-of-the-box, tunable knobs | Optional parameter tweaking | Weight learning rates |
| **C: SDK Framework** | SDK provides interface, developer provides implementation | Developer writes code against a contract | Custom tools |
| **D: Plugin/Extension** | Optional capability added via plugin | `pip install cortex-redis-memory` | Memory backends |

---

### 3.1 Weight Engine

**Recommendation: Category B (Built-In with Defaults, Inspectable)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| Behavioral Weights (verbosity, formality, etc.) | **B** | Developers should see and override initial values. The adaptive learning is automatic. |
| Tool Preference Weights | **A** | Pure infrastructure. Developers never manually set tool preference scores -- the system learns from success/failure. |
| Model Selection Weights | **B** | Developers may want to bias the system toward specific models for specific task types. Initial hints are useful. |
| Goal Alignment Weights | **A** | Entirely managed by GoalTracker. No developer interaction needed. |
| User Insight Weights | **A** | Learned from implicit signals. Developer override would break the adaptation model. |
| Enterprise Weights | **B** | Admin-configurable via `EnterpriseConfig`. Not learned, set by policy. |
| Global Weights | **D** | Opt-in feature for cloud deployments. Disabled by default. |

**Should developers inspect/override weights?**

YES to inspection: `session.get_weights()` returns a read-only snapshot. This belongs in `response.metadata.brain_state` so developers can log it, display it in admin dashboards, or use it for debugging.

PARTIALLY to override: Developers should be able to set *initial* behavioral weights (e.g., "this agent should be formal and verbose") via `WeightConfig`. They should NOT be able to override weights mid-session in normal use -- only via an explicit escape hatch (`session.override_weight()`) marked as advanced API.

NO to modifying learning dynamics: Learning rates, momentum, homeostatic regulation, and clamping bounds are infrastructure. Exposing them invites misconfiguration. If a developer sets `behavioral_lr=10.0`, the weight system oscillates wildly. This is a footgun with no upside for 99% of users.

**Public API Surface:**
```python
# Level 1: Zero config (weights adapt automatically)
engine = cortex.Engine(providers={...})

# Level 2: Set initial preferences
agent = engine.create_agent(
    weight_config=WeightConfig(verbosity=0.5, formality=0.8)
)

# Level 2: Inspect current state
weights = session.get_weights()  # Returns read-only snapshot

# Level 3 (Advanced): Override a specific weight
session.override_weight("verbosity", 0.9)  # Escape hatch
```

---

### 3.2 Goal Tracker

**Recommendation: Category B (Built-In with Defaults, Configurable Thresholds)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| Loop detection (state hashing) | **A** | Always active, invisible. No developer should ever disable loop detection. |
| Drift scoring | **A** | Automatic. Developers see the score in metadata but do not configure how it is calculated. |
| Progress tracking | **B** | Developers can optionally provide explicit plan steps via `tracker.set_plan()`. Without it, heuristic progress works. |
| Threshold tuning | **B** | `DRIFT_WARNING`, `DRIFT_CRITICAL`, `LOOP_THRESHOLD`, `PROGRESS_STALL_TURNS` should be configurable but with sane defaults. |
| LLM verification | **B** | Optional: uses a fast model to verify alignment. Auto-enabled when budget allows, disabled for speed. |
| Replan recommendations | **A** | The tracker recommends actions; the Session acts on them. Developer sees the recommendation in metadata. |

**Public API Surface:**
```python
# Level 1: Automatic (developer does not even know goal tracking exists)
response = await session.run("Build a REST API")
# response.metadata.goal_progress == 0.3
# response.metadata.drift_score == 0.1
# response.metadata.loop_detected == False

# Level 2: Provide a plan for more accurate tracking
session.set_plan(["Design schema", "Implement endpoints", "Write tests"])

# Level 2: Tune thresholds
agent = engine.create_agent(
    goal_tracking=GoalTrackingConfig(
        drift_warning=0.4,    # Default 0.3
        loop_threshold=5,      # Default 3
    )
)

# Level 2: Disable (rare, for simple Q&A agents)
agent = engine.create_agent(goal_tracking=False)
```

---

### 3.3 Feedback Engine

**Recommendation: Category A/B Hybrid (Core detection is invisible; enterprise rules are configurable)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| Tier 1: Direct signal detection (patterns) | **A** | The regex patterns and heuristics for detecting frustration, satisfaction, correction are infrastructure. Developers should never edit regex patterns for signal detection. |
| Tier 1: Signal-to-weight mapping | **A** | How a frustration signal maps to verbosity reduction is neuroscience-inspired and should not be developer-configurable. |
| Tier 2: Cross-session preference learning | **A** | Automatic accumulation. The system builds user profiles without developer intervention. |
| Tier 3: Enterprise rules | **B** | Admin-configurable topic-safety rules. Configured via `TenantConfig.safety`. |
| Tier 4: Global aggregation | **D** | Opt-in plugin for cloud deployments. |
| Custom signal types | **D** | Extension point: developers register domain-specific signal detectors. |

**Should signal detection patterns be customizable?**

NO for the built-in patterns. The default patterns for frustration, satisfaction, correction, brevity, and speed preference are carefully tuned and language-model-informed. Letting developers edit `FRUSTRATION_PATTERNS` invites subtle breakage (e.g., removing `\bugh\b` means the system misses obvious frustration).

YES for adding domain-specific signals via an extension point:
```python
# Level 4: Register a custom signal detector (Plugin)
@cortex.signal_detector("customer_churn_risk")
def detect_churn(message: str) -> Optional[FeedbackSignal]:
    if "cancel" in message.lower() or "refund" in message.lower():
        return FeedbackSignal(
            signal_type="customer_churn_risk",
            strength=0.8,
            source="custom_churn_detector",
            evidence="Cancel/refund language detected",
        )
    return None
```

---

### 3.4 Prediction Engine

**Recommendation: Category A (Fully Built-In, Invisible)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| Prediction generation | **A** | Automatic before every action. No developer configuration needed. |
| Surprise signal computation | **A** | Prediction error is the core learning signal. This must not be developer-configurable -- incorrect surprise thresholds break the entire learning loop. |
| Calibration tracking | **A** | The engine self-calibrates. Developers see `calibration_score` in metadata. |
| Statistics maintenance | **A** | Running averages for tool success rates, latencies, qualities. Pure infrastructure. |

**Why fully invisible?** The Prediction Engine implements Karl Friston's Free Energy Principle -- a mathematical framework for how brains minimize prediction error. This is deeply counterintuitive for most developers. Exposing knobs like "surprise_threshold" or "learning_signal_dampening" would confuse 99% of users and be mistuned by the remaining 1%. The engine's output (surprise signals) feeds into Plasticity, which feeds into Weights, which feeds into observable behavior. The chain is invisible; only the result is visible.

**Public API Surface:**
```python
# Level 1: Invisible. Developer never interacts with predictions.
# They see the effects in response.metadata:
# response.metadata.model_used  <- influenced by prediction-driven routing
# response.metadata.latency_ms  <- prediction learns to estimate this

# Level 2 (Diagnostic only):
stats = session.get_prediction_stats()
# {"calibration_error": 0.15, "total_predictions": 42}
```

---

### 3.5 Plasticity Manager

**Recommendation: Category A (Fully Built-In, Invisible)**

**Should plasticity rules be configurable?**

NO. This is the strongest "fully invisible" recommendation in the entire analysis. Here is why:

1. **Complexity barrier:** Hebbian learning, LTP, LTD, homeostatic regulation, critical periods, and metaplasticity are neuroscience concepts. Asking a SaaS developer to tune `ltp_threshold` or `homeostatic_target_mean` is like asking a car driver to tune fuel injection timing.

2. **Interdependence:** The plasticity rules form a *system*. Changing one parameter (e.g., increasing LTP bonus) without adjusting homeostatic regulation causes runaway weight amplification. The parameters are balanced as a unit.

3. **No upside:** No developer has a domain-specific reason to change how Hebbian learning works. It is a universal learning mechanism, not a business logic concern.

4. **All downside:** A misconfigured plasticity system manifests as agents that stop adapting, oscillate between behaviors, or get "stuck" in suboptimal weight configurations. These failures are extremely hard to diagnose.

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| Hebbian learning | **A** | Universal mechanism. No developer tuning. |
| LTP (Long-Term Potentiation) | **A** | Success streak strengthening. Pure infrastructure. |
| LTD (Long-Term Depression) | **A** | Failure streak weakening. Pure infrastructure. |
| Homeostatic regulation | **A** | Prevents runaway weights. MUST NOT be disableable. |
| Critical periods | **A** | Early-session higher learning rates. Invisible. |
| Consolidation | **A** | Session-end cleanup. Automatic. |

**Public API Surface:**
```python
# Level 1: Invisible. Developer never interacts with plasticity.
# They see the effects in response.metadata.weights_delta

# Level 2 (Diagnostic only):
stats = session.get_plasticity_stats()
# {"total_events": 15, "plasticity_multiplier": 1.4, ...}
```

---

### 3.6 Adaptation Filter

**Recommendation: Category A (Fully Built-In, Invisible)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| Rapid adaptation (signal decay) | **A** | How fast repeated signals decay is a perceptual constant, not a business parameter. |
| Sustained adaptation (habituation) | **A** | The habituation threshold determines when the system stops reacting to steady-state signals. This is neuroscience, not business logic. |
| Behavioral shift detection | **A** | Comparing current behavior to baseline is automatic. |
| Dishabituation recovery | **A** | How long until a habituated signal can fire again. Infrastructure timing parameter. |

**Rationale for full invisibility:** The Adaptation Filter's insight -- "react to CHANGES, not steady states" -- is universally correct. A user who always sends 3-word messages is not signaling brevity preference; that is their communication style. Letting developers configure `habituation_threshold=1` (immediate habituation) or `habituation_threshold=100` (never habituates) breaks this fundamental insight. The current default of 8 repetitions before habituation is calibrated and should remain infrastructure.

---

### 3.7 Memory Fabric

**Recommendation: Category B/C/D Hybrid (Most extensible module)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| Working Memory (session state) | **B** | Built-in with configurable capacity. Default 100 items is sane for most use cases. |
| Episodic Memory (past experiences) | **B** | Built-in with configurable capacity. Developers may want larger episodic stores for long-lived agents. |
| Semantic Memory (domain knowledge) | **B** | Built-in with configurable capacity. Developers may pre-populate with domain knowledge. |
| Memory consolidation (sleep cycle) | **A** | The algorithm for promoting important working memories to long-term storage is infrastructure. |
| Relevance scoring | **A** | How memories are ranked for retrieval is infrastructure. |
| In-Memory backend | **B** | Default. Works everywhere, volatile. |
| File-based backend | **B** | Built-in persistent option for on-prem. |
| MemoryBackend interface | **C** | The abstract base class for custom backends. Developer implements for their infrastructure. |
| Redis backend | **D** | Plugin: `pip install cortex-memory-redis` |
| PostgreSQL backend | **D** | Plugin: `pip install cortex-memory-postgres` |
| Vector DB backend | **D** | Plugin: `pip install cortex-memory-vectordb` |
| S3/GCS backend | **D** | Plugin: `pip install cortex-memory-cloud` |

**Should backends be swappable?**

YES. This is the single most important extensibility point. The `MemoryBackend` abstract class already defines the correct interface (`get`, `put`, `delete`, `search`, `list_all`, `clear`, `count`). Developers in enterprise environments will need:
- Redis for distributed sessions
- PostgreSQL for compliance (audit trails require durable storage)
- Vector databases for semantic search over large knowledge bases
- Cloud storage for multi-region deployments

The current architecture already supports this through dependency injection in `MemoryFabric.__init__()`. This is correct and should be prominently documented.

**Public API Surface:**
```python
# Level 1: Zero config (in-memory, volatile)
engine = cortex.Engine(providers={...})

# Level 2: File-based persistence
engine = cortex.Engine(
    memory=MemoryConfig(backend="file", path="./data/memory")
)

# Level 3: Custom backend
from cortex_memory_redis import RedisBackend
engine = cortex.Engine(
    memory=MemoryConfig(
        working_backend=RedisBackend(url="redis://localhost"),
        episodic_backend=RedisBackend(url="redis://localhost"),
        semantic_backend=VectorDBBackend(url="http://qdrant:6333"),
    )
)

# Level 2: Pre-populate semantic memory
agent = engine.create_agent(name="support")
session = agent.start_session(user_id="user_123")
session.memory.semantic.learn(SemanticEntry(
    entry_id="product_v2",
    topic="Product pricing",
    content="Premium plan costs $99/month...",
    confidence=1.0,
))

# Level 2: Adjust capacities
engine = cortex.Engine(
    memory=MemoryConfig(
        working_capacity=200,
        episodic_capacity=2000,
        semantic_capacity=10000,
    )
)
```

---

### 3.8 Population Coding

**Recommendation: Category A/D Hybrid (Built-in heuristics, extensible evaluators)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| PopulationDecoder (aggregation algorithm) | **A** | Confidence-weighted averaging with outlier suppression is the correct algorithm. Not configurable. |
| Default quality heuristics (length, completeness, error) | **A** | Built-in heuristics that provide baseline quality estimation. |
| Custom quality heuristics | **D** | Extension point: developers register domain-specific quality measures. |
| PopulationToolSelector | **A** | Internal mechanism for tool selection. Invisible to developers. |
| Custom tool evaluators | **D** | Extension point: developers add evaluators for their custom tools. |
| Outlier threshold | **B** | Default 2.0 (z-score). Rarely needs tuning, but available. |

**Should evaluators be extensible?**

YES. This is a well-scoped extension point. The default quality heuristics (length, completeness, error detection) are generic. Domain-specific applications need domain-specific quality measures:

```python
# Level 4: Add a domain-specific quality heuristic
@cortex.quality_heuristic("medical_accuracy")
def medical_quality(response: str, **context) -> EvaluatorResult:
    # Check for required medical disclaimers
    has_disclaimer = "consult a healthcare provider" in response.lower()
    return EvaluatorResult(
        score=0.9 if has_disclaimer else 0.3,
        confidence=0.8,
        label="medical_accuracy",
    )

# The heuristic is automatically added to the PopulationQualityEstimator
```

---

### 3.9 Orchestrator

**Recommendation: Category B (Built-In with Configurable Thresholds and Policies)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| Autonomy scoring algorithm (population-coded) | **A** | The multi-evaluator scoring approach is infrastructure. |
| Routing (autonomous/timer/blocking) | **A** | The three-tier routing model is the core value proposition. |
| Keyword risk analysis | **B** | Default `RISK_KEYWORDS` and `SAFE_KEYWORDS` are sane. Developers may need to add domain-specific risk terms. |
| Safety policy enforcement | **B** | Configured via `TenantConfig.safety`. Admin sets the policy, orchestrator enforces. |
| Autonomy thresholds | **B** | `blocking_threshold=3.0`, `timer_threshold=6.0` should be configurable per-tenant. |
| Timer duration | **B** | Default 5 minutes. Should be configurable. |
| Custom autonomy evaluators | **D** | Extension point: add domain-specific risk evaluators to the population decoder. |
| Pending decision management | **A** | Internal state management. Developer interacts via `approve()` and `veto()`. |

**Should autonomy thresholds be configurable per-tenant?**

YES. This is a critical enterprise requirement. Different tenants have different risk tolerances:
- A customer support SaaS: high autonomy (score 8.0+ for autonomous)
- A financial trading platform: low autonomy (score 2.0 for blocking everything)
- A healthcare application: strict (HIPAA requires human approval for all clinical decisions)

```python
# Level 2: Per-tenant autonomy configuration
tenant_config = TenantConfig(
    safety=SafetyPolicy(
        level=SafetyLevel.STRICT,
        max_autonomy=0.5,  # Cap autonomy at 50%
        require_human_approval=["delete", "transfer", "deploy"],
    ),
    custom_settings={
        "autonomy_thresholds": {
            "blocking": 5.0,   # Higher threshold = more things need approval
            "timer": 8.0,
        },
        "timer_duration_seconds": 600,  # 10 minutes instead of 5
    }
)
```

---

### 3.10 Tool Framework

**Recommendation: Category C (SDK Provides Framework, Developer Implements)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| `@cortex.tool` decorator | **B** | Built-in, zero-config type inference from Python signatures. |
| Tool execution engine (timeout, error handling) | **A** | Infrastructure: timeout enforcement, error wrapping, latency tracking. Invisible. |
| Tool definitions (JSON schema generation) | **A** | Auto-generated from type hints. Developer never writes JSON schema manually. |
| Tool implementations | **C** | Developer writes the tool functions. This is 100% business logic. |
| Built-in tools (web search, code exec, etc.) | **D** | Optional plugins. Not in core SDK. |
| MCP tool server compatibility | **D** | Future extension: bridge to MCP ecosystem. |

**Public API Surface:**
```python
# Level 2: Simple tool registration
@cortex.tool(name="lookup_order", description="Look up order status")
async def lookup_order(order_id: str) -> str:
    order = await db.orders.find(order_id)
    return f"Order {order_id}: {order.status}"

# Level 2: Pass tools to agent
agent = engine.create_agent(
    name="support",
    tools=[lookup_order, cancel_order, refund_order],
)

# Level 3: Custom tool with explicit schema
@cortex.tool(
    name="complex_query",
    parameters={
        "type": "object",
        "properties": {
            "sql": {"type": "string", "description": "SQL query to execute"},
            "database": {"type": "string", "enum": ["prod", "staging"]},
        },
        "required": ["sql", "database"],
    }
)
async def complex_query(sql: str, database: str) -> str:
    ...
```

---

### 3.11 Enterprise Config

**Recommendation: Category B (Admin-Configurable, Developer-Visible)**

| Sub-Component | Category | Admin | Developer | End User |
|--------------|----------|-------|-----------|----------|
| Safety policies (level, blocked topics, PII) | **B** | Full control | Read-only | N/A |
| Model policies (allowed models, token limits) | **B** | Full control | Read-only | N/A |
| Tool policies (allowed tools, approval required) | **B** | Full control | Read-only | N/A |
| Audit config (logging, destinations, retention) | **B** | Full control | N/A | N/A |
| Data retention policy | **B** | Full control | Read-only | N/A |
| Compliance frameworks | **B** | Full control | Read-only | N/A |
| User-overridable settings | **B** | Defines which settings users can override | Can check overridability | Can override allowed settings |
| License management | **A** | Activated once | Checked automatically | N/A |
| Custom enterprise settings | **B** | Full control | Read-only | N/A |

**What should be admin-only vs. developer-configurable?**

| Setting | Admin-Only | Developer-Configurable | Rationale |
|---------|-----------|----------------------|-----------|
| `safety.level` | Yes | No | Security cannot be relaxed by developers |
| `safety.blocked_topics` | Yes | No | Content policy is org-wide |
| `safety.max_autonomy` | Yes | No | Risk tolerance is organizational |
| `models.allowed_models` | Yes | No | Model selection is a cost/security decision |
| `models.max_tokens_per_request` | Yes | Per-agent override within admin limits | Developers need per-agent tuning |
| `tools.blocked_tools` | Yes | No | Tool access is a security concern |
| `audit.enabled` | Yes | No | Compliance requirements are non-negotiable |
| `data_retention` | Yes | No | Data governance is organizational |
| Agent system prompts | No | Yes | Business logic |
| Agent weight_config | No | Yes | Behavioral tuning |
| Session-level model override | No | Yes, within allowed_models | Developer knows best model for task |

---

### 3.12 LLM Router

**Recommendation: Category B (Built-In Routing, Configurable Providers and Models)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| Task classification heuristic | **A** | Internal routing logic. Developers do not classify tasks. |
| Temperature auto-selection | **A** | Per-task, per-provider temperature is infrastructure. |
| Failure tracking and fallback | **A** | Automatic retry with alternative model. Invisible. |
| Latency-based routing | **A** | Running averages influence model selection. Invisible. |
| Provider registration | **B** | Developer provides API keys and optional config. |
| Model weight hints | **B** | Developer can hint: "prefer Gemini for coding tasks." |
| Orchestrator/worker model selection | **B** | Developer sets which models serve which roles. |
| Custom provider adapter | **C** | SDK provides `BaseLLMProvider` interface; developer implements for custom LLM infrastructure. |

**Public API Surface:**
```python
# Level 1: Single provider, auto-routing
engine = cortex.Engine(providers={"openai": {"api_key": "sk-..."}})

# Level 2: Multi-provider with role assignment
engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-...", "default_model": "gpt-4o"},
        "gemini": {"api_key": "AIza...", "default_model": "gemini-3-pro"},
    },
    orchestrator_model="gpt-4o",
    worker_model="gemini-3-flash",
)

# Level 2: Model preference hints
engine.router.set_model_weights({
    "coding": {"gpt-4o": 0.9, "gemini-3-pro": 0.7},
    "conversation": {"gemini-3-pro": 0.9, "gpt-4o": 0.7},
})

# Level 3: Custom LLM provider
from cortex.core.llm.base import BaseLLMProvider
class MyCustomProvider(BaseLLMProvider):
    async def generate(self, messages, model, **kwargs) -> LLMResponse:
        ...
```

---

### 3.13 Licensing

**Recommendation: Category A (Fully Built-In, Invisible After Activation)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| License activation | **B** | One-time setup: `engine.activate("LK-xxx")` |
| License validation | **A** | Checked automatically. Developer never calls `check_status()`. |
| Seat management | **A** | Automatic tracking. |
| Usage metering | **A** | Automatic. Developer can read reports but does not manage metering. |
| Grace period logic | **A** | Invisible. On-prem deployments get 30-day grace automatically. |
| Feature gating | **A** | Features are enabled/disabled based on plan. No developer configuration. |

---

### 3.14 Update Manager

**Recommendation: Category A (Fully Built-In, Invisible)**

| Sub-Component | Category | Rationale |
|--------------|----------|-----------|
| On-prem update checking | **A** | Periodic check for new versions. |
| Update delivery | **A** | Downloads and stages updates. |
| Update application | **B** | Developer can choose when to apply updates (auto vs. manual). |
| Rollback | **B** | Developer can trigger rollback to previous version. |

---

## 4. SDK Surface API Design

### 4.1 What Developers See (Public API)

```python
import cortex

# --- Engine (Entry Point) ---
engine = cortex.Engine(
    providers={"openai": {"api_key": "..."}},      # Required
    enterprise_config=EnterpriseConfig(...),         # Optional
    orchestrator_model="gpt-4o",                    # Optional
    worker_model="gpt-4o-mini",                     # Optional
    memory=MemoryConfig(...),                       # Optional
    weight_persistence_path="./weights",            # Optional
)

# --- Agent (Template) ---
agent = engine.create_agent(
    name="support",                                 # Required
    system_prompt="You are a helpful assistant.",    # Recommended
    tools=[my_tool_1, my_tool_2],                  # Optional
    weight_config=WeightConfig(                    # Optional
        verbosity=0.5,
        formality=0.8,
        autonomy=0.7,
    ),
    goal_tracking=True,                            # Default: True
)

# --- Session (Stateful Conversation) ---
session = agent.start_session(user_id="user_123")
response = await session.run("Help me with my order")

# --- Response (Output) ---
response.content          # The LLM's text response
response.metadata         # ResponseMetadata with brain state
response.artifacts        # Any generated files/data

# --- Metadata (Brain State, Read-Only) ---
response.metadata.goal_progress      # 0.0 to 1.0
response.metadata.drift_score        # 0.0 to 1.0
response.metadata.loop_detected      # bool
response.metadata.model_used         # "gpt-4o"
response.metadata.tokens_used        # 1523
response.metadata.latency_ms         # 2340.5
response.metadata.tools_called       # ["lookup_order"]
response.metadata.weights_delta      # {"verbosity": -0.02, ...}

# --- Session Introspection (Diagnostic) ---
session.get_weights()              # Full weight snapshot
session.get_goal_progress()        # Goal tracking summary
session.get_prediction_stats()     # Prediction calibration
session.get_plasticity_stats()     # Plasticity events
session.get_adaptation_stats()     # Habituation state

# --- Session Lifecycle ---
summary = await session.close()    # Consolidate + cleanup
```

### 4.2 What Developers Do NOT See (Hidden Infrastructure)

These components operate automatically within `session.run()`:

1. **Feedback signal detection** -- Every user message is scanned for implicit signals (frustration, satisfaction, brevity, speed, detail). Signals are adaptation-filtered (repetitive signals decay, novel signals amplify).

2. **Weight updates** -- Feedback signals cause weight updates via the Feedback Engine. Tool success/failure updates tool preference weights. Model performance updates model selection weights.

3. **Prediction cycle** -- Before each LLM call, the Prediction Engine estimates outcome, latency, and quality. After the call, it compares prediction to actual, generating a surprise signal.

4. **Plasticity cascade** -- The surprise signal feeds into Plasticity Manager. Hebbian learning strengthens successful associations. LTP/LTD amplify streaks. Homeostasis prevents runaway weights. Critical period modulation adjusts learning rates based on session maturity.

5. **Goal verification** -- Every step is verified against the original goal via heuristic + optional LLM check. Drift is measured. Loops are detected via state hashing.

6. **Memory management** -- Working memory stores each turn. Consolidation promotes important items to episodic/semantic. Eviction removes low-importance items when capacity is reached.

7. **Quality estimation** -- Population coding aggregates multiple heuristic quality estimates for every response.

8. **Model routing** -- Task classification, weight-based scoring, failure tracking, and latency history all feed into model selection. Temperature is auto-selected per task type and provider.

---

## 5. Configuration Hierarchy

Configuration cascades from broadest scope (SDK defaults) to narrowest scope (enterprise policy), with later layers overriding earlier ones -- except where enterprise policy acts as an inviolable ceiling.

```
Layer 1: SDK Defaults (baked into code)
    |
    v
Layer 2: Engine Config (cortex.Engine constructor)
    |
    v
Layer 3: Agent Config (engine.create_agent parameters)
    |
    v
Layer 4: Session Config (agent.start_session / session.run parameters)
    |
    v
Layer 5: Runtime Adaptation (weight system, plasticity, feedback)
    |
    [ceiling]
    |
Layer 6: Enterprise Policy (TenantConfig -- constrains all above)
```

### Cascade Rules

| Parameter | SDK Default | Engine Override | Agent Override | Session Override | Enterprise Ceiling |
|-----------|------------|----------------|---------------|-----------------|-------------------|
| `verbosity` | 0.0 | N/A | Via WeightConfig | Via override_weight | N/A |
| `autonomy` | 0.5 | N/A | Via WeightConfig | Via override_weight | `safety.max_autonomy` |
| `safety_level` | moderate | Via EnterpriseConfig | N/A | N/A | Fixed by admin |
| `orchestrator_model` | First registered | Engine constructor | Per-agent override | Per-run override | `models.allowed_models` |
| `max_tokens` | 32000 | Engine config | Per-agent config | Per-run config | `models.max_tokens_per_request` |
| `goal_tracking` | True | N/A | Per-agent bool | N/A | N/A |
| `tool_timeout` | 30s | Engine config | N/A | N/A | `tools.tool_timeout_seconds` |
| `memory_capacity` | 100/500/1000 | Engine config | N/A | N/A | N/A |

### Enterprise Policy as Inviolable Ceiling

Enterprise config NEVER removes functionality. It CONSTRAINS:
- If admin sets `max_autonomy=0.5`, the weight system can adapt autonomy between 0.0 and 0.5, but never above.
- If admin sets `allowed_models=["gpt-4o"]`, the LLM Router only routes to GPT-4o, regardless of weight-based scoring.
- If admin sets `blocked_topics=["weapons"]`, the Orchestrator blocks requests matching that topic regardless of autonomy score.
- If admin sets `audit.enabled=True`, all events are logged regardless of developer preferences.

The developer cannot weaken enterprise policy. They can only operate within its bounds.

---

## 6. Extensibility Points

### 6.1 Formal Extension Interfaces

| Extension Point | Interface | Category | Use Case |
|----------------|-----------|----------|----------|
| **Memory Backend** | `MemoryBackend` ABC | C/D | Redis, PostgreSQL, Vector DB storage |
| **LLM Provider** | `BaseLLMProvider` ABC | C | Custom model infrastructure |
| **Tool Function** | `@cortex.tool` decorator | C | Domain-specific tools |
| **Quality Heuristic** | `Evaluator` callable | D | Domain-specific quality assessment |
| **Signal Detector** | `@cortex.signal_detector` (proposed) | D | Domain-specific feedback signals |
| **Autonomy Evaluator** | `Evaluator` callable (via PopulationDecoder) | D | Domain-specific risk assessment |
| **Audit Exporter** | `AuditExporter` (proposed) | D | Custom audit log destinations |

### 6.2 Plugin Architecture

Plugins should be separate pip-installable packages that register themselves via entry points:

```
cortex-memory-redis       -> RedisBackend for MemoryFabric
cortex-memory-postgres    -> PostgreSQLBackend for MemoryFabric
cortex-memory-qdrant      -> QdrantBackend for semantic search
cortex-provider-anthropic -> Claude provider adapter
cortex-provider-cohere    -> Cohere provider adapter
cortex-tools-browser      -> Browser automation subsystem
cortex-tools-code-exec    -> Sandboxed code execution
cortex-audit-splunk       -> Splunk audit log exporter
cortex-audit-datadog      -> Datadog metrics exporter
```

### 6.3 Event Bus for Extensibility

The existing EventBus (from `corteX/core/events.py`) should be the primary extensibility mechanism for cross-cutting concerns. Third-party code subscribes to events rather than modifying core behavior:

```python
# Extension: log every tool call to external system
@cortex.on_event("tool.executed")
def log_tool_call(event):
    external_logger.log({
        "tool": event.tool_name,
        "success": event.success,
        "latency_ms": event.latency_ms,
        "timestamp": event.timestamp,
    })

# Extension: custom metric on weight changes
@cortex.on_event("weight.updated")
def track_weight_drift(event):
    prometheus_gauge.set(event.category, event.key, event.new_value)
```

---

## 7. Anti-Patterns to Avoid

### 7.1 LangChain's Over-Abstraction

**The mistake:** Abstracting everything -- prompts, chains, agents, memory, tools, output parsers, callbacks -- into deep class hierarchies with multiple inheritance paths for the same concept.

**How corteX avoids it:** The "13 invisible steps in `session.run()`" pattern means the developer calls ONE method. They do not construct a chain of `PromptTemplate | LLM | OutputParser | Memory | Callback`. The framework orchestrates internally.

**Specific guardrails:**
- No public class should have more than one level of inheritance from ABC
- No developer should need to import more than 5 symbols from corteX for a typical use case
- The "getting started" code should be 4 lines, not 40

### 7.2 CrewAI's Over-Simplification

**The mistake:** Making the "role + goal + backstory" model the ONLY way to define agents. This forces every agent into a human-role metaphor, even when the agent is doing pure computation.

**How corteX avoids it:** `engine.create_agent(name="support", system_prompt="...")` is simple like CrewAI, but the developer is not forced into a role-playing metaphor. The system prompt can be anything. Weight configuration is optional. Tools are optional. Goal tracking is optional (defaulting to on).

**Specific guardrails:**
- Every feature must have a sensible default
- No feature should be mandatory except `name` and at least one LLM provider
- Advanced features must be accessible without changing the basic API shape

### 7.3 AutoGen's Conceptual Overhead

**The mistake:** Requiring developers to understand "Proxies," "Initiate Chats," "Termination Conditions," and "Group Chat Managers" before building anything useful. The mental model is too alien from how developers think.

**How corteX avoids it:** The mental model is: Engine -> Agent -> Session -> Run. This maps to how every web developer already thinks: Application -> Controller -> Session -> Request.

**Specific guardrails:**
- Every concept in the public API must be explainable in one sentence
- If a concept requires a paragraph to explain, it belongs in the hidden infrastructure

### 7.4 The "Leaky Abstraction" Anti-Pattern

**The mistake (common to all):** When abstractions leak, developers must understand the underlying implementation to debug problems. LangChain's nested chains leak when error messages reference internal chain state. CrewAI leaks when token limits hit mid-task.

**How corteX avoids it:** The Response object includes rich metadata (goal progress, drift, loop detection, model used, tokens, latency, tools called) specifically so that developers can diagnose problems *without* understanding internal state. The metadata IS the debugging interface.

**Specific guardrails:**
- Every error message should include enough context to diagnose without internal knowledge
- `response.metadata` should contain everything needed for debugging
- Internal exceptions should be wrapped in user-facing error types with clear messages

### 7.5 The "Invisible Failure" Anti-Pattern

**The mistake:** Brain engine modules failing silently. If the Prediction Engine throws an exception, `session.run()` should still return a response -- but the metadata should indicate that prediction was unavailable.

**How corteX avoids it:** Every brain module in the `session.run()` pipeline should be wrapped in try/except with degraded-mode operation. The response always includes what worked and what was degraded.

### 7.6 The "Configuration Explosion" Anti-Pattern

**The mistake:** Exposing every internal parameter as a configurable option. This creates the "configuration treadmill" where developers spend more time configuring than building.

**How corteX avoids it:** Of the 14 modules, 6 are Category A (zero config), 6 are Category B (optional tuning), 1 is Category C (developer-implemented), and nearly all have Category D extension points. A developer using only Category A and B features (the vast majority) touches at most: LLM provider config, system prompt, tools, and optionally weight_config. Four things. Not forty.

---

## 8. Summary: The Boundary Map

```
                    INVISIBLE                    VISIBLE
                  (Category A)               (Category B/C/D)
                       |                           |
    +-----------+------+------+-----------+--------+--------+---------+
    |           |             |           |                  |         |
 Prediction  Plasticity  Adaptation   Feedback          Weight      Memory
  Engine      Manager     Filter     Engine (T1/T2)   Engine     Fabric
    |           |             |           |              |           |
    |      Hebbian,LTP   Rapid/Sust.  Signal          Initial    Backends
    |      LTD,Homeo.    Habituation  Detection       Weights    (C/D)
    |      CritPeriods                                   |
    |           |             |           |              |
    +-----+----+------+------+-----+-----+------+------+
          |                        |                    |
     Goal Tracker              Population           LLM Router
     (A: loop/drift)          Coding               (B: providers,
     (B: thresholds)          (A: algorithm)         models, hints)
                              (D: evaluators)
          |                        |                    |
     +----+----+              +----+----+          +----+----+
     |         |              |         |          |         |
  Orchestrator Tool          Enterprise Licensing  Update
  (B: autonomy Framework     Config    (A)        Manager
   thresholds) (C: developer (B: admin             (A)
               implements)   configures)
```

### The One-Sentence Rule

> **If the developer needs to understand neuroscience to use it, it is Category A.**
> **If the developer needs to understand their business to use it, it is Category B or C.**
> **If the developer needs to understand their infrastructure to use it, it is Category D.**

---

## 9. Recommended Metadata in Response

**What the brain engine should expose in `response.metadata`:**

```python
@dataclass
class ResponseMetadata:
    # Goal tracking (visible)
    goal_progress: float           # 0.0 to 1.0
    drift_score: float             # 0.0 to 1.0
    loop_detected: bool
    recommended_action: str        # "continue", "adjust", "replan"

    # Operational (visible)
    model_used: str
    tokens_used: int
    latency_ms: float
    tools_called: List[str]
    steps_taken: int

    # Brain state summary (visible, read-only)
    weights_delta: Dict[str, float]   # Which weights changed this turn
    quality_estimate: float            # Population-coded quality (0.0-1.0)
    prediction_calibration: float      # How well-calibrated predictions are
    adaptation_state: str              # "learning" | "adapted" | "habituated"

    # NOT exposed (infrastructure):
    # - Raw plasticity events
    # - Individual population votes
    # - Surprise signal details
    # - Momentum values
    # - State hashes
    # - Prediction internals
```

The principle: expose *summaries* and *scores*, not *mechanics*. A developer can see "quality_estimate: 0.72" and act on it. They should never see "hebbian_delta for tool.code_interpreter: +0.043, modulated by critical_period_multiplier 1.6 and surprise_learning_signal 0.28."

---

## 10. Implementation Priority

### Phase 1: Solidify Category A Boundaries
- Ensure all Category A modules are completely invisible in the public API
- Remove any direct references to Plasticity, Adaptation, or Prediction from developer-facing code
- Wrap all brain modules in try/except with graceful degradation

### Phase 2: Formalize Category B Configuration
- Create `WeightConfig`, `GoalTrackingConfig`, `MemoryConfig` as Pydantic models with validated defaults
- Add comprehensive docstrings explaining what each configurable parameter does (in business terms, not neuroscience terms)
- Ensure enterprise config correctly acts as a ceiling on all other config

### Phase 3: Define Category C/D Interfaces
- Publish `MemoryBackend` as a stable, versioned interface
- Publish `BaseLLMProvider` as a stable, versioned interface
- Define `@cortex.quality_heuristic` and `@cortex.signal_detector` extension decorators
- Create a plugin registration mechanism via Python entry points

### Phase 4: Build Plugin Ecosystem
- Release `cortex-memory-redis` and `cortex-memory-postgres` as reference implementations
- Release `cortex-audit-*` exporters for common observability platforms
- Document plugin development guide with templates
