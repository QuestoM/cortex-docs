# How to Configure Brain-to-LLM Parameter Mapping

This guide shows you how to customize how corteX's brain components map to LLM API parameters (temperature, top_p, max_tokens, etc.).

## Understanding the Default Behavior

By default, corteX automatically resolves LLM parameters from brain state:

```python
import cortex

engine = cortex.Engine(providers={"openai": {"api_key": "sk-..."}})
agent = engine.create_agent(name="assistant")
session = agent.start_session(user_id="user_123")

# Brain automatically controls parameters
response = await session.run("Write a creative story about a robot")
# → High creativity weight → temperature ≈ 0.65
# → System2 process → top_p = 0.95
# → Verbosity = 0.0 → max_tokens ≈ 4096
```

The parameter resolver combines signals from:

- DualProcessRouter (System1 vs System2)
- PredictionEngine (surprise level)
- CalibrationEngine (confidence)
- AttentionSystem (priority level)
- BehavioralWeights (creativity, verbosity)
- ResourceHomunculus (token budget)

## Customizing Temperature Computation

### Via Behavioral Weights

The simplest way to influence temperature is through behavioral weights:

```python
# Initialize with high creativity
agent = engine.create_agent(
    name="creative_writer",
    weight_config=cortex.WeightConfig(
        verbosity=0.5,
        creativity=0.8,  # High creativity → higher temperature
        autonomy=0.7,
    )
)

session = agent.start_session()
response = await session.run("Write a poem")
# temperature ≈ 0.6 + (0.8 * 0.15) = 0.72
```

### Via Task-Specific Ceilings

Each task type has a built-in temperature ceiling:

```python
TASK_CEILING = {
    "coding": 0.5,        # Low exploration for code
    "validation": 0.3,    # Very low for checking
    "tool_use": 0.4,
    "planning": 0.9,      # High for brainstorming
    "conversation": 1.0,
    "summarization": 0.6,
    "reasoning": 0.7,
    "debugging": 0.5,
    "testing": 0.4,
    "research": 0.8,
}
```

The task classifier automatically applies these ceilings. You can influence classification through keywords:

```python
# This will hit the "coding" ceiling (0.5)
response = await session.run("Write a function to parse JSON")

# This will hit the "planning" ceiling (0.9)
response = await session.run("Plan an architecture for a microservices system")
```

### Via Runtime Weight Override

For one-off adjustments during a session:

```python
session = agent.start_session()

# Temporarily boost creativity for one turn
session.override_weight("creativity", 1.0)
response = await session.run("Generate 10 product name ideas")

# Weight persists until changed
session.override_weight("creativity", 0.0)
response = await session.run("Validate the JSON schema")
```

### Via Targeted Modulation (CLAMP)

For temporary hard overrides (useful for experiments):

```python
session = agent.start_session()

# CLAMP temperature to 0.9 for exactly 5 turns
session.modulator.clamp(
    "temperature",
    value=0.9,
    turns=5,
    reason="Testing high-temperature exploration"
)

# Next 5 turns use temperature=0.9 regardless of brain state
for i in range(5):
    response = await session.run(f"Idea #{i+1}")

# Turn 6: CLAMP expires, brain resumes control
response = await session.run("Continue")
```

!!! warning "Modulator vs Weight Override"
    - `modulator.clamp()` has highest priority and expires automatically
    - `override_weight()` affects the computation but can be overridden by modulators
    - CLAMPs are temporary and observable; weight overrides persist until changed

## Adding Column-Level Parameter Overrides

Functional columns can specify persistent parameter overrides for their specialization:

```python
from corteX.engine.columns import FunctionalColumn

# Create a specialized column for code review
code_review_column = FunctionalColumn(
    name="code_review",
    specialization="static_analysis_and_security",
    preferred_tools=["linter", "security_scanner", "complexity_analyzer"],
    preferred_model_tier="orchestrator",  # Always use best model
    weight_overrides={
        "temperature": 0.2,      # Very low exploration (deterministic)
        "top_p": 0.85,           # Tight nucleus sampling
        "verbosity": 0.6,        # Detailed explanations
        "formality": 0.8,        # Professional tone
        "code_density": 0.9,     # Include code snippets
    }
)

# Register the column
session.columns.register_column(code_review_column)

# When this column activates, its overrides apply
response = await session.run("Review this function for security issues")
# → Column detects "review" + "security" → activates code_review column
# → temperature=0.2, verbosity=0.6 from column overrides
```

Column overrides have priority over brain state computation but lower than modulator CLAMPs.

## Customizing Max Tokens

Max tokens combines three signals: attention priority, resource allocation, and verbosity.

### Via Verbosity Weight

```python
# Terse responses
agent = engine.create_agent(
    name="concise_assistant",
    weight_config=cortex.WeightConfig(verbosity=-0.5)
)
session = agent.start_session()
response = await session.run("Explain async/await")
# max_tokens ≈ 4096 * (1.0 + (-0.5 * 0.5)) = 3072

# Verbose responses
agent = engine.create_agent(
    name="detailed_assistant",
    weight_config=cortex.WeightConfig(verbosity=0.8)
)
session = agent.start_session()
response = await session.run("Explain async/await")
# max_tokens ≈ 4096 * (1.0 + (0.8 * 0.5)) = 5734
```

### Via Resource Allocation

The `ResourceHomunculus` allocates token budgets based on task type usage patterns:

```python
# Check current allocation for a task type
resource_map = session.get_resource_map()
print(resource_map["allocations"]["coding"])
# → {"token_budget": 1.2, "model_tier": "orchestrator", ...}

# token_budget=1.2 means 1.2 * 4096 = 4915 tokens max
```

Resource allocation adapts automatically based on:
- Frequency (how often the task type is used)
- Criticality (how important the task type is)
- Quality sensitivity (how much quality varies for this task type)

You don't typically override this manually - it learns from usage patterns.

### Via Attention Priority

Attention classification sets hard ceilings:

```python
ATTENTION_TOKEN_BUDGETS = {
    "critical": 8192,      # Emergency, complex problems
    "foreground": 4096,    # Normal tasks
    "background": 2048,    # Lower-priority tasks
    "subconscious": 1024,  # Routine, habituated tasks
    "suppressed": 256,     # Fully habituated, minimal processing
}
```

The attention system automatically classifies messages. You can influence this through explicit urgency keywords:

```python
# This triggers CRITICAL attention priority
response = await session.run("URGENT: Production is down, debug immediately")
# → max_tokens = 8192 (critical ceiling)

# This triggers SUBCONSCIOUS (routine)
response = await session.run("Thanks")
# → max_tokens = 1024 (subconscious ceiling)
```

## Disabling Brain Parameter Resolution

For testing or debugging, you can bypass the brain and use manual parameters:

```python
# Option 1: Use the router directly (bypasses Session)
response = await engine.router.generate(
    messages=[{"role": "user", "content": "Hello"}],
    role="orchestrator",
    temperature=0.7,  # Manual override
    max_tokens=2000,
)

# Option 2: Use modulator CLAMPs to override specific parameters
session.modulator.clamp("temperature", 0.7, turns=999)
session.modulator.clamp("max_tokens", 2000, turns=999)
response = await session.run("Hello")
```

!!! warning "Bypassing the Brain"
    Using manual parameters disables adaptive learning. The brain cannot track outcomes or adjust parameters based on feedback. Only use this for debugging.

## Customizing Frequency and Presence Penalties (OpenAI)

These parameters are OpenAI-specific and map directly from brain state:

### Frequency Penalty: Creativity Mapping

```python
# High creativity → low frequency penalty (allow repetition of interesting tokens)
# Low creativity → high frequency penalty (force variety)

agent = engine.create_agent(
    name="creative",
    weight_config=cortex.WeightConfig(creativity=0.8)
)
session = agent.start_session()
response = await session.run("Write a story")
# frequency_penalty = 0.3 + (0.8 * 0.6) = 0.78

agent = engine.create_agent(
    name="conservative",
    weight_config=cortex.WeightConfig(creativity=-0.6)
)
session = agent.start_session()
response = await session.run("Write a report")
# frequency_penalty = max(0, 0.3 + (-0.6 * 0.6)) = 0.0
```

### Presence Penalty: Surprise Mapping

```python
# High surprise → high presence penalty (encourage new vocabulary)
# Low surprise → low presence penalty (stick to known patterns)

# This is automatic based on prediction engine surprise
# No manual configuration needed
```

## Configuring Thinking Budget (Reasoning Models)

For reasoning models like OpenAI's o1:

```python
# Thinking budget scales with calibration health
# Critical health → 8192 thinking tokens (need deep reasoning to recover)
# Warning health → 4096 thinking tokens
# Healthy → 2048 thinking tokens

# Automatic - calibration engine determines health
# You can influence by triggering System2:

response = await session.run("This is a complex multi-step reasoning problem")
# → DualProcessRouter escalates to System2
# → thinking_budget = 4096 (assuming healthy calibration)
```

To force a thinking budget:

```python
session.modulator.clamp("thinking_budget", 8192, turns=1)
response = await session.run("Solve this difficult proof")
```

## Provider-Specific Parameter Handling

The resolver automatically adapts to provider capabilities:

```python
# OpenAI: supports temperature, top_p, max_tokens, frequency_penalty,
#         presence_penalty, seed
param_bundle = resolver.resolve(provider=ProviderType.OPENAI, ...)
kwargs = param_bundle.to_provider_kwargs(ProviderType.OPENAI)
# → includes frequency_penalty, presence_penalty

# Gemini: supports temperature, top_p, top_k, max_tokens
param_bundle = resolver.resolve(provider=ProviderType.GEMINI, ...)
kwargs = param_bundle.to_provider_kwargs(ProviderType.GEMINI)
# → includes top_k, excludes frequency_penalty

# Gemini 3: forced temperature=1.0
param_bundle = resolver.resolve(
    provider=ProviderType.GEMINI,
    model="gemini-3-pro-preview",
    ...
)
# → temperature=1.0 (forced), ignores all other temp signals
```

## Observability: Understanding Parameter Decisions

Track which signals influenced each parameter:

```python
session = agent.start_session()
response = await session.run("Write a function")

# Get parameter resolver stats
stats = session._param_resolver.get_stats()
print(stats)
```

Output:

```json
{
  "signals": {
    "dual_process_base": 0.2,
    "surprise_boost": 0.05,
    "confidence_boost": 0.04,
    "attention_adjustment": 0.0,
    "creativity_delta": 0.02,
    "combined_raw": 0.31,
    "task_ceiling": 0.5,
    "temperature_final": 0.31,
    "attention_tokens": 4096,
    "resource_tokens": 4915,
    "verbosity_multiplier": 1.0,
    "max_tokens_final": 4096
  },
  "decisions": {
    "temperature": "brain_state_computation",
    "top_p": "system1_default",
    "max_tokens": "attention_resource_verbosity",
    "frequency_penalty": "creativity_mapped",
    "thinking_budget": "system1_no_thinking"
  }
}
```

This shows:
- Temperature started at 0.2 (System1), added surprise/confidence/creativity, capped at task ceiling (0.5 for coding)
- Final temperature: 0.31
- Max tokens: attention ceiling (4096) was lower than resource ceiling (4915)
- Frequency penalty mapped from creativity weight

## Complete Example: Custom Parameter Strategy

```python
import cortex

# 1. Create engine with Gemini
engine = cortex.Engine(
    providers={"gemini": {"api_key": "AIza..."}},
    orchestrator_model="gemini-2.5-pro",
    worker_model="gemini-2.5-flash",
)

# 2. Create agent with custom weights
agent = engine.create_agent(
    name="code_assistant",
    system_prompt="You are an expert Python developer.",
    weight_config=cortex.WeightConfig(
        verbosity=0.3,       # Moderately detailed
        creativity=-0.2,     # Prefer conventional patterns for code
        autonomy=0.6,        # Proactive but not too aggressive
        formality=0.4,       # Professional but approachable
    ),
)

# 3. Create specialized columns
from corteX.engine.columns import FunctionalColumn

# Column 1: Code generation (low temperature, high precision)
code_gen_column = FunctionalColumn(
    name="code_generation",
    specialization="python_implementation",
    weight_overrides={
        "temperature": 0.25,
        "code_density": 0.9,
        "verbosity": 0.2,
    }
)

# Column 2: Architecture design (high temperature, exploration)
arch_column = FunctionalColumn(
    name="architecture_design",
    specialization="system_design",
    weight_overrides={
        "temperature": 0.8,
        "creativity": 0.7,
        "verbosity": 0.6,
    }
)

# 4. Start session and register columns
session = agent.start_session(user_id="dev_123")
session.columns.register_column(code_gen_column)
session.columns.register_column(arch_column)

# 5. Use the session
# This activates code_generation column → temperature=0.25
response = await session.run("Implement a binary search function")

# This activates architecture_design column → temperature=0.8
response = await session.run("Design a microservices architecture for e-commerce")

# 6. Override for a specific turn
session.modulator.clamp("verbosity", 1.0, turns=1, reason="Need detailed explanation")
response = await session.run("Explain how async works in Python")

# 7. Inspect parameter decisions
stats = session._param_resolver.get_stats()
print(f"Last temperature decision: {stats['decisions']['temperature']}")
print(f"Contributing signals: {stats['signals']}")
```

## Best Practices

### ✅ Do: Let the Brain Learn

The brain adapts parameters based on outcomes:

```python
# Brain starts with defaults
response1 = await session.run("Implement quick sort")
# → temperature ≈ 0.3

# If this produces high-quality results, brain reinforces low temperature for coding
# If it produces low-quality results, brain may increase exploration

response2 = await session.run("Implement merge sort")
# → temperature adjusted based on outcome of response1
```

### ✅ Do: Use Columns for Persistent Specialization

For consistent parameter sets across related tasks:

```python
# ✅ Create a column with overrides
debug_column = FunctionalColumn(
    name="debugging",
    weight_overrides={"temperature": 0.3, "verbosity": 0.8}
)
session.columns.register_column(debug_column)
```

### ✅ Do: Use Modulators for Experiments

For temporary, observable overrides:

```python
# ✅ Modulator CLAMP for A/B testing
session.modulator.clamp("temperature", 0.9, turns=10, reason="High-temp experiment")
```

### ❌ Don't: Hardcode Temperature in Production

```python
# ❌ Bypasses adaptive learning
response = await engine.router.generate(messages, temperature=0.7)

# ✅ Let the brain decide
response = await session.run(message)
```

### ❌ Don't: Override Every Turn

```python
# ❌ Prevents learning
for msg in messages:
    session.override_weight("creativity", 0.5)
    response = await session.run(msg)

# ✅ Set once and let brain adapt
session.override_weight("creativity", 0.5)
for msg in messages:
    response = await session.run(msg)  # Brain learns and adjusts
```

## Summary

corteX provides four levels of parameter control:

1. **Behavioral weights** (persistent, influences computation)
2. **Column overrides** (persistent, specialization-specific)
3. **Runtime weight overrides** (session-scoped, manual)
4. **Modulator CLAMPs** (temporary, highest priority)

For most use cases, configure behavioral weights at agent creation and let the brain handle the rest. Use columns for specialized processing modes. Use modulators for experiments and debugging.

The parameter resolver ensures that every brain computation - from prediction surprise to calibration health - influences LLM behavior through explicit, traceable mechanisms.
