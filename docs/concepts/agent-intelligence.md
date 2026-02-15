# Agent Intelligence

Agent Intelligence modules give corteX agents the ability to collaborate across multiple models, speculate about future steps, maintain decision audit trails, estimate progress, run controlled experiments, and monitor provider health. These capabilities transform agents from simple prompt-response systems into adaptive, self-aware executors.

## What It Does

The agent intelligence system provides six capabilities:

1. **ModelMosaic**: Multi-model collaboration within a single logical turn
2. **SpeculativeExecutor**: Pre-computes context for predicted next steps to reduce latency
3. **DecisionLog**: Audit trail for every agent branch point with alternatives and reasoning
4. **ProgressEstimator**: Velocity-based progress prediction with trend analysis
5. **ABTestManager**: Statistically rigorous model and strategy comparison
6. **ProviderHealthMonitor**: Real-time health tracking with circuit breaker for LLM providers

Together, these components enable agents to work faster, make better decisions, and self-diagnose problems.

---

## Model Mosaic

**Multi-model collaboration within a single logical turn.**

Instead of sending every request to a single model, the ModelMosaic enables patterns where multiple models collaborate. A fast model drafts a response, a quality model polishes it. Or multiple models vote on the best answer. The result is higher quality output without always paying for the most expensive model.

### Collaboration Patterns

| Pattern | How It Works | When to Use |
|---|---|---|
| **Draft-Polish** | Fast model drafts, quality model refines | Cost-effective quality improvement |
| **Plan-Critique-Refine** | Model A plans, Model B critiques, Model A refines | Complex multi-step tasks |
| **Ensemble Vote** | Multiple models answer, best response wins | High-stakes decisions |

### How to Use It

```python
import cortex

engine = cortex.Engine(providers={
    "gemini": {"api_key": "..."},
    "openai": {"api_key": "..."},
})

agent = engine.create_agent(
    name="analyst",
    system_prompt="Provide thorough analysis.",
    mosaic_pattern="draft_polish"  # Enable draft-polish pattern
)

session = agent.start_session(user_id="analyst_1")

# The agent automatically uses draft-polish:
# 1. gemini-3-flash drafts the analysis (fast, cheap)
# 2. gpt-4o polishes and validates (thorough, accurate)
# Result: Quality of gpt-4o at ~60% of the cost
response = await session.run("Analyze our Q4 revenue trends")
```

### Pattern Selection

The agent can automatically select the best pattern based on task complexity:

- **Simple tasks**: Single model (no mosaic overhead)
- **Standard tasks**: Draft-polish (cost-effective quality)
- **Complex tasks**: Plan-critique-refine (thorough reasoning)
- **Critical decisions**: Ensemble vote (maximum reliability)

---

## Speculative Executor

**Pre-computes context for the predicted next step to save time.**

The SpeculativeExecutor predicts what the agent will need to do next and begins assembling the context before the current step completes. When the prediction is correct, the next step executes immediately with pre-warmed context. When wrong, the speculative work is discarded at minimal cost.

### How It Works

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="developer",
    system_prompt="Build software.",
    speculative_execution=True
)

session = agent.start_session(user_id="dev_1")

# During agentic execution, speculation happens automatically:
#
# Step 1: Agent writes auth.py
#   -> Speculator predicts: "Next step will be writing tests for auth.py"
#   -> Pre-assembles: test framework context, auth.py content, test patterns
#
# Step 2: Agent decides to write tests for auth.py (prediction correct!)
#   -> Context is already assembled -- step executes ~40% faster
#
# If prediction was wrong (e.g., agent writes routes.py instead):
#   -> Speculative context is discarded, no harm done
```

### Speculation Confidence

The executor only speculates when confidence is high enough to justify the compute cost:

| Confidence | Action |
|---|---|
| Above 60% | Speculate and pre-compute |
| Below 60% | Skip speculation, wait for actual decision |

The system tracks accuracy over time and adjusts its confidence calibration.

---

## Decision Log

**Complete audit trail for every branch point in agent execution.**

The DecisionLog records every decision the agent makes: which model was selected, which tool was called, which plan was chosen. For each decision, it captures the alternatives that were considered, the reasoning behind the choice, and the eventual outcome.

### Why It Matters

When an agent produces an unexpected result, you need to trace the decision chain that led to it. The DecisionLog enables:

- **Post-hoc analysis**: "Why did the agent choose tool X over tool Y?"
- **Compliance auditing**: "What data did the agent access and why?"
- **What-if reasoning**: "What would have happened with the alternative choice?"
- **Debugging**: "At which decision point did the agent go wrong?"

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="support",
    system_prompt="Help customers.",
    decision_logging=True
)

session = agent.start_session(user_id="user_1")
response = await session.run("Refund my last order")

# Review the decision log
decisions = session.get_decision_log()

for decision in decisions:
    print(f"Step {decision.step}: {decision.decision_type}")
    print(f"  Chosen: {decision.chosen_action}")
    print(f"  Alternatives: {decision.alternatives}")
    print(f"  Reasoning: {decision.reasoning}")
    print(f"  Confidence: {decision.confidence:.2%}")
    print(f"  Outcome: {decision.outcome}")

# Example output:
# Step 1: tool_selection
#   Chosen: lookup_order
#   Alternatives: [search_customer, check_refund_policy]
#   Reasoning: User mentioned "last order" - direct order lookup is most efficient
#   Confidence: 91%
#   Outcome: success - found order #12345
#
# Step 2: tool_selection
#   Chosen: process_refund
#   Alternatives: [escalate_to_human, check_refund_policy]
#   Reasoning: Order found, within refund window, amount under auto-approve threshold
#   Confidence: 87%
#   Outcome: success - refund processed
```

### Exporting Decision Logs

```python
# Export for analysis
log_json = session.export_decision_log(format="json")

# Export for compliance
log_csv = session.export_decision_log(format="csv")
```

---

## Progress Estimator

**Velocity-based progress prediction with trend analysis.**

The ProgressEstimator tracks how fast the agent is making progress toward its goal and predicts when it will finish. It detects stalls, acceleration, and deceleration patterns, enabling proactive intervention before the agent wastes resources.

### How to Use It

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="developer",
    system_prompt="Build software.",
    progress_estimation=True
)

session = agent.start_session(user_id="dev_1")

# During agentic execution, progress is tracked automatically
response = await session.run_agentic("Build a REST API with auth")

# Check progress during execution
progress = session.get_progress()

print(f"Completed: {progress.percent_complete:.0%}")
print(f"Status: {progress.status}")  # on_track, at_risk, or stalled
print(f"Velocity: {progress.velocity:.1%} per step")
print(f"Estimated steps remaining: {progress.steps_remaining}")
print(f"Estimated time remaining: {progress.time_remaining_s:.0f}s")

# Example output:
# Completed: 45%
# Status: on_track
# Velocity: 5.2% per step
# Estimated steps remaining: 11
# Estimated time remaining: 45s
```

### Progress Status

| Status | Meaning | Automatic Response |
|---|---|---|
| **On Track** | Velocity is healthy, ETA is within budget | Continue normally |
| **At Risk** | Velocity declining or approaching budget limit | Log warning, increase monitoring |
| **Stalled** | No progress for multiple steps | Trigger drift engine, consider replanning |

---

## A/B Test Manager

**Statistically rigorous comparison of models, prompts, and strategies.**

The ABTestManager runs controlled experiments to determine which model, prompt, or strategy performs better for your specific workload. It uses the Mann-Whitney U test for statistical significance, ensuring you make data-driven decisions rather than relying on intuition.

### How to Use It

```python
import cortex

engine = cortex.Engine()

# Create an A/B test comparing two models
test = engine.create_ab_test(
    name="orchestrator_comparison",
    variant_a={"model": "gemini-3-pro-preview"},
    variant_b={"model": "gpt-4o"},
    metric="quality_score",
    min_samples=100,
    significance_level=0.05
)

# Sessions are automatically assigned to variants
session = agent.start_session(user_id="user_1")
# This session uses variant A (gemini-3-pro)

session2 = agent.start_session(user_id="user_2")
# This session uses variant B (gpt-4o)

# After enough samples, check results
results = test.get_results()

print(f"Status: {results.status}")
print(f"Variant A mean: {results.variant_a_mean:.3f}")
print(f"Variant B mean: {results.variant_b_mean:.3f}")
print(f"Winner: {results.winner}")
print(f"P-value: {results.p_value:.4f}")
print(f"Significant: {results.is_significant}")

# Example output:
# Status: concluded
# Variant A mean: 0.847
# Variant B mean: 0.812
# Winner: variant_a
# P-value: 0.0234
# Significant: True
```

### What You Can Test

| Experiment Type | Example |
|---|---|
| **Model comparison** | gemini-3-pro vs. gpt-4o for your workload |
| **Prompt variants** | System prompt A vs. system prompt B |
| **Temperature settings** | temperature=0.3 vs. temperature=0.7 |
| **Mosaic patterns** | draft-polish vs. single model |
| **Context strategies** | Aggressive compression vs. conservative |

---

## Provider Health Monitor

**Real-time health tracking with circuit breaker for LLM providers.**

The ProviderHealthMonitor tracks success rates, latency percentiles, and error patterns for each LLM provider using sliding windows. When a provider becomes unhealthy, a circuit breaker opens to prevent cascading failures, automatically routing requests to healthy alternatives.

### How It Works

```python
import cortex

engine = cortex.Engine(providers={
    "gemini": {"api_key": "..."},
    "openai": {"api_key": "..."},
    "anthropic": {"api_key": "..."},
})

# Health monitoring is automatic
# Check provider health at any time
health = engine.get_provider_health()

for provider, status in health.items():
    print(f"{provider}:")
    print(f"  Success rate: {status.success_rate:.1%}")
    print(f"  Avg latency: {status.avg_latency_ms:.0f}ms")
    print(f"  P99 latency: {status.p99_latency_ms:.0f}ms")
    print(f"  Circuit breaker: {status.circuit_state}")

# Example output:
# gemini:
#   Success rate: 99.2%
#   Avg latency: 450ms
#   P99 latency: 1200ms
#   Circuit breaker: closed (healthy)
# openai:
#   Success rate: 78.5%
#   Avg latency: 2100ms
#   P99 latency: 8500ms
#   Circuit breaker: open (unhealthy - requests routed elsewhere)
```

### Circuit Breaker States

| State | Meaning | Behavior |
|---|---|---|
| **Closed** | Provider is healthy | All requests pass through normally |
| **Open** | Provider has failed repeatedly | All requests are routed to alternatives |
| **Half-Open** | Testing if provider has recovered | A small number of probe requests are sent |

The circuit breaker opens after consecutive failures and automatically attempts recovery with probe requests after a cooldown period.

---

## Integration

All agent intelligence modules work together within the session:

```python
import cortex

engine = cortex.Engine()
agent = engine.create_agent(
    name="developer",
    system_prompt="Build production software.",
    mosaic_pattern="auto",         # Automatic pattern selection
    speculative_execution=True,    # Pre-compute next steps
    decision_logging=True,         # Full decision audit trail
    progress_estimation=True,      # Track velocity and ETA
)

session = agent.start_session(user_id="dev_1")
response = await session.run_agentic("Build a REST API with auth")

# All modules contribute:
# 1. ModelMosaic selects the optimal collaboration pattern per step
# 2. SpeculativeExecutor pre-warms context for the next step
# 3. DecisionLog records every choice with reasoning
# 4. ProgressEstimator tracks velocity and predicts completion
# 5. ProviderHealthMonitor ensures requests go to healthy providers
```

---

## See Also

- [Intelligent Model Routing](model-routing.md) - ModelRegistry, CognitiveClassifier, CostTracker
- [Cognitive Context Pipeline](cognitive-context.md) - Context quality and state management
- [Anti-Drift System](anti-drift.md) - Loop detection and drift recovery
- [Observability](observability.md) - Decision tracing and metrics collection
