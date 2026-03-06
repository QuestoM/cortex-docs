# Model Routing Architecture Research Report

**Date:** 2026-02-15
**Author:** Senior AI Engineer Research
**Status:** Research Complete -- Awaiting Implementation Decision

---

## Executive Summary

This report analyzes how corteX should intelligently route different tasks to different LLM models, with an updatable model registry that decouples model knowledge from SDK code. The research covers role taxonomy, dynamic selection algorithms, registry architecture, cost optimization, multi-tenancy, and future-proofing.

**Key findings:**

1. The current router has solid foundations (circuit breaker, rate limiter, weight-based scoring, task classification) but lacks an externalized model registry -- all model metadata is hardcoded inside provider classes.
2. Industry best practices show that intelligent routing can reduce costs by **50-85%** while maintaining 95%+ quality, using a tiered approach where cheap models handle simple tasks and expensive models handle complex ones.
3. The model landscape is evolving rapidly (GPT-5.x, Claude Opus 4.6, Gemini 3, DeepSeek V3.2, Llama 4, Kimi K2.5), making an externalized, hot-reloadable registry essential.
4. corteX's brain-inspired architecture (attention system, dual-process routing, resource homunculus) already provides most of the signals needed for intelligent model selection -- the routing layer just needs to consume them more explicitly.

---

## 1. Model Role Taxonomy

### 1.1 Current State in corteX

The SDK currently recognizes only **two roles**: `orchestrator` and `worker`. These are set globally on the `Engine` and selected per-turn based on brain signals (attention priority, dual-process type, resource allocation, column recommendations, concept recommendations).

From `sdk.py` lines 2996-3014, the role selection logic:

```python
if ctx.attention_priority == AttentionalPriority.SUBCONSCIOUS:
    ctx.role = "worker"
elif ctx.attention_priority == AttentionalPriority.CRITICAL:
    ctx.role = "orchestrator"
elif ctx.process_type == ProcessType.SYSTEM2:
    ctx.role = "orchestrator"
elif ctx.resource_alloc is not None and ctx.resource_alloc.model_tier == "orchestrator":
    ctx.role = "orchestrator"
elif concept_model_tier == "orchestrator":
    ctx.role = "orchestrator"
else:
    ctx.role = ctx.column_recs.get("model_tier", "worker")
```

### 1.2 Recommended Role Taxonomy

An agentic system should recognize **8 distinct roles**, each with specific model requirements:

| Role | Purpose | Key Requirements | Ideal Model Class | Cost Sensitivity |
|------|---------|-----------------|-------------------|------------------|
| **Orchestrator** | Complex reasoning, planning, goal decomposition, multi-step strategy | Max intelligence, deep reasoning (thinking/CoT), strong instruction following | Frontier (Opus 4.6, GPT-5, Gemini 3 Pro) | Low -- quality critical |
| **Worker** | Fast execution: tool calls, code generation, simple responses | Speed, good tool calling, adequate instruction following | Mid-tier fast (Sonnet 4.5, GPT-4o, Gemini 3 Flash) | High -- high volume |
| **Summarizer** | Context compression, memory consolidation, conversation digest | Long context window, faithfulness, concision | Mid-tier with long context (Gemini 3 Flash 1M, GPT-4o 128K) | High -- frequent ops |
| **Judge** | Quality verification, reflection, output validation, safety checks | Strong reasoning, calibration, low hallucination rate | Frontier or mid-tier with thinking (Opus 4.6, o3, Gemini 3 Pro) | Medium -- selective use |
| **Embedder** | Semantic search, memory retrieval, similarity scoring | Not a generative model; specialized embedding model | text-embedding-3-large, Gemini embedding, local e5-large-v2 | Very high -- massive volume |
| **Classifier** | Intent detection, task routing, complexity estimation, topic detection | Speed, good classification accuracy, structured output | Small/fast (Haiku 4.5, GPT-4o-mini, Gemini 2.5 Flash-Lite) | Very high -- every request |
| **Coder** | Code generation, debugging, refactoring, code review | Strong coding benchmarks (SWE-bench, HumanEval), code-specific training | Code-specialized (GPT-5.2 Codex, Opus 4.6, Gemini 3 Pro) | Medium -- specialized tasks |
| **Creative** | Open-ended generation, brainstorming, content writing | Temperature flexibility, diverse outputs, natural language quality | Mid-tier to frontier, higher temperature | Medium |

### 1.3 Mapping to corteX Brain Signals

The existing brain-inspired systems already produce signals that map well to these roles:

| Brain Signal | Source Module | Maps to Role |
|-------------|--------------|--------------|
| `AttentionalPriority.CRITICAL` | `AttentionSystem` | Orchestrator |
| `AttentionalPriority.SUBCONSCIOUS` | `AttentionSystem` | Worker / Classifier |
| `ProcessType.SYSTEM2` | `DualProcessRouter` | Orchestrator / Judge |
| `ProcessType.SYSTEM1` | `DualProcessRouter` | Worker / Classifier |
| `resource_alloc.model_tier` | `ResourceHomunculus` | Direct mapping |
| `TaskType.VALIDATION` | `LLMRouter._classify_task` | Judge |
| `TaskType.SUMMARIZATION` | `LLMRouter._classify_task` | Summarizer |
| `TaskType.CODING` | `LLMRouter._classify_task` | Coder |
| `TaskType.PLANNING` | `LLMRouter._classify_task` | Orchestrator |
| `TaskType.CONVERSATION` | `LLMRouter._classify_task` | Worker / Creative |
| `concept_recs.suggested_model_tier` | `ConceptGraphManager` | Direct mapping |

### 1.4 Recommendation

Expand from 2 roles to **6 generative roles** (orchestrator, worker, summarizer, judge, coder, creative) plus **2 non-generative roles** (embedder, classifier). The router should accept a `role` parameter and resolve it to a specific model using the registry. The existing brain signals should drive role selection more granularly than the current binary orchestrator/worker split.

---

## 2. Dynamic Model Selection

### 2.1 Task Complexity Detection

The current `_classify_task()` method uses keyword heuristics, which is brittle. Industry approaches for complexity detection:

**Approach 1: Lightweight Classifier (Recommended for corteX)**
- Use the cheapest available model (Haiku/GPT-4o-mini/Gemini Flash-Lite) to classify task complexity as LOW / MEDIUM / HIGH / CRITICAL.
- Cost: ~$0.0001 per classification (negligible).
- Latency: ~100-200ms.
- This is what RouteLLM and similar systems do.

**Approach 2: Embedding-Based Routing (Future Enhancement)**
- Convert the prompt to an embedding vector.
- Compare against reference vectors for each task type.
- Route to the model associated with the closest cluster.
- Faster than Approach 1 (no LLM call), but requires maintaining reference embeddings.

**Approach 3: Heuristic + Metadata (Current Approach, Enhanced)**
- Keep keyword-based classification but augment with:
  - Message length (longer = more complex)
  - Tool count (more tools = more complex)
  - Conversation depth (deeper = may need stronger model)
  - Previous step results (errors escalate complexity)
  - Explicit user/developer hints

**Recommended Hybrid:**
```
COMPLEXITY = weighted_sum(
    keyword_heuristic_score,           # Current _classify_task()
    message_length_score,              # Normalized token count
    tool_count_score,                  # Number of available tools
    conversation_depth_score,          # Turn number in session
    error_escalation_score,            # Errors in last N turns
    attention_priority_score,          # From brain's attention system
    dual_process_score,                # System1 vs System2
    resource_homunculus_score,         # From resource allocation
    explicit_hint_score,              # Developer override
)

if COMPLEXITY < 0.3:   role = "worker"
elif COMPLEXITY < 0.6: role = "worker" (with mid-tier model)
elif COMPLEXITY < 0.8: role = "orchestrator"
else:                  role = "orchestrator" (with frontier model + thinking)
```

### 2.2 How Claude Code Handles Model Selection

Claude Code provides an excellent reference implementation for role-based model routing:

- **Default:** Uses Claude Sonnet 4.5 for most tasks (balanced cost/quality).
- **Fast mode (`/fast`):** Same model but optimized for speed (faster output, premium pricing).
- **Opus mode:** Users can switch to Opus 4.6 for complex reasoning tasks.
- **OpusPlan:** A hybrid workflow where Opus handles planning and Sonnet handles execution -- essentially an orchestrator/worker split within a single task.

This validates corteX's orchestrator/worker model but adds the crucial insight that **the split can happen within a single user task** (plan with expensive model, execute with cheap model).

### 2.3 How Cursor Uses Different Models

Cursor (AI code editor) uses a similar multi-model approach:

- Fast editing: Uses smaller, faster models for code completion and inline edits.
- Complex tasks: Routes to stronger models for multi-file refactoring, architecture decisions.
- Background indexing: Uses embedding models for codebase search.
- Users can select models per-task.

### 2.4 Automatic Cost Optimization

The system should **automatically pick the cheapest model that can handle the task**:

```
Algorithm: CostAwareRouting(task, budget_remaining)
    1. Classify task complexity -> LOW / MEDIUM / HIGH / CRITICAL
    2. Filter models by:
       a. Required capabilities (tool_calling, vision, thinking, etc.)
       b. Minimum intelligence tier for the complexity level
       c. Provider availability (circuit breaker state)
       d. Rate limit headroom
    3. Among qualifying models, sort by:
       a. Primary: estimated cost (input + output tokens * price)
       b. Secondary: latency (prefer faster at same cost)
       c. Tertiary: historical success rate for this task type
    4. Select top candidate
    5. If budget_remaining is low, force downgrade to cheapest qualifying model
    6. Log decision for cost tracking
```

### 2.5 Fallback Strategy

The current router already has basic fallback (try alternatives on failure). Recommended enhancement:

```
Fallback Chain (per role):
    orchestrator: Opus 4.6 -> GPT-5 -> Gemini 3 Pro -> Sonnet 4.5 (degraded)
    worker:       Flash -> GPT-4o-mini -> Haiku 4.5 -> Gemini 2.5 Flash-Lite
    summarizer:   Flash -> GPT-4o-mini -> Haiku 4.5
    judge:        Opus 4.6 -> GPT-5 -> Gemini 3 Pro
    coder:        Opus 4.6 -> GPT-5.2 Codex -> Gemini 3 Pro -> Sonnet 4.5
    embedder:     text-embedding-3-large -> Gemini embedding -> local e5
    classifier:   Haiku 4.5 -> GPT-4o-mini -> Flash-Lite -> keyword heuristic

Fallback triggers:
    - API error (non-auth)
    - Rate limit (429)
    - Circuit breaker open
    - Timeout (>30s for fast roles, >120s for orchestrator)
    - Context length exceeded (auto-truncate or use longer-context model)
```

---

## 3. Updatable Model Registry

### 3.1 Current Problem

All model metadata is **hardcoded inside provider classes**:

- `openai_client.py` lines 135-182: Hardcoded list of `ModelInfo` for gpt-4o, gpt-4o-mini, o3.
- `gemini_adapter.py` lines 165-236: Hardcoded list for Gemini 3 Pro/Flash, Gemini 2.5 Pro/Flash.
- `anthropic_client.py` lines 146-153: Hardcoded list for Opus 4.6, Sonnet 4.5, Haiku 4.5.

When GPT-5.3 was released (February 2026), or when Gemini 2.0 models get deprecated (March 31, 2026), users must **update the SDK code** to change model availability. This is unacceptable for production systems.

### 3.2 Recommended Registry Architecture

The registry should be a **layered system** with three levels:

```
Layer 1: Built-in defaults (in code, ships with SDK)
    |
    v
Layer 2: External config file (YAML, loaded at startup)
    |
    v
Layer 3: Runtime overrides (API calls, per-tenant config)
```

#### 3.2.1 Registry Schema (YAML)

```yaml
# cortex_models.yaml -- Model Registry Configuration
# This file can be updated independently of the SDK.
# Place in: ~/.cortex/models.yaml, ./cortex_models.yaml, or set CORTEX_MODEL_REGISTRY env var.

version: "2.0"
last_updated: "2026-02-15"

# ============================================================================
# MODEL DEFINITIONS
# ============================================================================
models:

  # --- OpenAI Models ---
  gpt-5:
    provider: openai
    model_id: "gpt-5"
    display_name: "GPT-5"
    capabilities: [text, vision, tool_calling, structured_output, streaming]
    context_window: 400000
    max_output_tokens: 32768
    cost_per_million_input: 1.25
    cost_per_million_output: 10.00
    cost_per_million_cached_input: 0.125    # 90% cache discount
    speed_tier: medium       # fast | medium | slow
    intelligence_tier: frontier  # low | medium | high | frontier
    supported_roles: [orchestrator, worker, coder, judge, creative]
    recommended_temperature:
      planning: 0.7
      coding: 0.3
      validation: 0.1
      conversation: 0.8
      tool_use: 0.2
    status: ga               # preview | ga | deprecated
    deprecation_date: null
    notes: "Flagship model. 400K context. Strong at coding and agentic tasks."

  gpt-5-mini:
    provider: openai
    model_id: "gpt-5-mini"
    display_name: "GPT-5 Mini"
    capabilities: [text, vision, tool_calling, structured_output, streaming]
    context_window: 400000
    max_output_tokens: 16384
    cost_per_million_input: 0.30
    cost_per_million_output: 1.25
    speed_tier: fast
    intelligence_tier: medium
    supported_roles: [worker, summarizer, classifier]
    status: ga

  o3:
    provider: openai
    model_id: "o3"
    display_name: "OpenAI o3"
    capabilities: [text, tool_calling, thinking, streaming]
    context_window: 200000
    max_output_tokens: 100000
    cost_per_million_input: 2.00
    cost_per_million_output: 8.00
    speed_tier: slow
    intelligence_tier: frontier
    supported_roles: [orchestrator, judge, coder]
    notes: "Reasoning model. Uses internal reasoning tokens billed as output."
    status: ga

  o4-mini:
    provider: openai
    model_id: "o4-mini"
    display_name: "OpenAI o4-mini"
    capabilities: [text, tool_calling, thinking, streaming]
    context_window: 200000
    max_output_tokens: 65536
    cost_per_million_input: 0.40
    cost_per_million_output: 1.60
    speed_tier: fast
    intelligence_tier: high
    supported_roles: [worker, judge, coder]
    status: ga

  gpt-4o:
    provider: openai
    model_id: "gpt-4o"
    display_name: "GPT-4o"
    capabilities: [text, vision, tool_calling, structured_output, streaming]
    context_window: 128000
    max_output_tokens: 16384
    cost_per_million_input: 2.50
    cost_per_million_output: 10.00
    speed_tier: medium
    intelligence_tier: high
    supported_roles: [worker, coder, creative]
    status: ga
    notes: "Legacy model. Still widely used. Being superseded by GPT-5."

  gpt-4o-mini:
    provider: openai
    model_id: "gpt-4o-mini"
    display_name: "GPT-4o Mini"
    capabilities: [text, vision, tool_calling, streaming]
    context_window: 128000
    max_output_tokens: 16384
    cost_per_million_input: 0.15
    cost_per_million_output: 0.60
    speed_tier: fast
    intelligence_tier: medium
    supported_roles: [worker, summarizer, classifier]
    status: ga

  text-embedding-3-large:
    provider: openai
    model_id: "text-embedding-3-large"
    display_name: "OpenAI Embeddings Large"
    capabilities: [embeddings]
    context_window: 8191
    cost_per_million_input: 0.13
    cost_per_million_output: 0.0
    speed_tier: fast
    intelligence_tier: high       # For embeddings: quality tier
    supported_roles: [embedder]
    status: ga

  # --- Anthropic Claude Models ---
  claude-opus-4-6:
    provider: anthropic
    model_id: "claude-opus-4-6"
    display_name: "Claude Opus 4.6"
    capabilities: [text, vision, tool_calling, thinking, streaming]
    context_window: 200000
    max_output_tokens: 32768
    cost_per_million_input: 5.00
    cost_per_million_output: 25.00
    cost_per_million_cached_input: 0.50
    speed_tier: slow
    intelligence_tier: frontier
    supported_roles: [orchestrator, judge, coder, creative]
    recommended_temperature:
      planning: 0.8
      coding: 0.3
      validation: 0.1
      conversation: 1.0
      tool_use: 0.2
      reasoning: 0.7
    status: ga
    notes: "Most capable Claude. 1M context in extended mode. SWE-bench leader."

  claude-sonnet-4-5:
    provider: anthropic
    model_id: "claude-sonnet-4-5"
    display_name: "Claude Sonnet 4.5"
    capabilities: [text, vision, tool_calling, thinking, streaming]
    context_window: 200000
    max_output_tokens: 16384
    cost_per_million_input: 3.00
    cost_per_million_output: 15.00
    cost_per_million_cached_input: 0.30
    speed_tier: medium
    intelligence_tier: high
    supported_roles: [orchestrator, worker, coder, creative, judge]
    status: ga
    notes: "Best balance of cost and quality. Default for most tasks."

  claude-haiku-4-5:
    provider: anthropic
    model_id: "claude-haiku-4-5"
    display_name: "Claude Haiku 4.5"
    capabilities: [text, vision, tool_calling, streaming]
    context_window: 200000
    max_output_tokens: 8192
    cost_per_million_input: 1.00
    cost_per_million_output: 5.00
    cost_per_million_cached_input: 0.10
    speed_tier: fast
    intelligence_tier: medium
    supported_roles: [worker, summarizer, classifier]
    status: ga
    notes: "Fastest Claude. Good for high-volume, simpler tasks."

  # --- Google Gemini Models ---
  gemini-3-pro-preview:
    provider: gemini
    model_id: "gemini-3-pro-preview"
    display_name: "Gemini 3 Pro"
    capabilities: [text, vision, tool_calling, thinking, streaming, code_execution]
    context_window: 1000000
    max_output_tokens: 65536
    cost_per_million_input: 2.00
    cost_per_million_output: 12.00
    speed_tier: medium
    intelligence_tier: frontier
    supported_roles: [orchestrator, coder, judge, creative]
    recommended_temperature:
      default: 1.0     # Google recommends 1.0 for Gemini 3
      validation: 0.5
    status: preview
    notes: "1M context. Google recommends temperature=1.0."

  gemini-3-flash-preview:
    provider: gemini
    model_id: "gemini-3-flash-preview"
    display_name: "Gemini 3 Flash"
    capabilities: [text, vision, tool_calling, thinking, streaming, code_execution]
    context_window: 1000000
    max_output_tokens: 65536
    cost_per_million_input: 0.50
    cost_per_million_output: 3.00
    speed_tier: fast
    intelligence_tier: high
    supported_roles: [worker, summarizer, coder, creative]
    status: preview
    notes: "Extremely cost-effective. Best price/performance for most tasks."

  gemini-2.5-pro:
    provider: gemini
    model_id: "gemini-2.5-pro"
    display_name: "Gemini 2.5 Pro"
    capabilities: [text, vision, tool_calling, thinking, streaming, code_execution]
    context_window: 1000000
    max_output_tokens: 65536
    cost_per_million_input: 1.25
    cost_per_million_output: 10.00
    speed_tier: medium
    intelligence_tier: frontier
    supported_roles: [orchestrator, coder, judge]
    status: ga
    notes: "Stable fallback for Gemini 3 Pro."

  gemini-2.5-flash:
    provider: gemini
    model_id: "gemini-2.5-flash"
    display_name: "Gemini 2.5 Flash"
    capabilities: [text, vision, tool_calling, thinking, streaming, code_execution]
    context_window: 1000000
    max_output_tokens: 65536
    cost_per_million_input: 0.15
    cost_per_million_output: 0.60
    speed_tier: fast
    intelligence_tier: high
    supported_roles: [worker, summarizer, classifier]
    status: ga
    notes: "Cheapest Gemini with thinking capability."

  gemini-2.5-flash-lite:
    provider: gemini
    model_id: "gemini-2.5-flash-lite"
    display_name: "Gemini 2.5 Flash-Lite"
    capabilities: [text, tool_calling, streaming]
    context_window: 1000000
    max_output_tokens: 65536
    cost_per_million_input: 0.10
    cost_per_million_output: 0.40
    speed_tier: fast
    intelligence_tier: medium
    supported_roles: [classifier, worker]
    status: ga
    notes: "Ultra-cheap. Good for classification and simple tasks."

# ============================================================================
# ROLE-TO-MODEL MAPPING (Default Preferences)
# ============================================================================
role_defaults:

  orchestrator:
    primary: gemini-3-pro-preview
    fallbacks: [claude-opus-4-6, gpt-5, gemini-2.5-pro, claude-sonnet-4-5]
    min_intelligence: frontier
    prefer: intelligence

  worker:
    primary: gemini-3-flash-preview
    fallbacks: [gpt-4o-mini, claude-haiku-4-5, gemini-2.5-flash]
    min_intelligence: medium
    prefer: speed

  summarizer:
    primary: gemini-3-flash-preview
    fallbacks: [gpt-4o-mini, claude-haiku-4-5, gemini-2.5-flash]
    min_intelligence: medium
    prefer: cost

  judge:
    primary: claude-opus-4-6
    fallbacks: [gpt-5, gemini-3-pro-preview, o3, claude-sonnet-4-5]
    min_intelligence: high
    prefer: intelligence

  coder:
    primary: claude-opus-4-6
    fallbacks: [gpt-5, gemini-3-pro-preview, claude-sonnet-4-5]
    min_intelligence: high
    prefer: intelligence

  creative:
    primary: claude-sonnet-4-5
    fallbacks: [gpt-5, gemini-3-pro-preview, gemini-3-flash-preview]
    min_intelligence: high
    prefer: quality

  embedder:
    primary: text-embedding-3-large
    fallbacks: []
    prefer: quality

  classifier:
    primary: gemini-2.5-flash-lite
    fallbacks: [gpt-4o-mini, claude-haiku-4-5, gemini-2.5-flash]
    min_intelligence: low
    prefer: cost

# ============================================================================
# DEPRECATION NOTICES
# ============================================================================
deprecations:
  - model_id: "gemini-2.0-flash"
    deprecated_since: "2026-01-15"
    sunset_date: "2026-03-31"
    replacement: "gemini-2.5-flash"
    message: "Gemini 2.0 models sunset March 31, 2026. Migrate to 2.5+."

  - model_id: "gemini-2.0-pro"
    deprecated_since: "2026-01-15"
    sunset_date: "2026-03-31"
    replacement: "gemini-2.5-pro"
```

#### 3.2.2 Registry Resolution Order

```
1. Check CORTEX_MODEL_REGISTRY env var for file path
2. Check ./cortex_models.yaml in working directory
3. Check ~/.cortex/models.yaml in user home
4. Fall back to built-in defaults (current hardcoded models)

Models from later layers override earlier layers (merge semantics).
```

#### 3.2.3 Hot-Reload Mechanism

The registry should support hot-reload without restarting the SDK:

```python
class ModelRegistry:
    """External model registry with hot-reload support."""

    def __init__(self, config_path: Optional[str] = None):
        self._config_path = self._resolve_path(config_path)
        self._models: Dict[str, ModelEntry] = {}
        self._role_defaults: Dict[str, RoleConfig] = {}
        self._deprecations: List[DeprecationNotice] = []
        self._last_loaded: float = 0.0
        self._file_mtime: float = 0.0
        self._load()

    def reload_if_changed(self) -> bool:
        """Check file mtime and reload if changed. Call periodically."""
        if self._config_path and os.path.exists(self._config_path):
            mtime = os.path.getmtime(self._config_path)
            if mtime > self._file_mtime:
                self._load()
                return True
        return False

    def get_model_for_role(
        self,
        role: str,
        available_providers: Set[ProviderType],
        required_capabilities: Optional[List[str]] = None,
        prefer: str = "auto",  # auto | cost | speed | intelligence
        budget_remaining: Optional[float] = None,
    ) -> ModelEntry:
        """Select the best model for a given role from available providers."""
        ...

    def estimate_cost(
        self,
        model_id: str,
        input_tokens: int,
        output_tokens: int,
        cached: bool = False,
    ) -> float:
        """Estimate cost in USD for a given request."""
        ...
```

### 3.3 Model Deprecation Handling

```
On every model selection:
    1. Check if selected model has a deprecation notice
    2. If deprecated and sunset_date is past:
       - Log WARNING with replacement suggestion
       - Auto-switch to replacement model
    3. If deprecated and sunset_date is approaching (< 30 days):
       - Log INFO with migration suggestion
    4. Emit deprecation event through EventBus for tenant notification
```

### 3.4 A/B Testing New Models

To safely test new models without risking production quality:

```yaml
# In cortex_models.yaml
ab_tests:
  - name: "gemini-3-pro-ga-test"
    description: "Test Gemini 3 Pro GA against preview"
    control_model: "gemini-3-pro-preview"
    test_model: "gemini-3-pro"
    traffic_percentage: 10          # 10% of requests use test model
    roles: [orchestrator, coder]    # Only for these roles
    metrics: [quality_score, latency_ms, cost_usd, error_rate]
    start_date: "2026-03-01"
    end_date: "2026-03-15"
    auto_promote: true              # Auto-switch if test wins
    min_sample_size: 100
```

```python
class ABTestManager:
    """Manages A/B tests for model comparison."""

    def should_use_test_model(self, test_name: str, request_id: str) -> bool:
        """Deterministic routing based on request_id hash."""
        test = self._tests[test_name]
        hash_val = hash(request_id) % 100
        return hash_val < test.traffic_percentage

    def record_result(self, test_name: str, model: str, metrics: Dict) -> None:
        """Record outcome for statistical analysis."""
        ...

    def evaluate(self, test_name: str) -> ABTestResult:
        """Statistical significance test (Mann-Whitney U or similar)."""
        ...
```

---

## 4. Cost Optimization

### 4.1 Current Model Pricing Landscape (February 2026)

**Per 1 million tokens (input / output):**

| Model | Input | Output | Cached Input | Speed | Intelligence |
|-------|-------|--------|-------------|-------|-------------|
| **Gemini 2.5 Flash-Lite** | $0.10 | $0.40 | $0.01 | Fast | Medium |
| **Gemini 2.5 Flash** | $0.15 | $0.60 | $0.015 | Fast | High |
| **GPT-4o-mini** | $0.15 | $0.60 | N/A | Fast | Medium |
| **DeepSeek V3.2** | $0.27 | $1.10 | N/A | Medium | High |
| **GPT-5 Mini** | $0.30 | $1.25 | $0.03 | Fast | Medium |
| **Gemini 3 Flash** | $0.50 | $3.00 | $0.05 | Fast | High |
| **Claude Haiku 4.5** | $1.00 | $5.00 | $0.10 | Fast | Medium |
| **GPT-5** | $1.25 | $10.00 | $0.125 | Medium | Frontier |
| **Gemini 2.5 Pro** | $1.25 | $10.00 | $0.125 | Medium | Frontier |
| **Gemini 3 Pro** | $2.00 | $12.00 | $0.20 | Medium | Frontier |
| **GPT-4o** | $2.50 | $10.00 | N/A | Medium | High |
| **Claude Sonnet 4.5** | $3.00 | $15.00 | $0.30 | Medium | High |
| **Claude Opus 4.6** | $5.00 | $25.00 | $0.50 | Slow | Frontier |

### 4.2 Cost Impact of Routing Strategies

Consider a complex agentic task with 100 LLM calls:

**Strategy A: All Frontier (No Routing)**
- 100 calls x avg 2K input + 1K output tokens
- Using Claude Opus 4.6: (200K * $5.00 + 100K * $25.00) / 1M = $3.50 per task

**Strategy B: 80/20 Split (Current corteX)**
- 80 worker calls (Gemini 3 Flash): (160K * $0.50 + 80K * $3.00) / 1M = $0.32
- 20 orchestrator calls (Gemini 3 Pro): (40K * $2.00 + 20K * $12.00) / 1M = $0.32
- Total: **$0.64 per task** (82% savings vs Strategy A)

**Strategy C: 5-Role Split (Recommended)**
- 5 orchestrator calls (Opus 4.6): (10K * $5.00 + 5K * $25.00) / 1M = $0.175
- 50 worker calls (Gemini 2.5 Flash): (100K * $0.15 + 50K * $0.60) / 1M = $0.045
- 20 coder calls (Sonnet 4.5): (40K * $3.00 + 20K * $15.00) / 1M = $0.42
- 10 summarizer calls (Flash-Lite): (20K * $0.10 + 10K * $0.40) / 1M = $0.006
- 5 judge calls (Gemini 3 Pro): (10K * $2.00 + 5K * $12.00) / 1M = $0.08
- 10 classifier calls (Flash-Lite): (20K * $0.10 + 10K * $0.40) / 1M = $0.006
- Total: **$0.73 per task** (79% savings vs Strategy A, but with better quality allocation than B because expensive models are used precisely where they add value)

**Strategy D: Aggressive Cost Optimization**
- Same as C but using cheapest possible models per role
- Orchestrator: GPT-5 instead of Opus ($0.10)
- Worker: Gemini 2.5 Flash-Lite ($0.01)
- Coder: Gemini 3 Flash ($0.18)
- Total: **$0.38 per task** (89% savings)

### 4.3 Per-Task Cost Tracking

```python
@dataclass
class CostTracker:
    """Track LLM costs per session, per tenant, per task."""

    tenant_id: str
    session_id: str

    total_input_tokens: int = 0
    total_output_tokens: int = 0
    total_cached_tokens: int = 0
    total_cost_usd: float = 0.0

    cost_by_role: Dict[str, float] = field(default_factory=dict)
    cost_by_model: Dict[str, float] = field(default_factory=dict)
    cost_by_provider: Dict[str, float] = field(default_factory=dict)

    def record(self, model_id: str, role: str, usage: Dict[str, int], registry: ModelRegistry) -> float:
        """Record usage and return cost in USD."""
        model = registry.get_model(model_id)
        input_cost = usage["input_tokens"] * model.cost_per_million_input / 1_000_000
        output_cost = usage["output_tokens"] * model.cost_per_million_output / 1_000_000
        cached_cost = usage.get("cached_tokens", 0) * model.cost_per_million_cached_input / 1_000_000
        total = input_cost + output_cost + cached_cost

        self.total_input_tokens += usage["input_tokens"]
        self.total_output_tokens += usage["output_tokens"]
        self.total_cost_usd += total
        self.cost_by_role[role] = self.cost_by_role.get(role, 0.0) + total
        self.cost_by_model[model_id] = self.cost_by_model.get(model_id, 0.0) + total
        self.cost_by_provider[model.provider] = self.cost_by_provider.get(model.provider, 0.0) + total
        return total
```

### 4.4 Budget Enforcement

```python
class BudgetPolicy:
    """Enforce cost limits per tenant/session."""

    max_cost_per_session: Optional[float] = None     # e.g., $5.00
    max_cost_per_day_per_tenant: Optional[float] = None  # e.g., $100.00
    cost_warning_threshold: float = 0.8              # Warn at 80%

    def check(self, tracker: CostTracker) -> BudgetStatus:
        if self.max_cost_per_session and tracker.total_cost_usd > self.max_cost_per_session:
            return BudgetStatus.EXCEEDED
        if self.max_cost_per_session and tracker.total_cost_usd > self.max_cost_per_session * self.cost_warning_threshold:
            return BudgetStatus.WARNING
        return BudgetStatus.OK

    def get_allowed_max_tier(self, tracker: CostTracker) -> str:
        """Downgrade model tier as budget gets consumed."""
        remaining = (self.max_cost_per_session or float('inf')) - tracker.total_cost_usd
        if remaining < 0.10:
            return "low"       # Only cheapest models
        elif remaining < 0.50:
            return "medium"    # No frontier models
        return "frontier"      # Full access
```

---

## 5. Multi-Tenancy for Models

### 5.1 Per-Tenant Configuration

Different tenants may have different model access, preferences, and budgets:

```yaml
# Tenant configuration (stored in enterprise config or database)
tenants:
  tenant_acme:
    allowed_providers: [openai, gemini]    # No Anthropic access
    allowed_models: [gpt-5, gpt-4o-mini, gemini-3-pro-preview, gemini-3-flash-preview]
    blocked_models: []
    model_overrides:
      orchestrator: gpt-5                  # Prefer GPT-5 for orchestration
      worker: gemini-3-flash-preview
    rate_limits:
      openai: 100                          # RPM per provider
      gemini: 200
    budget:
      max_per_session: 10.00
      max_per_day: 500.00
    features:
      thinking_enabled: true
      streaming_enabled: true

  tenant_startup:
    allowed_providers: [gemini]            # Budget-conscious, Gemini only
    allowed_models: [gemini-3-flash-preview, gemini-2.5-flash, gemini-2.5-flash-lite]
    model_overrides:
      orchestrator: gemini-3-flash-preview
      worker: gemini-2.5-flash-lite
    rate_limits:
      gemini: 25                           # Tier-1 limits
    budget:
      max_per_session: 1.00
      max_per_day: 20.00
```

### 5.2 Provider Outage Handling

```python
class TenantModelResolver:
    """Resolve model selection per-tenant with outage awareness."""

    def resolve(
        self,
        tenant_config: TenantConfig,
        role: str,
        registry: ModelRegistry,
        provider_health: Dict[str, bool],
    ) -> str:
        """Select model for tenant, respecting access and health."""
        # 1. Get role default from registry
        role_config = registry.get_role_config(role)

        # 2. Apply tenant overrides
        if role in tenant_config.model_overrides:
            primary = tenant_config.model_overrides[role]
        else:
            primary = role_config.primary

        # 3. Check if primary is allowed for tenant
        if primary not in tenant_config.allowed_models:
            primary = None

        # 4. Check provider health
        if primary:
            model = registry.get_model(primary)
            if not provider_health.get(model.provider, True):
                primary = None  # Provider is down

        # 5. Walk fallback chain
        if not primary:
            for fallback in role_config.fallbacks:
                if fallback in tenant_config.allowed_models:
                    model = registry.get_model(fallback)
                    if provider_health.get(model.provider, True):
                        return fallback
            raise NoModelAvailableError(f"No available model for role={role}, tenant={tenant_config.id}")

        return primary
```

### 5.3 Per-Tenant Rate Limiting

The current `RateLimiter` in `resilience.py` tracks rates per provider globally. For multi-tenancy, it needs to track per `(tenant_id, provider)` pair:

```python
# Current: self._rate_limiter.acquire("openai")
# Enhanced: self._rate_limiter.acquire("openai", tenant_id="acme")
```

This ensures one tenant's heavy usage does not exhaust another tenant's quota.

---

## 6. Future-Proofing

### 6.1 Current Model Landscape (February 2026)

The model landscape as of February 2026 includes several major players:

**Frontier Models (for Orchestrator/Judge):**
- Claude Opus 4.6 (Anthropic, Feb 2026) -- 1M context, strongest on SWE-bench
- GPT-5.2 / GPT-5.3 Codex (OpenAI, Aug 2025 / Feb 2026) -- 400K context, strong coding
- Gemini 3 Pro (Google, Feb 2026) -- 1M context, thinking, code execution
- o3 (OpenAI) -- reasoning model with internal CoT

**Fast/Cheap Models (for Worker/Classifier):**
- Gemini 2.5 Flash-Lite (Google) -- $0.10/$0.40 per M, ultra-cheap
- GPT-4o-mini (OpenAI) -- $0.15/$0.60 per M, widely used
- Gemini 2.5 Flash (Google) -- $0.15/$0.60 per M, with thinking
- Gemini 3 Flash (Google) -- $0.50/$3.00 per M, best price/performance
- Claude Haiku 4.5 (Anthropic) -- $1.00/$5.00 per M

**Open-Source / On-Prem Models:**
- DeepSeek V3.2 -- MIT license, frontier quality, $0.27/$1.10 via API, free self-hosted
- Llama 4 (Meta) -- Community license (< 700M MAU), natively multimodal
- Mistral Large 3 -- MoE architecture, commercial license required at scale
- Qwen 3 (Alibaba) -- Strong multilingual, Apache 2.0 license

### 6.2 Emerging Capabilities to Accommodate

The registry schema should be extensible for capabilities that do not yet exist:

| Capability | Status | Impact on Routing |
|-----------|--------|-------------------|
| **Extended Thinking / CoT** | Available (Opus, o3, Gemini 3) | Route complex tasks to thinking-capable models |
| **1M+ Context Windows** | Available (Gemini, expanding) | Eliminates need for summarizer role in some cases |
| **Native Tool Use** | Standard | Core requirement for worker role |
| **Multimodal (Image/Audio/Video)** | Expanding | Route vision/audio tasks to capable models |
| **Code Execution** | Available (Gemini) | Enables agentic code tasks without external sandbox |
| **Real-time / Sub-second** | Emerging | New speed tier for interactive applications |
| **Agent-to-Agent Communication** | Emerging (MCP) | May change how multi-agent routing works |
| **Fine-tuned / Distilled Models** | Available | Per-tenant custom models in registry |
| **Reasoning Tokens** | Available (o3, o4-mini) | Cost tracking must account for hidden reasoning tokens |

### 6.3 Architecture for Unknown Future Models

The key design principle is: **never assume you know what models exist at compile time.**

```python
# BAD: Hardcoded model knowledge
if model.startswith("gemini-3"):
    temperature = 1.0  # Gemini 3 specific

# GOOD: Registry-driven behavior
temp_override = registry.get_model(model).recommended_temperature.get(task_type)
if temp_override is not None:
    temperature = temp_override
```

```python
# BAD: Hardcoded provider detection
if provider_type == ProviderType.GEMINI:
    from corteX.core.llm.gemini_adapter import GeminiAdapter
    provider = GeminiAdapter(...)

# GOOD: Plugin-based provider registration
provider_class = registry.get_provider_class(provider_type)
provider = provider_class(**config)
```

The registry YAML should support arbitrary metadata fields that the SDK does not need to understand but that custom routing logic can use:

```yaml
models:
  future-model-x:
    provider: newprovider
    # Standard fields...
    extensions:
      supports_real_time: true
      max_agents: 10
      custom_auth_flow: "oauth2"
      minimum_gpu_memory: "80GB"
```

---

## 7. Specific Improvements to Current Router

### 7.1 Gap Analysis

| Area | Current State | Recommended State | Priority |
|------|--------------|-------------------|----------|
| Model Registry | Hardcoded in provider classes | External YAML with hot-reload | **CRITICAL** |
| Role Taxonomy | 2 roles (orchestrator/worker) | 8 roles (6 generative + 2 non-generative) | HIGH |
| Task Classification | Keyword heuristics | Hybrid: heuristics + brain signals + optional classifier | HIGH |
| Cost Tracking | None | Per-session, per-tenant, per-role cost tracking | HIGH |
| Budget Enforcement | None | Configurable limits with auto-downgrade | HIGH |
| Per-Tenant Models | Not supported | Full per-tenant model/provider access control | HIGH |
| A/B Testing | Not supported | Config-driven A/B test framework | MEDIUM |
| Deprecation Handling | None | Automatic deprecation warnings + migration | MEDIUM |
| Temperature Config | Hardcoded per-provider dicts | Registry-driven per-model-per-task config | MEDIUM |
| Fallback Chains | Single alternative | Configurable per-role fallback chains | MEDIUM |
| Embedding Models | Not in router | Separate embedder role with dedicated models | LOW |

### 7.2 Files That Need Changes

1. **NEW: `corteX/core/llm/registry.py`** -- ModelRegistry class, YAML loader, hot-reload
2. **NEW: `cortex_models.yaml`** -- Default model registry shipped with SDK
3. **MODIFY: `corteX/core/llm/router.py`** -- Consume registry instead of hardcoded models, expand roles, add cost tracking
4. **MODIFY: `corteX/core/llm/base.py`** -- Expand ModelCapability enum, enhance ModelInfo
5. **MODIFY: `corteX/core/llm/openai_client.py`** -- Remove hardcoded model list, load from registry
6. **MODIFY: `corteX/core/llm/gemini_adapter.py`** -- Remove hardcoded model list, load from registry
7. **MODIFY: `corteX/core/llm/anthropic_client.py`** -- Remove hardcoded model list, load from registry
8. **MODIFY: `corteX/sdk.py`** -- Enhanced role selection using 8 roles, cost tracking integration
9. **NEW: `corteX/core/llm/cost_tracker.py`** -- Cost tracking and budget enforcement
10. **NEW: `corteX/core/llm/ab_testing.py`** -- A/B test framework for model comparison

### 7.3 Routing Algorithm Pseudocode

```python
async def route_request(
    messages: List[LLMMessage],
    tools: Optional[List[ToolDefinition]],
    role: str,                           # From brain's role selection
    tenant_config: TenantConfig,
    session_cost_tracker: CostTracker,
    registry: ModelRegistry,
    provider_health: Dict[str, ProviderHealth],
) -> RoutingDecision:
    """
    The complete routing algorithm combining all signals.

    Returns a RoutingDecision with selected model, provider, and metadata.
    """

    # ==========================================
    # PHASE 1: TASK ANALYSIS
    # ==========================================

    # 1a. Quick heuristic classification (existing _classify_task)
    task_type = classify_task_heuristic(messages, tools)

    # 1b. Estimate complexity from brain signals (already computed in _prepare_turn)
    # This uses attention_priority, process_type, resource_alloc, etc.
    # Result: complexity in [0.0, 1.0]
    # complexity is already encoded in the role selection

    # ==========================================
    # PHASE 2: MODEL CANDIDATE GENERATION
    # ==========================================

    # 2a. Get all models for this role from registry
    role_config = registry.get_role_config(role)
    candidates = [role_config.primary] + role_config.fallbacks

    # 2b. Filter by tenant access
    candidates = [m for m in candidates if m in tenant_config.allowed_models]

    # 2c. Filter by required capabilities
    required_caps = determine_required_capabilities(tools, messages)
    # e.g., if tools present -> need tool_calling
    # e.g., if images in messages -> need vision
    candidates = [m for m in candidates
                  if registry.get_model(m).has_capabilities(required_caps)]

    # 2d. Filter by provider health
    candidates = [m for m in candidates
                  if provider_health[registry.get_model(m).provider].is_healthy]

    # 2e. Filter by rate limit headroom
    candidates = [m for m in candidates
                  if rate_limiter.has_headroom(
                      registry.get_model(m).provider,
                      tenant_config.tenant_id
                  )]

    # 2f. Filter by budget constraints
    budget_tier = budget_policy.get_allowed_max_tier(session_cost_tracker)
    candidates = [m for m in candidates
                  if intelligence_tier_rank(registry.get_model(m).intelligence_tier)
                     <= intelligence_tier_rank(budget_tier)]

    if not candidates:
        raise NoModelAvailableError(
            f"No model available for role={role}, tenant={tenant_config.id}"
        )

    # ==========================================
    # PHASE 3: MODEL SCORING & SELECTION
    # ==========================================

    # 3a. Score each candidate
    scored = []
    for model_id in candidates:
        model = registry.get_model(model_id)

        # Base weight from weight engine (existing)
        weight_score = weight_engine.get_weight(task_type, model_id)

        # Cost score: normalized inverse of cost (cheaper = higher score)
        max_cost = max(registry.get_model(c).cost_per_million_output for c in candidates)
        cost_score = 1.0 - (model.cost_per_million_output / max_cost) if max_cost > 0 else 0.5

        # Speed score
        speed_score = {"fast": 1.0, "medium": 0.5, "slow": 0.0}[model.speed_tier]

        # Intelligence score
        intel_score = {"low": 0.0, "medium": 0.33, "high": 0.66, "frontier": 1.0}[model.intelligence_tier]

        # Historical performance for this task type
        history_score = performance_history.get_score(model_id, task_type)

        # Latency history
        latency_score = latency_history.get_normalized_score(model_id)

        # Preference weights based on role config
        prefer = role_config.prefer  # "cost" | "speed" | "intelligence" | "quality"
        if prefer == "cost":
            weights = {"weight": 0.1, "cost": 0.4, "speed": 0.1, "intel": 0.1, "history": 0.2, "latency": 0.1}
        elif prefer == "speed":
            weights = {"weight": 0.1, "cost": 0.1, "speed": 0.4, "intel": 0.1, "history": 0.2, "latency": 0.1}
        elif prefer == "intelligence":
            weights = {"weight": 0.2, "cost": 0.05, "speed": 0.05, "intel": 0.4, "history": 0.2, "latency": 0.1}
        else:  # quality
            weights = {"weight": 0.2, "cost": 0.1, "speed": 0.1, "intel": 0.3, "history": 0.2, "latency": 0.1}

        final_score = (
            weights["weight"] * weight_score +
            weights["cost"] * cost_score +
            weights["speed"] * speed_score +
            weights["intel"] * intel_score +
            weights["history"] * history_score +
            weights["latency"] * latency_score
        )

        scored.append((model_id, final_score))

    # 3b. Sort by score
    scored.sort(key=lambda x: x[1], reverse=True)

    # 3c. Check for A/B test override
    selected_model = scored[0][0]
    for ab_test in registry.get_active_ab_tests():
        if (ab_test.control_model == selected_model
            and role in ab_test.roles
            and ab_test.should_use_test(request_id)):
            selected_model = ab_test.test_model

    # ==========================================
    # PHASE 4: DECISION OUTPUT
    # ==========================================

    model = registry.get_model(selected_model)

    # Resolve temperature from registry
    temperature = model.recommended_temperature.get(
        task_type,
        model.recommended_temperature.get("default", 0.7)
    )

    return RoutingDecision(
        provider=model.provider,
        model=selected_model,
        role=role,
        task_type=task_type,
        temperature=temperature,
        reason=f"role={role}, prefer={role_config.prefer}, score={scored[0][1]:.3f}",
        confidence=scored[0][1],
        alternatives=[s[0] for s in scored[1:3]],
        estimated_cost=registry.estimate_cost(selected_model, estimated_tokens),
    )
```

---

## 8. Implementation Roadmap

### Phase 1: Model Registry (Week 1)
- Create `ModelRegistry` class with YAML loading
- Create default `cortex_models.yaml` with all current models
- Wire registry into `LLMRouter`
- Remove hardcoded model lists from provider classes (providers still need to know how to *talk* to models, but not *which* models exist)
- Add deprecation warning system

### Phase 2: Extended Roles (Week 1-2)
- Expand `TaskType` to include all 8 roles
- Enhance role selection logic in `sdk.py` to use the full taxonomy
- Add role-to-model mapping in registry
- Update `TurnContext` with new role field

### Phase 3: Cost Tracking (Week 2)
- Create `CostTracker` class
- Integrate cost recording into `LLMRouter.generate()`
- Add per-session and per-tenant cost reporting
- Add budget enforcement with auto-downgrade

### Phase 4: Multi-Tenancy (Week 2-3)
- Add per-tenant model access configuration
- Per-tenant rate limiting in `RateLimiter`
- Tenant-aware model resolution

### Phase 5: A/B Testing & Analytics (Week 3-4)
- A/B test framework
- Performance history tracking per model per task type
- Dashboard data export

---

## 9. Research Sources

- [RouteLLM: An Open-Source Framework for Cost-Effective LLM Routing (LMSYS / UC Berkeley)](https://lmsys.org/blog/2024-07-01-routellm/)
- [The Complete Guide to LLM Routing: 5 AI Gateways (2026)](https://medium.com/@kamyashah2018/the-complete-guide-to-llm-routing-5-ai-gateways-transforming-production-ai-infrastructure-b5c68ee6d641)
- [Intelligent LLM Routing: How Multi-Model AI Cuts Costs by 85% (Swfte AI)](https://www.swfte.com/blog/intelligent-llm-routing-multi-model-ai)
- [Intelligent LLM Routing in Enterprise AI (Requesty)](https://www.requesty.ai/blog/intelligent-llm-routing-in-enterprise-ai-uptime-cost-efficiency-and-model)
- [Multi-LLM Routing Strategies on AWS](https://aws.amazon.com/blogs/machine-learning/multi-llm-routing-strategies-for-generative-ai-applications-on-aws/)
- [LLMRouter: Intelligent Routing System (MarkTechPost)](https://www.marktechpost.com/2025/12/30/meet-llmrouter-an-intelligent-routing-system-designed-to-optimize-llm-inference-by-dynamically-selecting-the-most-suitable-model-for-each-query/)
- [LLM Semantic Router (Red Hat)](https://developers.redhat.com/articles/2025/05/20/llm-semantic-router-intelligent-request-routing)
- [Claude Code Model Configuration](https://code.claude.com/docs/en/model-config)
- [Choosing the Right Claude Model (Anthropic)](https://platform.claude.com/docs/en/about-claude/models/choosing-a-model)
- [Multi-Tenant LLM Access (Portkey)](https://portkey.ai/docs/guides/use-cases/multi-tenant-ai-feature)
- [Multi-Tenant Architecture with LiteLLM](https://docs.litellm.ai/docs/proxy/multi_tenant_architecture)
- [OpenAI API Pricing (2026)](https://platform.openai.com/docs/pricing)
- [Claude API Pricing (2026)](https://platform.claude.com/docs/en/about-claude/pricing)
- [Gemini API Pricing (2026)](https://ai.google.dev/gemini-api/docs/pricing)
- [Claude Opus 4.6 vs GPT-5.2 vs Gemini 3 Pro Comparison](https://www.humai.blog/claude-opus-4-6-vs-gpt-5-2-vs-gemini-3-pro-which-ai-model-should-you-actually-use-in-2026/)
- [Best AI Models 2026: Complete Comparison](https://www.humai.blog/best-ai-models-2026-gpt-5-vs-claude-4-5-opus-vs-gemini-3-pro-complete-comparison/)
- [Top 10 Open Source LLMs 2026 (o-mega)](https://o-mega.ai/articles/top-10-open-source-llms-the-deepseek-revolution-2026)
- [Model Routing Agents: LLM Mesh Architectures](https://ai-academy.training/2025/11/14/model-routing-agents-the-emerging-pattern-of-llm-mesh-architectures/)
- [MasRouter: Learning to Route LLMs for Multi-Agent System (ACL 2025)](https://aclanthology.org/2025.acl-long.757.pdf)
- [NVIDIA LLM Router Blueprint](https://github.com/NVIDIA-AI-Blueprints/llm-router)
- [OpenRouter Provider Routing Documentation](https://openrouter.ai/docs/guides/routing/provider-selection)
- [RouteLLM Published at ICLR 2025](https://proceedings.iclr.cc/paper_files/paper/2025/file/5503a7c69d48a2f86fc00b3dc09de686-Paper-Conference.pdf)
- [Cost and Latency Constrained Routing for LLMs (Harvard)](http://minlanyu.seas.harvard.edu/writeup/sllm25-score.pdf)
- [Dynamic LLM Routing and Selection Based on User Preferences (arXiv)](https://arxiv.org/abs/2502.16696)

---

## 10. Conclusion

corteX is well-positioned to build a best-in-class model routing system. The brain-inspired architecture (attention, dual-process routing, resource homunculus, weight engine) already produces the signals needed for intelligent model selection. The primary gaps are:

1. **Externalized model registry** -- the most critical missing piece, preventing easy updates when new models launch.
2. **Expanded role taxonomy** -- moving beyond binary orchestrator/worker to leverage the full spectrum of available model tiers.
3. **Cost tracking and budget enforcement** -- essential for enterprise multi-tenancy.
4. **Per-tenant model configuration** -- required for the enterprise use case.

The recommended YAML-based registry with hot-reload capability provides the right balance of simplicity (human-editable config file) and power (supports deprecations, A/B tests, per-model temperature overrides, and arbitrary extensions). When GPT-6 or Gemini 4 launches, the user updates one YAML file instead of waiting for an SDK release.

Estimated total implementation effort: **3-4 weeks** for a single developer, or **1-2 weeks** with parallel agent teams.
