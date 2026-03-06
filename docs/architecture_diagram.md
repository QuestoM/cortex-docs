# corteX SDK -- Complete Architecture Diagram

> **corteX** is an enterprise-grade, brain-inspired AI Agent SDK that replaces LangChain, CrewAI,
> and AutoGen. It delivers consistent, goal-driven AI agents with 100% on-prem capability.
> This document contains a comprehensive Mermaid architecture diagram covering every module,
> component, and data flow in the SDK.
>
> **6,200+ tests passing** | **103+ modules** | **4 LLM providers** (OpenAI, Gemini, Anthropic, Local) | **Agentic Engine** with full brain integration | **18 new Wave 1 modules** (Goal Intelligence, Anti-Drift, Cognitive Context, LLM Routing, Multi-Tenancy)

## How to Read This Diagram

| Color | Component Type |
|-------|---------------|
| Yellow (`dev`) | Developer / User entry points |
| Blue (`sdk`) | SDK Core classes (Engine, Agent, Session) |
| Dark Blue (`agentic`) | Agentic Engine (orchestration, planning, context, delegation) |
| Purple (`brain`) | Brain Engine (neuroscience-inspired) |
| Green (`mem`) | Context & Memory systems |
| Red/Orange (`llm`) | LLM Providers & AI layer |
| Teal (`tool`) | Tools & Plugins |
| Grey (`ent`) | Enterprise & Security |
| Pink (`math`) | Mathematical foundations |
| Maroon (`nl`) | NeuroLlama (Layer 3) |

---

```mermaid
graph TD
    classDef dev fill:#FFF3CD,stroke:#856404,color:#856404,stroke-width:2px
    classDef sdk fill:#CCE5FF,stroke:#004085,color:#004085,stroke-width:2px
    classDef agentic fill:#B3D9FF,stroke:#003366,color:#003366,stroke-width:2px
    classDef brain fill:#E8D5F5,stroke:#6F42C1,color:#6F42C1,stroke-width:2px
    classDef mem fill:#D4EDDA,stroke:#155724,color:#155724,stroke-width:2px
    classDef llm fill:#F8D7DA,stroke:#721C24,color:#721C24,stroke-width:2px
    classDef tool fill:#D1ECF1,stroke:#0C5460,color:#0C5460,stroke-width:2px
    classDef ent fill:#E2E3E5,stroke:#383D41,color:#383D41,stroke-width:2px
    classDef math fill:#FCE4EC,stroke:#880E4F,color:#880E4F,stroke-width:2px
    classDef nl fill:#FFCCBC,stroke:#BF360C,color:#BF360C,stroke-width:2px

    %% ===================================================================
    %% A. DEVELOPER ENTRY POINTS
    %% ===================================================================
    subgraph DEV_LAYER ["<b>A. נקודות כניסה -- Developer API</b>"]
        direction TB
        DEV_CODE["Developer Code<br/><i>import cortex</i>"]:::dev
        TOOL_DECORATOR["@cortex.tool()<br/><i>Tool Decorator</i>"]:::dev
        CONFIG["WeightConfig / EnterpriseConfig<br/>ContextManagementConfig"]:::dev
    end

    %% ===================================================================
    %% B. SDK CORE
    %% ===================================================================
    subgraph SDK_CORE ["<b>B. ליבת ה-SDK -- Core Classes</b>"]
        direction TB
        ENGINE["<b>Engine</b><br/>Provider management<br/>Agent factory"]:::sdk
        AGENT["<b>Agent</b><br/>Stateless template<br/>system_prompt + tools + config"]:::sdk
        SESSION["<b>Session</b><br/>Stateful runtime<br/>Full brain integration"]:::sdk
        RUN["<b>run()</b><br/>Single-turn execution<br/>ContextCompiler.compile() before LLM<br/>MemoryFabric.get_relevant_context()<br/>Full 7-param brain bundle"]:::sdk
        RUN_AGENTIC["<b>run_agentic()</b><br/>Multi-step goal-driven<br/>Full 7-param brain bundle"]:::sdk
        RUN_STREAM["<b>run_stream()</b><br/>Streaming + tool execution loop<br/>StreamChunk: model, chunk_type<br/>Full 7-param brain bundle"]:::sdk
        RESPONSE["<b>Response</b><br/>content + metadata<br/>+ artifacts"]:::sdk
    end

    DEV_CODE -->|"engine = cortex.Engine(providers=...)"| ENGINE
    ENGINE -->|"engine.create_agent()"| AGENT
    AGENT -->|"agent.start_session(user_id)"| SESSION
    CONFIG -->|"Configuration"| ENGINE
    TOOL_DECORATOR -->|"Register tools"| AGENT
    SESSION --> RUN
    SESSION --> RUN_AGENTIC
    SESSION --> RUN_STREAM
    RUN --> RESPONSE
    RUN_AGENTIC --> RESPONSE
    RUN_STREAM --> RESPONSE

    %% ===================================================================
    %% C. AGENTIC ENGINE
    %% ===================================================================
    subgraph AGENTIC_ENGINE ["<b>C. מנוע Agentic -- Agentic Engine</b>"]
        direction TB
        AGENT_LOOP["<b>AgentLoop</b><br/>Generator yields LoopActions<br/>Thin orchestrator<br/>Uses AdaptiveBudget for step/token limits"]:::agentic
        LOOP_ACTIONS["<b>LoopActionType</b><br/>LLM_CALL | TOOL_CALL<br/>USER_INTERACTION<br/>COMPACTION | PLAN_GEN<br/>REFLECTION | IMPROVEMENT<br/>COMPLETE | ABORT"]:::agentic

        subgraph AGENTIC_SUB_ENGINES ["Sub-Engines"]
            direction TB
            CTX_COMPILER["<b>ContextCompiler</b><br/>4-Zone Assembly<br/>System(12%) | Persistent(8%)<br/>Working(40%) | Recent(40%)<br/>Wired into run() + run_agentic()<br/>Receives L2/L3 summaries"]:::agentic
            PLANNER["<b>PlanningEngine</b><br/>Goal decomposition<br/>Step scheduling<br/>Replanning on failure<br/>Uses GoalTree for hierarchical plans"]:::agentic
            REFLECTOR["<b>ReflectionEngine</b><br/>Quality verification<br/>Lesson bank<br/>Improvement prompts"]:::agentic
            RECOVERY_ENG["<b>RecoveryEngine</b><br/>Error classification<br/>Retry strategies<br/>Exponential backoff<br/>Backtracking"]:::agentic
            INTERACTION_MGR["<b>InteractionManager</b><br/>L1-L5 Autonomy<br/>Smart timeout<br/>Auto-decision"]:::agentic
            POLICY_ENG["<b>PolicyEngine</b><br/>5 Guardrails:<br/>IntentGuard | Playbook<br/>ToolApproval | ToolGuide<br/>OutputFormatter"]:::agentic
            SUB_AGENT_MGR["<b>SubAgentManager</b><br/>Isolated context windows<br/>Shared token budget<br/>Creates + executes sub-agents<br/>Task delegation + result summary"]:::agentic
        end
    end

    RUN -->|"compile() before LLM"| CTX_COMPILER
    RUN_AGENTIC -->|"Drives loop"| AGENT_LOOP
    RUN_STREAM -->|"Tool execution loop"| TOOL_EXEC
    RUN_STREAM -->|"Stream + tools"| LLM_ROUTER
    AGENT_LOOP -->|"Yields"| LOOP_ACTIONS
    AGENT_LOOP --> CTX_COMPILER
    AGENT_LOOP --> PLANNER
    AGENT_LOOP --> REFLECTOR
    AGENT_LOOP --> RECOVERY_ENG
    AGENT_LOOP --> INTERACTION_MGR
    AGENT_LOOP --> POLICY_ENG
    AGENT_LOOP --> SUB_AGENT_MGR
    SUB_AGENT_MGR -->|"Creates sub-agent tasks"| SESSION
    SUB_AGENT_MGR -->|"Isolated execution"| LLM_ROUTER

    %% ===================================================================
    %% D. BRAIN ENGINE -- NEUROSCIENCE CORE
    %% ===================================================================
    subgraph BRAIN_ENGINE ["<b>D. מנוע המוח -- Brain Engine</b>"]
        direction TB

        subgraph BRAIN_CORE ["Core Brain Systems"]
            direction TB
            WEIGHTS["<b>WeightEngine</b><br/>7 categories:<br/>Behavioral | Tool | Model<br/>Goal | User | Enterprise | Global<br/>Bayesian posteriors"]:::brain
            GOAL_TRACK["<b>GoalTracker</b><br/>Progress monitoring<br/>Enhanced by GoalDNA + LoopDetector<br/>Uses DriftEngine for drift detection"]:::brain
            GOAL_DNA["<b>GoalDNA</b><br/>Token fingerprint<br/>O(1) drift detection<br/>Goal identity hash"]:::brain
            GOAL_REMINDER["<b>GoalReminderInjector</b><br/>Adaptive goal reminders<br/>Per LLM call injection<br/>Drift-based triggering"]:::brain
            GOAL_TREE["<b>GoalTree</b><br/>Hierarchical goals<br/>Goal→SubGoals→Steps<br/>Feeds PlanningEngine"]:::brain
            LOOP_DETECTOR["<b>MultiResolutionLoopDetector</b><br/>4 parallel detectors:<br/>State hash | Semantic | Action | Temporal"]:::brain
            DRIFT_ENGINE["<b>DriftEngine</b><br/>5-signal drift scoring<br/>Graduated response<br/>Combines GoalDNA + PredictionEngine"]:::brain
            ADAPTIVE_BUDGET["<b>AdaptiveBudget</b><br/>Dynamic step/token budget<br/>Based on task complexity<br/>Feeds AgentLoop"]:::brain
            FEEDBACK_ENG["<b>FeedbackEngine</b><br/>4-Tier:<br/>T1: Direct (amygdala)<br/>T2: User Insights (hippocampus)<br/>T3: Enterprise (PFC)<br/>T4: Global (collective)"]:::brain
            PREDICTION_ENG["<b>PredictionEngine</b><br/>Predict-Compare-Surprise<br/>Bayesian surprise<br/>Dopamine error signal<br/>Feeds DriftEngine"]:::brain
            PLASTICITY_ENG["<b>PlasticityManager</b><br/>Hebbian Learning<br/>LTP / LTD<br/>Homeostatic regulation<br/>Critical periods<br/>Consolidation<br/>Metaplasticity"]:::brain
            ADAPTATION_ENG["<b>AdaptationFilter</b><br/>Sensory adaptation<br/>Habituation<br/>Signal fatigue"]:::brain
            POPULATION_ENG["<b>PopulationQualityEstimator</b><br/>Ensemble decisions<br/>Population coding<br/>Agreement scoring"]:::brain
        end

        subgraph BRAIN_ADVANCED ["Advanced Brain Components (P0-P3)"]
            direction TB
            PROACTIVE["<b>P0: ProactivePredictionEngine</b><br/>Markov chain trajectory<br/>Pre-warming scheduler<br/>Prediction chain cache"]:::brain
            CROSS_MODAL["<b>P1: CrossModalAssociator</b><br/>Hebbian binding<br/>Spreading activation<br/>LTD decay + pruning"]:::brain
            CALIBRATION["<b>P1: ContinuousCalibrationEngine</b><br/>ECE tracking<br/>Platt scaling<br/>MetaCognitionMonitor"]:::brain
            COLUMNS["<b>P2: ColumnManager</b><br/>Functional columns<br/>Winner-take-all competition<br/>Lateral inhibition<br/>Column recruitment"]:::brain
            RESOURCE_MAP["<b>P2: ResourceHomunculus</b><br/>Non-uniform allocation<br/>Task-type budgets<br/>Model tier selection"]:::brain
            ATTENTION["<b>P2: AttentionSystem</b><br/>Subconscious processing<br/>Change detection<br/>Priority classification"]:::brain
            CONCEPTS["<b>P3: ConceptGraphManager</b><br/>Distributed representations<br/>Spreading activation<br/>Concept formation<br/>Auto-discovery"]:::brain
            REORGANIZER["<b>P3: CorticalMapReorganizer</b><br/>Territory merging<br/>Redistribution<br/>Use-it-or-lose-it"]:::brain
            MODULATOR["<b>P3: TargetedModulator</b><br/>Optogenetic activation<br/>Silencing overrides<br/>CLAMP parameters"]:::brain
            SIMULATOR["<b>P3: ComponentSimulator</b><br/>Digital twin<br/>A/B testing<br/>What-if analysis<br/>Monte Carlo<br/>Session replay"]:::brain
        end
    end

    SESSION --> WEIGHTS
    SESSION --> GOAL_TRACK
    SESSION --> GOAL_DNA
    SESSION --> GOAL_REMINDER
    SESSION --> GOAL_TREE
    SESSION --> LOOP_DETECTOR
    SESSION --> DRIFT_ENGINE
    SESSION --> ADAPTIVE_BUDGET
    SESSION --> FEEDBACK_ENG
    SESSION --> PREDICTION_ENG
    SESSION --> PLASTICITY_ENG
    SESSION --> ADAPTATION_ENG
    SESSION --> POPULATION_ENG
    SESSION --> PROACTIVE
    SESSION --> CROSS_MODAL
    SESSION --> CALIBRATION
    SESSION --> COLUMNS
    SESSION --> RESOURCE_MAP
    SESSION --> ATTENTION
    SESSION --> CONCEPTS
    SESSION --> REORGANIZER
    SESSION --> MODULATOR
    SESSION --> SIMULATOR

    GOAL_DNA -->|"Complements"| GOAL_TRACK
    GOAL_REMINDER -->|"Injects reminders"| AGENT_LOOP
    GOAL_TREE -->|"Hierarchical plans"| PLANNER
    LOOP_DETECTOR -->|"Enhanced loop detection"| GOAL_TRACK
    DRIFT_ENGINE -->|"Drift signals from"| GOAL_DNA
    DRIFT_ENGINE -->|"Drift signals from"| PREDICTION_ENG
    ADAPTIVE_BUDGET -->|"Budget management"| AGENT_LOOP
    FEEDBACK_ENG -->|"Weight updates"| WEIGHTS
    PLASTICITY_ENG -->|"Hebbian/LTP/LTD"| WEIGHTS
    PREDICTION_ENG -->|"Surprise signal"| PLASTICITY_ENG
    GOAL_TRACK -->|"Drift score"| WEIGHTS

    %% ===================================================================
    %% E. BRAIN-LLM BRIDGE (3 Layers)
    %% ===================================================================
    subgraph BRAIN_BRIDGE ["<b>E. גשר מוח-LLM -- Brain-LLM Bridge</b>"]
        direction TB

        subgraph BRIDGE_L1 ["Layer 1: Prompt & Parameter Bridge"]
            direction LR
            BSI["<b>BrainStateInjector</b><br/>Thalamocortical annotation<br/>Brain state -> prompt text"]:::brain
            BPR["<b>BrainParameterResolver</b><br/>Cognitive signals -> API params<br/>temperature, top_p, top_k<br/>max_tokens, penalties"]:::brain
        end

        subgraph BRIDGE_L1_SUB ["Parameter Sub-Resolvers"]
            direction LR
            TEMP_RES["TemperatureResolver"]:::brain
            TOKEN_RES["TokenResolver"]:::brain
            PENALTY_RES["PenaltyResolver"]:::brain
            PARAM_HELP["ParameterHelpers"]:::brain
        end

        subgraph BRIDGE_L2 ["Layer 2: Inference Hooks & Training"]
            direction TB
            INF_HOOKS["<b>InferenceHookPipeline</b><br/>HebbianAccumulator<br/>AttentionHabituation<br/>AdaptiveHeadTemperature<br/>PopulationWeightedVoting"]:::brain
            NEURO_ADAPT["<b>NeuroAdapter</b><br/>LoRA framework<br/>Weight loading/merging<br/>NeuroscienceAdapterSpec"]:::brain
            TRAIN_COLL["<b>TrainingCollector</b><br/>SFT / DPO pairs<br/>Brain-conditioned data<br/>JSONL export"]:::brain
            EARLY_EXIT["<b>EarlyExit</b><br/>System 1/2 at model level<br/>Exit classifiers<br/>Adaptive computation"]:::brain
            STRUCT_OUT["<b>StructuredOutputInjector/Parser</b><br/>LLM self-assessment<br/>Confidence + Difficulty<br/>SignalAggregator"]:::brain
            CONTENT_PRED["<b>ContentPredictor</b><br/>LLM-powered prediction<br/>Tool / response / sentiment"]:::brain
        end
    end

    WEIGHTS -->|"Behavioral weights"| BSI
    COLUMNS -->|"Active column"| BSI
    GOAL_TRACK -->|"Goal state"| BSI
    ATTENTION -->|"Change highlights"| BSI
    CALIBRATION -->|"ECE alerts"| BSI
    PREDICTION_ENG -->|"Surprise context"| BSI
    CONCEPTS -->|"Active concepts"| BSI
    PROACTIVE -->|"Next predictions"| BSI

    BSI -->|"Brain digest prompt"| LLM_ROUTER
    BPR -->|"LLMParameterBundle"| LLM_ROUTER
    BPR --> TEMP_RES
    BPR --> TOKEN_RES
    BPR --> PENALTY_RES
    BPR --> PARAM_HELP

    PREDICTION_ENG -->|"Surprise"| BPR
    CALIBRATION -->|"Confidence"| BPR
    ATTENTION -->|"Priority"| BPR
    RESOURCE_MAP -->|"Token budget"| BPR
    COLUMNS -->|"Weight overrides"| BPR
    MODULATOR -->|"CLAMP overrides"| BPR

    %% ===================================================================
    %% F. NEUROLLAMA (Layer 3)
    %% ===================================================================
    subgraph NEUROLLAMA ["<b>F. NeuroLlama -- Layer 3 Architecture</b>"]
        direction TB
        NL_CONFIG["<b>NeuroLlamaConfig</b><br/>Master configuration"]:::nl
        NL_SYNAP["<b>SynapticAttention</b><br/>SynapticScaling<br/>NeuromodulatedAttention<br/>SynapticModulationMatrix"]:::nl
        NL_CORTICAL["<b>CorticalColumns</b><br/>CorticalColumnAttention<br/>LateralInhibition<br/>HierarchicalColumnOrganizer"]:::nl
        NL_PREDICT["<b>PredictiveCoding</b><br/>Prediction error minimization"]:::nl
        NL_HEBBIAN["<b>HebbianAttention</b><br/>Co-occurrence learning"]:::nl
        NL_HABIT["<b>HabituationLayer</b><br/>HabituatingAttention<br/>AttentionDecay<br/>NoveltyDetector"]:::nl
        NL_POP["<b>PopulationOutput</b><br/>ConfidenceWeightedVoting<br/>EntropyBasedVoting<br/>PopulationVectorDecoder"]:::nl
        NL_TRAIN["<b>TrainingObjectives</b><br/>Neuroscience loss functions"]:::nl
        NL_MODEL["<b>NeuroLlamaModel</b><br/>Full model integration"]:::nl
    end

    INF_HOOKS -->|"Hooks into"| NL_SYNAP
    INF_HOOKS -->|"Hooks into"| NL_HABIT
    INF_HOOKS -->|"Hooks into"| NL_POP
    NEURO_ADAPT -->|"LoRA weights"| NL_MODEL
    EARLY_EXIT -->|"Exit classifiers"| NL_MODEL
    NL_CONFIG --> NL_MODEL

    %% ===================================================================
    %% G. CONTEXT & MEMORY
    %% ===================================================================
    subgraph CONTEXT_MEMORY ["<b>G. הקשר וזיכרון -- Context & Memory</b>"]
        direction TB

        subgraph COGNITIVE_PKG ["corteX/engine/cognitive/ Package"]
            direction TB
            COG_PIPELINE["<b>CognitiveContextPipeline</b><br/>8-phase unified pipeline<br/>Replaces dual ContextCompiler + CCE<br/>Phases: Extract→Filter→Summarize<br/>→Score→Rank→Window→Inject→Cache"]:::mem
            CTX_QUALITY["<b>ContextQualityEngine</b><br/>6-dimensional scoring<br/>Goal Retention (GRS) | Information Density (IDI)<br/>Entanglement Completeness (EC) | Temporal Coherence (TC)<br/>Decision Preservation (DPR) | Anti-Hallucination (AHS)"]:::mem
            STATE_FILE_MGR["<b>StateFileManager</b><br/>3-layer compaction-proof state<br/>Crystallized | Fluid | Insights<br/>Persistent brain state"]:::mem
        end

        CCE["<b>CorticalContextEngine (CCE)</b><br/>10,000+ step management<br/>Hot/Warm/Cold hierarchy<br/>Progressive summarization<br/>L0-L3 compression"]:::mem
        CTX_SUMM["<b>ContextSummarizer</b><br/>L2: LLM summaries (fully executing)<br/>L3: Structured JSON digests<br/>Fed back to ContextCompiler"]:::mem
        SEM_SCORER["<b>SemanticImportanceScorer</b><br/>TF-IDF vector embeddings<br/>On-prem, numpy-only"]:::mem

        subgraph MEM_FABRIC ["Memory Fabric"]
            direction TB
            MEM_WORKING["<b>Working Memory</b><br/>Prefrontal cortex<br/>Current session state"]:::mem
            MEM_EPISODIC["<b>Episodic Memory</b><br/>Hippocampus<br/>Past experiences"]:::mem
            MEM_SEMANTIC["<b>Semantic Memory</b><br/>Neocortex<br/>Factual knowledge"]:::mem
            MEM_RETRIEVAL["<b>MemoryFabric.get_relevant_context()</b><br/>Called before every LLM call<br/>Periodic consolidation"]:::mem
        end

        MEM_MGR["<b>MemoryManager</b><br/>File + Vertex search drivers"]:::mem
    end

    SESSION --> COG_PIPELINE
    SESSION --> CTX_QUALITY
    SESSION --> STATE_FILE_MGR
    SESSION --> CCE
    SESSION --> MEM_WORKING
    SESSION --> MEM_EPISODIC
    SESSION --> MEM_SEMANTIC
    SESSION --> MEM_RETRIEVAL
    COG_PIPELINE -->|"Replaces"| CTX_COMPILER
    COG_PIPELINE -->|"Replaces"| CCE
    COG_PIPELINE -->|"Uses"| CTX_QUALITY
    STATE_FILE_MGR -->|"Persistent state"| MEM_FABRIC
    CCE --> CTX_SUMM
    CCE --> SEM_SCORER
    CTX_COMPILER -->|"zones"| CCE
    CTX_SUMM -->|"L2 LLM summaries"| CTX_COMPILER
    CTX_SUMM -->|"L3 JSON digests"| CTX_COMPILER
    CTX_SUMM -->|"Summarization via"| LLM_ROUTER
    MEM_RETRIEVAL -->|"Relevant context before every LLM call"| LLM_ROUTER
    MEM_RETRIEVAL -->|"Feeds"| CTX_COMPILER
    MEM_RETRIEVAL --> MEM_WORKING
    MEM_RETRIEVAL --> MEM_EPISODIC
    MEM_RETRIEVAL --> MEM_SEMANTIC
    CROSS_MODAL -->|"Enrichment"| MEM_EPISODIC
    CROSS_MODAL -->|"Enrichment"| MEM_SEMANTIC

    %% ===================================================================
    %% H. LLM PROVIDERS
    %% ===================================================================
    subgraph LLM_LAYER ["<b>H. ספקי LLM -- 4 Providers</b>"]
        direction TB
        LLM_ROUTER["<b>LLMRouter</b><br/>Thalamus: routes all requests<br/>Uses ModelRegistry + CognitiveClassifier<br/>Weight-based scoring<br/>Latency tracking<br/>Auto fallback<br/>CostTracker integration"]:::llm
        MODEL_REGISTRY["<b>ModelRegistry</b><br/>YAML model registry<br/>8 roles: orchestrator/worker/reasoner/<br/>coder/vision/embedding/critic/summarizer<br/>Hot-reload capability"]:::llm
        COGNITIVE_CLASSIFIER["<b>CognitiveClassifier</b><br/>Zero-LLM task classifier<br/>Pattern matching<br/>Feeds LLMRouter"]:::llm
        COST_TRACKER["<b>CostTracker</b><br/>Per-tenant cost tracking<br/>Budget enforcement<br/>Usage analytics"]:::llm
        MODEL_YAML["<b>cortex_models.yaml</b><br/>External model configuration<br/>Provider settings<br/>Role mappings"]:::llm
        LLM_BASE["<b>BaseLLMProvider</b><br/>Abstract interface<br/>generate() | stream()<br/>generate_structured()<br/>health_check()"]:::llm
        GEMINI_ADAPT["<b>GeminiAdapter</b><br/>Gemini 3 Pro/Flash<br/>Gemini 2.5 Pro/Flash"]:::llm
        OPENAI_PROV["<b>OpenAIProvider</b><br/>GPT-4o / GPT-4o-mini<br/>o1 / o3 reasoning"]:::llm
        ANTHROPIC_PROV["<b>AnthropicProvider</b><br/>Claude Opus 4.6 / Sonnet 4.5<br/>Haiku 4.5<br/>Extended thinking / Vision"]:::llm
        LOCAL_PROV["<b>Local Provider</b><br/>OpenAI-compatible<br/>vLLM / Ollama / etc."]:::llm
        GEMINI_CLIENT["<b>GeminiClient</b><br/>Direct Gemini API"]:::llm
        RETRY["<b>retry_with_backoff</b><br/>Exponential backoff<br/>Transient error retry"]:::llm
    end

    RUN -->|"LLM call"| LLM_ROUTER
    RUN_STREAM -->|"Stream call"| LLM_ROUTER
    MODEL_REGISTRY -->|"Model info"| LLM_ROUTER
    MODEL_YAML -->|"Config source"| MODEL_REGISTRY
    COGNITIVE_CLASSIFIER -->|"Task classification"| LLM_ROUTER
    COST_TRACKER -->|"Budget check"| LLM_ROUTER
    LLM_ROUTER -->|"Track costs"| COST_TRACKER
    LLM_ROUTER --> GEMINI_ADAPT
    LLM_ROUTER --> OPENAI_PROV
    LLM_ROUTER --> ANTHROPIC_PROV
    LLM_ROUTER --> LOCAL_PROV
    LLM_BASE --> GEMINI_ADAPT
    LLM_BASE --> OPENAI_PROV
    LLM_BASE --> ANTHROPIC_PROV
    LLM_BASE --> LOCAL_PROV
    GEMINI_ADAPT --> GEMINI_CLIENT
    GEMINI_ADAPT --> RETRY
    OPENAI_PROV --> RETRY
    ANTHROPIC_PROV --> RETRY

    %% ===================================================================
    %% I. GAME THEORY & ROUTING
    %% ===================================================================
    subgraph GAME_THEORY ["<b>I. תורת משחקים -- Game Theory</b>"]
        direction TB
        DUAL_PROC["<b>DualProcessRouter</b><br/>System 1: fast/cached<br/>System 2: slow/deliberate"]:::brain
        REPUTATION["<b>ReputationSystem</b><br/>Modified Tit-for-Tat<br/>Tool trust + quarantine"]:::brain
        NASH_OPT["<b>NashRoutingOptimizer</b><br/>Stable model-task routing<br/>Best-response dynamics"]:::brain
        SHAPLEY["<b>ShapleyAttributor</b><br/>Fair multi-tool credit<br/>Coalition value computation"]:::brain
        MINIMAX["<b>MinimaxSafetyGuard</b><br/>Risk minimization<br/>Enterprise decisions"]:::brain
        TRUTH_SCORE["<b>TruthfulScoringMechanism</b><br/>Incentive-compatible<br/>Capability scoring"]:::brain
        NASH_BRIDGE["<b>NashRoutingBridge</b><br/>SDK pipeline integration"]:::brain
        SHAPLEY_BRIDGE["<b>ShapleyAttributionBridge</b><br/>SDK pipeline integration"]:::brain
    end

    SESSION --> DUAL_PROC
    SESSION --> REPUTATION
    SESSION --> NASH_BRIDGE
    SESSION --> SHAPLEY_BRIDGE
    NASH_BRIDGE --> NASH_OPT
    SHAPLEY_BRIDGE --> SHAPLEY
    DUAL_PROC -->|"System 1/2 decision"| LLM_ROUTER
    NASH_OPT -->|"Optimal routing"| WEIGHTS

    %% ===================================================================
    %% J. MATHEMATICAL FOUNDATIONS
    %% ===================================================================
    subgraph MATH_FOUND ["<b>J. יסודות מתמטיים -- Mathematical Foundations</b>"]
        direction TB

        subgraph BAYESIAN ["Bayesian Statistics"]
            direction TB
            BETA_DIST["<b>BetaDistribution</b><br/>Binary success/failure<br/>Conjugate prior"]:::math
            GAMMA_DIST["<b>GammaDistribution</b><br/>Latency modeling<br/>Conjugate prior"]:::math
            NORMAL_UPD["<b>NormalNormalUpdater</b><br/>Quality score tracking"]:::math
            DIRICHLET["<b>DirichletMultinomialUpdater</b><br/>Categorical choice"]:::math
            BAYES_SURPRISE["<b>BayesianSurpriseCalculator</b><br/>KL divergence surprise"]:::math
            PROSPECT["<b>ProspectTheoreticUpdater</b><br/>Kahneman-Tversky<br/>Asymmetric loss"]:::math
            THOMPSON["<b>BayesianToolSelector</b><br/>Thompson Sampling<br/>Explore vs exploit"]:::math
            UCB1["<b>UCB1Selector</b><br/>Deterministic alternative"]:::math
            ANCHOR["<b>AnchorManager</b><br/>Informed initialization"]:::math
            AVAIL["<b>AvailabilityFilter</b><br/>Controlled recency bias"]:::math
        end

        subgraph GAME_MATH ["Game Theory Math"]
            direction TB
            NASH_MATH["Nash Equilibrium<br/>Best-response dynamics"]:::math
            SHAPLEY_MATH["Shapley Values<br/>Coalition attribution"]:::math
            MINIMAX_MATH["Minimax Theorem<br/>Worst-case optimization"]:::math
            TFT["Tit-for-Tat<br/>Cooperation dynamics"]:::math
        end

        SEM_SCORE_MATH["<b>SemanticScorer</b><br/>TF-IDF vectors<br/>Cosine similarity"]:::math
    end

    WEIGHTS --> BETA_DIST
    WEIGHTS --> GAMMA_DIST
    WEIGHTS --> PROSPECT
    WEIGHTS --> THOMPSON
    WEIGHTS --> ANCHOR
    WEIGHTS --> AVAIL
    WEIGHTS --> BAYES_SURPRISE
    PREDICTION_ENG --> BAYES_SURPRISE
    PROACTIVE --> BETA_DIST
    PROACTIVE --> GAMMA_DIST
    COLUMNS --> BETA_DIST
    NASH_OPT --> NASH_MATH
    SHAPLEY --> SHAPLEY_MATH
    MINIMAX --> MINIMAX_MATH
    REPUTATION --> TFT

    %% ===================================================================
    %% K. TOOLS & PLUGINS
    %% ===================================================================
    subgraph TOOLS_PLUGINS ["<b>K. כלים ותוספים -- Tools & Plugins</b>"]
        direction TB

        subgraph TOOL_FW ["Tool Framework"]
            direction TB
            TOOL_WRAP["<b>ToolWrapper</b><br/>Decorator-based registration<br/>Auto JSON Schema"]:::tool
            TOOL_EXEC["<b>ToolExecutor</b><br/>Safe execution<br/>Timeout + error handling"]:::tool
            BUILTIN_TOOLS["<b>Builtin Tools</b><br/>Default tool library"]:::tool
        end

        subgraph PLUGIN_SYS ["Plugin Subsystems"]
            direction TB
            CODE_INTERP["<b>CodeInterpreter</b><br/>Safe code execution"]:::tool
            CODE_SANDBOX["<b>CodeSandbox</b><br/>Isolated execution"]:::tool
            BROWSER_V2["<b>WebEngineV2</b><br/>Playwright browser<br/>LLM planning + self-heal"]:::tool
            COMPUTER_USE["<b>ComputerUse</b><br/>Desktop automation"]:::tool
        end

        subgraph AGENT_PLUGINS ["Agent Plugins"]
            direction TB
            SUPERVISOR["<b>AgentSupervisor</b><br/>Multi-agent coordination"]:::tool
            AGENT_MGR["<b>AgentManager</b><br/>Agent lifecycle"]:::tool
        end
    end

    TOOL_DECORATOR --> TOOL_WRAP
    SESSION -->|"Tool calls"| TOOL_EXEC
    TOOL_EXEC --> CODE_INTERP
    TOOL_EXEC --> CODE_SANDBOX
    TOOL_EXEC --> BROWSER_V2
    TOOL_EXEC --> COMPUTER_USE
    REPUTATION -->|"Trust filtering"| TOOL_EXEC
    POLICY_ENG -->|"Approval check"| TOOL_EXEC

    %% ===================================================================
    %% L. ENTERPRISE & SECURITY
    %% ===================================================================
    subgraph ENTERPRISE ["<b>L. ארגוני ואבטחה -- Enterprise & Security</b>"]
        direction TB
        LICENSE["<b>LicenseManager</b><br/>Ed25519 validation<br/>Per-seat licensing<br/>Offline grace period"]:::ent
        SAFETY["<b>SafetyPolicy</b><br/>Content filtering<br/>Input validation<br/>Blocked topics"]:::ent
        TENANT["<b>TenantConfig</b><br/>Per-tenant isolation<br/>Compliance frameworks<br/>GDPR | SOC2 | HIPAA"]:::ent
        ENT_CONFIG["<b>EnterpriseConfig</b><br/>Safety level<br/>Data retention<br/>Audit logging"]:::ent
        UPDATES["<b>UpdatesManager</b><br/>Version checking"]:::ent

        subgraph TENANCY_PKG ["corteX/tenancy/ Package"]
            direction TB
            TENANT_CTX["<b>TenantContext</b><br/>contextvars-based isolation<br/>3-layer model support<br/>Ensures instance-level state"]:::ent
        end
    end

    ENGINE -->|"License check"| LICENSE
    SESSION -->|"Policy enforcement"| SAFETY
    SESSION -->|"Tenant rules"| TENANT
    SESSION -->|"Tenant isolation"| TENANT_CTX
    CONFIG -->|"Enterprise settings"| ENT_CONFIG
    TENANT --> SAFETY
    TENANT_CTX -->|"Isolates"| EVENTS
    TENANT_CTX -->|"Isolates"| TOOL_WRAP
    TENANT_CTX -->|"Isolates"| MEM_MGR

    %% ===================================================================
    %% M. SERVER & INTEGRATIONS
    %% ===================================================================
    subgraph SERVER_INT ["<b>M. שרת ואינטגרציות -- Server & Integrations</b>"]
        direction TB
        FASTAPI["<b>FastAPI Server</b><br/>REST API<br/>Health endpoint"]:::sdk
        HEALTH["<b>Health Check</b><br/>Readiness probe"]:::sdk

        subgraph LANGFLOW ["Langflow Integration"]
            direction TB
            LF_AGENT["AgentComponent"]:::tool
            LF_ORCH["OrchestratorComponent"]:::tool
            LF_CODE["CodeInterpreterComponent"]:::tool
            LF_CU["ComputerUseComponent"]:::tool
            LF_SUP["SupervisorComponent"]:::tool
            LF_UI["UIComponent"]:::tool
        end
    end

    ENGINE --> FASTAPI
    FASTAPI --> HEALTH

    %% ===================================================================
    %% N. RUNTIME ORCHESTRATOR
    %% ===================================================================
    subgraph RUNTIME ["<b>N. אורקסטרטור -- Runtime Layer</b>"]
        direction TB
        ORCHESTRATOR["<b>Orchestrator</b><br/>Central Nervous System<br/>Autonomy scoring<br/>Safety enforcement<br/>Context enrichment<br/>Decision routing"]:::sdk
        AUTO_DECISION["<b>AutonomyDecision</b><br/>AUTONOMOUS | TIMER<br/>BLOCKING | BLOCKED"]:::sdk
    end

    SESSION --> ORCHESTRATOR
    ORCHESTRATOR --> WEIGHTS
    ORCHESTRATOR --> SAFETY
    ORCHESTRATOR --> MEM_WORKING
    ORCHESTRATOR --> POPULATION_ENG
    ORCHESTRATOR --> AUTO_DECISION

    %% ===================================================================
    %% O. STORAGE
    %% ===================================================================
    subgraph STORAGE ["<b>O. אחסון -- Storage</b>"]
        direction TB
        ARTIFACTS["<b>ArtifactStorage</b><br/>File-based persistence"]:::mem
    end

    MEM_EPISODIC --> ARTIFACTS
    TRAIN_COLL -->|"JSONL files"| ARTIFACTS

    %% ===================================================================
    %% P. CORE CONTRACTS & INFRASTRUCTURE
    %% ===================================================================
    subgraph CORE_INFRA ["<b>P. תשתית ליבה -- Core Infrastructure</b>"]
        direction TB
        CONTRACTS["<b>Contracts</b><br/>ISubsystem interface<br/>ContextSlice | Artifact"]:::ent
        EVENTS["<b>EventBus</b><br/>Instance-level Pub/Sub<br/>Tenant-isolated"]:::ent
        TOOL_REGISTRY["<b>ToolRegistry</b><br/>Per-instance registration<br/>Tenant-isolated"]:::ent
        REGISTRY["<b>PluginRegistry</b><br/>Decorator registration<br/>Discovery"]:::ent
        LIFECYCLE["<b>Lifecycle</b><br/>Init | Start | Stop"]:::ent
    end

    CONTRACTS --> CODE_INTERP
    CONTRACTS --> BROWSER_V2
    CONTRACTS --> COMPUTER_USE
    REGISTRY --> CODE_INTERP
    REGISTRY --> BROWSER_V2
    REGISTRY --> COMPUTER_USE

    %% ===================================================================
    %% DATA FLOW ARROWS (Complete Pipeline)
    %% ===================================================================

    %% Main execution flow (all generate() calls pass full 7-param brain bundle)
    RUN -->|"0. ContextCompiler.compile()"| CTX_COMPILER
    RUN -->|"0. MemoryFabric.get_relevant_context()"| MEM_RETRIEVAL
    RUN -->|"1. Feedback processing"| FEEDBACK_ENG
    RUN -->|"2. Goal tracking"| GOAL_TRACK
    RUN -->|"3. Attention filter"| ATTENTION
    RUN -->|"3. Column selection"| COLUMNS
    RUN -->|"3. Resource allocation"| RESOURCE_MAP
    RUN -->|"3. Concept activation"| CONCEPTS
    RUN -->|"3. Cross-modal enrichment"| CROSS_MODAL
    RUN -->|"4. Reactive prediction"| PREDICTION_ENG
    RUN -->|"4. Proactive prediction"| PROACTIVE
    RUN -->|"5. Dual-process routing"| DUAL_PROC
    RUN -->|"5. Targeted modulation"| MODULATOR
    RUN -->|"6. Brain state injection"| BSI
    RUN -->|"6. Parameter resolution (7-param bundle)"| BPR
    RUN -->|"7. LLM generation (7-param bundle)"| LLM_ROUTER
    RUN -->|"8. Tool execution"| TOOL_EXEC
    RUN -->|"9. Quality estimation"| POPULATION_ENG
    RUN -->|"9. Surprise comparison"| PREDICTION_ENG
    RUN -->|"10. Plasticity update"| PLASTICITY_ENG
    RUN -->|"10. Memory consolidation"| MEM_WORKING

    %% Learning loop
    POPULATION_ENG -->|"Quality signal"| PLASTICITY_ENG
    PREDICTION_ENG -->|"Surprise signal"| PLASTICITY_ENG
    PLASTICITY_ENG -->|"LTP/LTD/Hebbian"| WEIGHTS
    CALIBRATION -->|"Calibrated confidence"| BPR
    CALIBRATION -->|"Platt scaling"| PREDICTION_ENG
    REORGANIZER -->|"Territory updates"| COLUMNS
    SIMULATOR -->|"What-if results"| WEIGHTS

    %% Structured signals
    STRUCT_OUT -->|"Inject instruction"| LLM_ROUTER
    LLM_ROUTER -->|"LLM response"| STRUCT_OUT
    STRUCT_OUT -->|"Parsed signals"| CALIBRATION
```

---

## Component Count Summary

> **6,200+ tests passing** across 33+ test files

| Layer | Module Count | Key Files |
|-------|-------------|-----------|
| **SDK Core** | 3 classes | `sdk.py` (Engine, Agent, Session) |
| **Agentic Engine** | 7 modules | `agent_loop.py`, `context_compiler.py`, `planner.py`, `reflection.py`, `recovery.py`, `interaction.py`, `policy_engine.py`, `sub_agent.py` |
| **Brain Core** | 14 engines | `weights.py`, `goal_tracker.py`, `goal_dna.py`, `goal_reminder.py`, `goal_tree.py`, `loop_detector.py`, `drift_engine.py`, `adaptive_budget.py`, `feedback.py`, `prediction.py`, `plasticity.py`, `adaptation.py`, `population.py` |
| **Brain Advanced (P0-P3)** | 10 modules | `proactive.py`, `cross_modal.py`, `calibration.py`, `columns.py`, `resource_map.py`, `attention.py`, `concepts.py`, `reorganization.py`, `modulator.py`, `simulator.py` (+ 6 sub-modules) |
| **Brain-LLM Bridge L1** | 6 files | `brain_state_injector.py`, `parameter_resolver.py`, `temperature_resolver.py`, `token_resolver.py`, `penalty_resolver.py`, `param_helpers.py` |
| **Brain-LLM Bridge L2** | 6 modules | `inference_hooks.py`, `neuro_adapter.py`, `training_collector.py`, `early_exit.py`, `structured_output.py`, `content_predictor.py` |
| **NeuroLlama (L3)** | 10 modules | `config.py`, `synaptic_attention.py`, `cortical_columns.py`, `predictive_coding.py`, `hebbian_attention.py`, `habituation_layer.py`, `population_output.py`, `training_objectives.py`, `model.py` |
| **Context & Memory** | 9 modules | `cognitive/pipeline.py`, `cognitive/quality.py`, `cognitive/state_file.py`, `context.py`, `context_summarizer.py`, `semantic_scorer.py`, `context_compiler.py`, `memory.py`, `memory_fabric.py` |
| **LLM Providers** | 11 files | `router.py`, `registry.py`, `classifier.py`, `cost_tracker.py`, `cortex_models.yaml`, `base.py`, `gemini_adapter.py`, `gemini_client.py`, `openai_client.py`, `anthropic_client.py`, `local_provider.py` |
| **Mathematical** | 3 files | `bayesian.py` (10 classes), `game_theory.py` (6 classes), `game_integration.py` |
| **Tools & Plugins** | 8 modules | `decorator.py`, `executor.py`, 4 plugin subsystems, `supervisor.py`, `manager.py` |
| **Enterprise** | 5 files | `licensing.py`, `config.py`, `updates.py`, `tenancy/context.py` |
| **Server** | 2 files | `main.py`, `health.py` |
| **Runtime** | 1 file | `orchestrator.py` |
| **Integrations** | 6 files | Langflow components |
| **TOTAL** | **~103+ modules** | Full production SDK |

## Data Flow: Developer Code to Brain Learning

> All 14 `generate()` call sites pass the full 7-parameter brain bundle consistently.

```
Developer Code
    |
    v
Engine.create_agent() --> Agent (stateless template)
    |
    v
Agent.start_session() --> Session (stateful, full brain)
    |         TenantContext ensures instance-level isolation (EventBus, ToolRegistry, ContextBroker)
    v
Session.run("message")  /  Session.run_agentic()  /  Session.run_stream()
    |
    +---> [0] CognitiveContextPipeline (8-phase unified pipeline: Extract→Filter→Summarize→Score→Rank→Window→Inject→Cache)
    |         OR legacy ContextCompiler.compile() assembles 4-zone context (ALL modes: chat + agentic)
    +---> [0] ContextQualityEngine scores context on 6 dimensions (GRS, IDI, EC, TC, DPR, AHS)
    +---> [0] MemoryFabric.get_relevant_context() retrieves relevant memories
    |         Working + Episodic + Semantic memory queried before every LLM call
    +---> [0] StateFileManager maintains 3-layer compaction-proof state (Crystallized, Fluid, Insights)
    +---> [0] Periodic memory consolidation runs in background
    +---> [1] FeedbackEngine detects implicit signals from user text
    +---> [2] GoalTracker initializes / verifies alignment
    +---> [2] GoalDNA creates token fingerprint for O(1) drift detection
    +---> [2] GoalTree builds hierarchical Goal→SubGoals→Steps structure
    +---> [2] MultiResolutionLoopDetector runs 4 parallel detectors (State hash, Semantic, Action, Temporal)
    +---> [2] DriftEngine scores drift using 5 signals (GoalDNA, PredictionEngine, etc.) + graduated response
    +---> [2] AdaptiveBudget calculates dynamic step/token budget based on task complexity
    +---> [2] GoalReminderInjector injects adaptive goal reminders per LLM call
    +---> [3] AttentionSystem + ColumnManager + ResourceHomunculus classify & allocate
    +---> [3] ConceptGraph activates distributed representations
    +---> [3] CrossModalAssociator enriches from learned associations
    +---> [4] PredictionEngine predicts outcome (reactive)
    +---> [4] ProactivePredictionEngine predicts user's NEXT action
    +---> [5] DualProcessRouter selects System 1 (fast) or System 2 (slow)
    +---> [5] TargetedModulator applies optogenetic overrides
    +---> [6] BrainStateInjector compiles brain state into LLM prompt
    +---> [6] BrainParameterResolver maps cognitive signals to full 7-param bundle
    |         (temperature, top_p, top_k, max_tokens, penalties, thinking_budget, stop_sequences)
    +---> [7] ModelRegistry provides 8-role model catalog (orchestrator, worker, reasoner, coder, vision, embedding, critic, summarizer)
    +---> [7] CognitiveClassifier classifies task type (zero-LLM heuristic)
    +---> [7] CostTracker checks budget and tracks per-tenant costs
    +---> [7] LLMRouter routes to best provider (OpenAI / Gemini / Anthropic / Local)
    |         with Nash-optimal model selection -- 4 providers fully supported
    +---> [8] ToolExecutor runs tool calls (reputation-gated)
    |         run_stream() supports tool execution loop with StreamChunk(model, chunk_type)
    +---> [8] SubAgentManager creates + executes sub-agent tasks with isolated context
    |         Delegation wired: parent delegates, child runs, results summarized back
    +---> [9] PopulationQualityEstimator scores response quality
    +---> [9] PredictionEngine compares actual vs predicted (Bayesian surprise)
    +---> [9] StructuredOutputParser extracts LLM self-assessed confidence
    +---> [10] PlasticityManager updates weights (Hebbian + LTP/LTD)
    +---> [10] CorticalContextEngine manages context window
    |          L2: LLM summaries fully executing
    |          L3: Structured JSON digests fully executing
    |          L2/L3 fed back into ContextCompiler for next cycle
    +---> [10] CalibrationEngine tracks prediction accuracy (ECE)
    +---> [10] ShapleyAttributor distributes credit across tools
    |
    v
Response (content + metadata + artifacts)
    |
    v
Brain has LEARNED: weights updated, patterns consolidated, predictions calibrated
Memory consolidated: L2/L3 summaries generated, relevant context cached for next call
StateFileManager persists crystallized knowledge across compactions
```
