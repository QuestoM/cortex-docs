# Intelligent Model Routing

corteX uses intelligent model routing to select the right LLM for each task based on cost, quality, latency, and learned performance. The system combines external model registries, zero-LLM task classification, and dynamic cost tracking to optimize model selection across 13+ models from 4 providers.

## What It Does

The model routing system provides three core capabilities:

1. **ModelRegistry**: External YAML-based model catalog with 8 role types and tenant overrides
2. **CognitiveClassifier**: Zero-LLM heuristic task classifier (10 types × 5 tiers, <1ms latency)
3. **CostTracker**: Per-session/task/tenant cost tracking with budget enforcement and anomaly detection

Together, these components enable intelligent, cost-effective model selection without hardcoded configurations.

---

## ModelRegistry

**corteX Innovation**: External YAML-based model registry with hot-reload and tenant-specific overrides.

The ModelRegistry externalizes model configuration into a `cortex_models.yaml` file, enabling model updates without code changes. It supports 8 role types, multi-provider failover, and per-tenant customization.

### Why: The Model Explosion Problem

As of February 2026, there are 13+ production-ready models across 4 providers:

| Provider | Models | Use Cases |
|----------|--------|-----------|
| **Google** | gemini-3-pro-preview, gemini-3-flash-preview, gemini-2.5-pro, gemini-2.5-flash | Orchestration, fast workers, long context |
| **OpenAI** | gpt-4o, gpt-4o-mini, o1-preview, o1-mini | High-quality reasoning, cost-effective workers |
| **Anthropic** | claude-opus-4-6, claude-sonnet-4-5, claude-haiku-4-5 | Complex tasks, coding, analysis |
| **Local** | llama-3-70b, mixtral-8x7b | On-prem, privacy-sensitive deployments |

Hardcoding model selection in code creates:
- **Deployment friction**: Every model change requires code push
- **Tenant inflexibility**: All tenants use the same models
- **Provider lock-in**: Difficult to switch providers

ModelRegistry solves this with a **hot-reloadable external catalog**.

### How It Works

```python
from corteX.core.llm.registry import ModelRegistry, ModelRole

# Load registry from YAML file
registry = ModelRegistry(config_paths=["cortex_models.yaml"])

# Get best model for a role
orchestrator = registry.get_model_for_role(ModelRole.ORCHESTRATOR)
print(orchestrator.model_id)
# "gemini-3-pro-preview"

print(orchestrator.provider)
# "gemini"

print(orchestrator.pricing)
# {"input": 0.000125, "output": 0.0005, "currency": "USD"}

# Get all models for a role (for failover)
workers = registry.get_models_for_role(ModelRole.WORKER)
for model in workers:
    print(f"{model.model_id}: ${model.pricing['input']}/1K input tokens")

# Output:
# gemini-3-flash-preview: $0.000075/1K input tokens
# gpt-4o-mini: $0.00015/1K input tokens
# claude-haiku-4-5: $0.00025/1K input tokens
```

### Eight Model Roles

| Role | Purpose | Example Models |
|------|---------|----------------|
| **orchestrator** | Complex multi-step reasoning | gemini-3-pro, claude-opus-4-6, gpt-4o |
| **worker** | Fast routine tasks | gemini-3-flash, gpt-4o-mini, claude-haiku-4-5 |
| **summarizer** | Context compression | gemini-2.5-flash, gpt-4o-mini |
| **judge** | Quality evaluation | gemini-2.5-pro, gpt-4o |
| **embedder** | Vector embeddings | text-embedding-004, text-embedding-3-large |
| **classifier** | Intent/category detection | gemini-2.5-flash, gpt-4o-mini |
| **creative** | Content generation | gemini-3-pro, claude-sonnet-4-5 |
| **fast** | Sub-second responses | gemini-3-flash, gpt-4o-mini |

### YAML Format

The `cortex_models.yaml` file defines models with metadata:

```yaml
models:
  - id: gemini-3-pro-preview
    provider: gemini
    display_name: "Gemini 3 Pro Preview"
    roles: [orchestrator, judge, creative]
    features:
      context_window: 1000000
      max_output: 8192
      supports_tools: true
      supports_streaming: true
      supports_thinking: false
    pricing:
      input: 0.000125  # USD per 1K tokens
      output: 0.0005
      currency: USD
    quality:
      reasoning: 0.95
      coding: 0.90
      creative: 0.92
    rate_limits:
      rpm: 25  # requests per minute
      rpd: 250  # requests per day
      tpm: 2000000  # tokens per minute
    active: true

  - id: gemini-3-flash-preview
    provider: gemini
    display_name: "Gemini 3 Flash Preview"
    roles: [worker, summarizer, classifier, fast]
    features:
      context_window: 1000000
      max_output: 8192
      supports_tools: true
      supports_streaming: true
      supports_thinking: false
    pricing:
      input: 0.000075
      output: 0.0003
      currency: USD
    quality:
      reasoning: 0.80
      coding: 0.75
      creative: 0.70
    rate_limits:
      rpm: 25
      rpd: 250
      tpm: 4000000
    active: true

  # ... 11 more models ...
```

### Hot-Reload

Update the YAML file and reload the registry without restarting:

```python
# Initial load
registry = ModelRegistry.from_yaml("cortex_models.yaml")

# ... time passes, YAML file is updated ...

# Reload without restarting the app
registry.reload()

# New model configuration is active
```

### Tenant Overrides

Each tenant can override the default registry:

```python
# Default registry
default_registry = ModelRegistry.from_yaml("cortex_models.yaml")

# Tenant-specific override (uses only Claude models)
tenant_override = {
    "models": [
        {"id": "claude-opus-4-6", "provider": "anthropic", "roles": ["orchestrator"]},
        {"id": "claude-haiku-4-5", "provider": "anthropic", "roles": ["worker"]},
    ]
}

# Create tenant registry
tenant_registry = ModelRegistry.from_dict(tenant_override)

# Use tenant registry for this session
session = cortex.Session(
    tenant_id="acme_corp",
    model_registry=tenant_registry,
)
```

### Model Filtering

Filter models by capabilities:

```python
# Get all models that support tool calling
tool_models = registry.filter_models(
    supports_tools=True,
    min_context_window=100000,
)

# Get cost-effective orchestrators
cheap_orchestrators = registry.filter_models(
    roles=[ModelRole.ORCHESTRATOR],
    max_input_cost=0.0002,  # Max $0.0002/1K tokens
)

# Get models with high reasoning quality
reasoning_models = registry.filter_models(
    min_quality_reasoning=0.90,
)
```

---

## CognitiveClassifier

**corteX Innovation**: Zero-LLM heuristic task classification with <1ms latency.

The CognitiveClassifier categorizes user messages into 10 cognitive types and 5 complexity tiers using deterministic heuristics, eliminating the need for expensive LLM-based classification.

### Why: The Classification Cost Problem

Traditional task classification uses LLM calls:

```python
# Slow, expensive, requires LLM call
task_type = llm.classify(message)
```

For 10,000 tasks, this costs $100+ in classification alone. CognitiveClassifier replaces this with **keyword-based heuristics** that run in <1ms with zero API cost.

### How It Works

The classifier uses keyword patterns and complexity heuristics:

```python
from corteX.core.llm.classifier import CognitiveClassifier, CognitiveType, ComplexityTier

classifier = CognitiveClassifier()

# Classify a simple request
task = classifier.classify("What is 2 + 2?")
print(task.cognitive_type)
# CognitiveType.FACTUAL

print(task.complexity)
# ComplexityTier.TRIVIAL

# Classify a complex request
task = classifier.classify(
    "Build a production-ready REST API with JWT authentication, "
    "PostgreSQL database, Redis caching, and comprehensive test coverage"
)
print(task.cognitive_type)
# CognitiveType.CREATIVE

print(task.complexity)
# ComplexityTier.EXPERT

print(task.confidence)
# 0.87 (87% confidence in classification)

print(task.suggested_model_role)
# ModelRole.ORCHESTRATOR
```

### Ten Cognitive Types

| Type | Keywords | Example |
|------|----------|---------|
| **REASONING** | why, analyze, compare, explain, deduce | "Why is Python popular?" |
| **PLANNING** | plan, strategy, roadmap, steps, decompose | "Plan a migration from MongoDB to PostgreSQL" |
| **CODING** | implement, code, function, class, debug | "Implement a binary search tree in Python" |
| **CREATIVE** | write, create, design, generate, build | "Write a poem about coding" |
| **FACTUAL_RECALL** | what, who, when, where, define | "What is Python?" |
| **SUMMARIZATION** | summarize, tldr, brief, overview | "Summarize this 50-page document" |
| **VALIDATION** | check, validate, review, verify | "Validate this code implementation" |
| **DECISION** | should, decide, choose, recommend | "Should I use TypeScript or JavaScript?" |
| **TOOL_USE** | call, execute, run, use | "Execute the database migration" |
| **CLASSIFICATION** | classify, categorize, label, tag | "Classify this issue as bug or feature" |

### Five Complexity Tiers

| Tier | Indicators | Suggested Model |
|------|-----------|-----------------|
| **TRIVIAL** | <10 words, simple question | Fast worker |
| **SIMPLE** | 10-30 words, single concept | Worker |
| **MODERATE** | 30-100 words, multiple concepts | Worker or orchestrator |
| **COMPLEX** | 100-200 words, multi-step | Orchestrator |
| **CRITICAL** | 200+ words, highly technical | High-quality orchestrator |

### Complexity Scoring

The classifier uses multiple heuristics:

```python
complexity_score = (
    0.3 * word_count_score +
    0.2 * technical_term_score +
    0.2 * multi_step_indicator +
    0.15 * domain_specificity +
    0.15 * constraint_count
)
```

### Confidence Calibration

The classifier reports confidence based on keyword match strength:

- **High confidence (>0.8)**: Strong keyword match, unambiguous type
- **Medium confidence (0.5-0.8)**: Partial keyword match, some ambiguity
- **Low confidence (<0.5)**: Weak keyword match, fallback classification

```python
task = classifier.classify("Do something with the database")
print(task.confidence)
# 0.42 (low confidence - vague request)

if task.confidence < 0.5:
    # Escalate to LLM-based classification or ask user for clarification
    pass
```

---

## CostTracker

**corteX Innovation**: Real-time cost tracking with budget enforcement, anomaly detection, and multi-level aggregation.

The CostTracker monitors spending across sessions, tasks, and tenants, enforcing budgets and detecting cost anomalies.

### Why: The Runaway Cost Problem

LLM costs can spiral out of control:

- A single bug triggers 1,000 retries → $50 wasted
- A long-running agent consumes 10M tokens → $100 unexpected cost
- A tenant abuses the system with 10,000 requests → $1,000 loss

CostTracker prevents this with **proactive budget enforcement** and **anomaly detection**.

### How It Works

```python
from corteX.core.llm.cost_tracker import CostTracker, BudgetPolicy

# Initialize tracker
tracker = CostTracker()

# Set budgets
tracker.set_budget(
    scope="session",
    session_id="cs-001",
    soft_limit=5.00,  # Warning at $5
    hard_limit=10.00,  # Block at $10
)

tracker.set_budget(
    scope="tenant",
    tenant_id="acme_corp",
    soft_limit=100.00,
    hard_limit=150.00,
)

# Record LLM usage
tracker.record_usage(
    session_id="cs-001",
    tenant_id="acme_corp",
    task_id="task-123",
    model_id="gemini-3-pro-preview",
    input_tokens=1500,
    output_tokens=500,
    cost=0.00031,  # Calculated from model pricing
)

# Check budget status
status = tracker.get_budget_status(
    scope="session",
    session_id="cs-001"
)
print(f"Spent: ${status.spent:.4f} / ${status.hard_limit:.2f}")
print(f"Remaining: ${status.remaining:.4f}")
print(f"Warning: {status.soft_limit_exceeded}")
print(f"Blocked: {status.hard_limit_exceeded}")

# Output:
# Spent: $0.0003 / $10.00
# Remaining: $9.9997
# Warning: False
# Blocked: False
```

### Budget Enforcement

The tracker enforces budgets at two levels:

1. **Soft Limit**: Log warning, notify user, continue execution
2. **Hard Limit**: Block execution, raise exception

```python
# Before LLM call, check budget
if tracker.would_exceed_budget(
    scope="session",
    session_id="cs-001",
    estimated_cost=0.50
):
    raise BudgetExceededError("Session budget exceeded")

# Proceed with LLM call
response = llm.generate(...)

# Record actual cost
tracker.record_usage(...)
```

### Anomaly Detection

The tracker detects unusual spending patterns:

```python
# Record normal usage
for i in range(100):
    tracker.record_usage(
        session_id=f"cs-{i}",
        tenant_id="acme_corp",
        task_id=f"task-{i}",
        model_id="gemini-3-flash-preview",
        input_tokens=1000,
        output_tokens=500,
        cost=0.00015,
    )

# Record anomalous usage (10x normal cost)
tracker.record_usage(
    session_id="cs-101",
    tenant_id="acme_corp",
    task_id="task-101",
    model_id="gemini-3-pro-preview",
    input_tokens=50000,  # 50x normal
    output_tokens=20000,
    cost=0.01625,  # 100x normal
)

# Detect anomaly
anomalies = tracker.detect_anomalies(
    tenant_id="acme_corp",
    threshold=3.0,  # 3x standard deviation
)

for anomaly in anomalies:
    print(f"ANOMALY: {anomaly.session_id}")
    print(f"  Cost: ${anomaly.cost:.4f} (expected: ${anomaly.expected_cost:.4f})")
    print(f"  Severity: {anomaly.severity}x normal")

# Output:
# ANOMALY: cs-101
#   Cost: $0.0163 (expected: $0.0002)
#   Severity: 108.3x normal
```

### Multi-Level Aggregation

Track costs at three levels:

```python
# Session-level costs
session_cost = tracker.get_total_cost(scope="session", session_id="cs-001")

# Task-level costs
task_cost = tracker.get_total_cost(scope="task", task_id="task-123")

# Tenant-level costs
tenant_cost = tracker.get_total_cost(scope="tenant", tenant_id="acme_corp")

# Breakdown by model
breakdown = tracker.get_cost_breakdown(tenant_id="acme_corp")
for model, cost in breakdown.items():
    print(f"{model}: ${cost:.4f}")

# Output:
# gemini-3-pro-preview: $0.0031
# gemini-3-flash-preview: $0.0015
# claude-opus-4-6: $0.0042
```

### Cost Optimization Insights

The tracker provides optimization recommendations:

```python
insights = tracker.get_optimization_insights(tenant_id="acme_corp")

for insight in insights:
    print(f"{insight.category}: {insight.recommendation}")
    print(f"  Potential savings: ${insight.potential_savings:.2f}/month")

# Output:
# model_selection: Switch 30% of orchestrator calls to worker models
#   Potential savings: $45.00/month
# context_compression: Enable aggressive compression for long conversations
#   Potential savings: $23.00/month
```

---

## Integration with LLM Router

All three components integrate seamlessly into the LLM router:

```python
# In LLMRouter.__init__()
self.model_registry = ModelRegistry.from_yaml("cortex_models.yaml")
self.classifier = CognitiveClassifier()
self.cost_tracker = CostTracker()

# In LLMRouter.generate()

# 1. Classify the task
task = self.classifier.classify(user_message)

# 2. Select model based on classification
if task.complexity >= ComplexityTier.COMPLEX:
    role = ModelRole.ORCHESTRATOR
else:
    role = ModelRole.WORKER

model = self.model_registry.get_model_for_role(role)

# 3. Check budget before calling
estimated_cost = self._estimate_cost(model, message_tokens)
if self.cost_tracker.would_exceed_budget(session_id, estimated_cost):
    raise BudgetExceededError()

# 4. Call LLM
response = self._call_llm(model, messages)

# 5. Record actual cost
actual_cost = self._calculate_cost(model, input_tokens, output_tokens)
self.cost_tracker.record_usage(
    session_id=session_id,
    tenant_id=tenant_id,
    task_id=task_id,
    model_id=model.model_id,
    input_tokens=input_tokens,
    output_tokens=output_tokens,
    cost=actual_cost,
)

# 6. Check for anomalies
anomalies = self.cost_tracker.detect_anomalies(tenant_id, threshold=3.0)
if anomalies:
    logger.warning(f"Cost anomaly detected: {anomalies}")
```

---

## Configuration

```python
from corteX.core.llm.registry import ModelRegistry
from corteX.core.llm.classifier import CognitiveClassifier
from corteX.core.llm.cost_tracker import CostTracker, BudgetPolicy

# ModelRegistry configuration
registry = ModelRegistry(
    config_paths=["cortex_models.yaml"],
    auto_reload=True  # Hot-reload on file change
)

# CognitiveClassifier configuration
classifier = CognitiveClassifier()

# CostTracker configuration
tracker = CostTracker()
```

---

## API Reference

```python
from corteX.core.llm.registry import (
    ModelRegistry,
    ModelEntry,
    ModelRole,
    RoleMapping,
    ModelFeatures,
)
from corteX.core.llm.classifier import (
    CognitiveClassifier,
    CognitiveType,
    ComplexityTier,
    CognitiveProfile,
)
from corteX.core.llm.cost_tracker import (
    CostTracker,
    BudgetPolicy,
    BudgetCheck,
    BudgetLevel,
    CostRecord,
    CostSummary,
    CostAnomaly,
)
```

See also:
- [LLM Routing](llm-routing.md) - Original multi-model routing documentation
- [Multi-Tenant Setup](../enterprise/multi-tenant.md) - Tenant-specific model configurations
- [Cognitive Context](cognitive-context.md) - Context quality and state management
