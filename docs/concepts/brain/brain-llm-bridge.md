# Brain-LLM Bridge: Closing the Prompt Gap

## Overview

The Brain-LLM Bridge is corteX's solution to a critical challenge in AI agent architectures: **the prompt gap**. While the brain engine computes extensive cognitive state (behavioral weights, goal progress, attention shifts, prediction surprise, calibration health), this state must be communicated to the LLM for it to influence actual behavior. Without this bridge, the brain operates in a parallel universe from the LLM.

The bridge consists of two complementary systems:

1. **BrainStateInjector** - Compiles brain state into natural language context injected into the LLM prompt
2. **BrainParameterResolver** - Maps cognitive state to LLM API parameters (temperature, top_p, max_tokens, etc.)

Together, these systems ensure that every brain computation becomes actionable LLM context.

## The Prompt Gap Problem

Before the Brain-LLM Bridge, corteX faced a fundamental disconnect:

```python
# Brain computes extensive state
weights.behavioral.weights["verbosity"] = 0.8
weights.behavioral.weights["creativity"] = -0.3
goal_tracker.progress = 0.45
attention.detect_drift = True
calibration.ece["tool_success"] = 0.22

# But LLM generation didn't see any of it
response = await router.generate(
    messages=messages,
    tools=tools,
    # No brain state passed! LLM is blind to the brain.
)
```

The brain tracked behavioral preferences, detected goal drift, identified attention changes, and monitored calibration health - but none of this reached the LLM. The result: sophisticated control systems steering a black box.

## BrainStateInjector: Natural Language Context

The `BrainStateInjector` solves the prompt gap by compiling brain state into structured natural language that augments the system prompt.

### What It Compiles

The injector processes eight types of brain state:

| Brain Component | Injected Context |
|----------------|------------------|
| **Behavioral Weights** | Style instructions (verbosity, formality, creativity, initiative) |
| **Functional Column** | Active processing mode and specialization |
| **Goal State** | Goal description, progress percentage, drift warnings |
| **Attention Changes** | Context change highlights (topic shifts, quality drift) |
| **Calibration Warnings** | ECE alerts, metacognition signals (oscillation, stagnation) |
| **Prediction Context** | Recent surprise levels, predicted outcomes |
| **Active Concepts** | Top-activated concepts from the concept graph |
| **Proactive Predictions** | Anticipated next user actions |

### Example: Behavioral Weights to Natural Language

The injector translates numerical weights into actionable instructions:

```python
# Brain state
behavioral_weights = {
    "verbosity": 0.5,      # User wants detailed responses
    "formality": -0.4,     # User prefers casual tone
    "creativity": 0.6,     # User wants novel approaches
    "initiative": 0.8,     # User wants proactive suggestions
}

# Injected context
"""
## Brain Context

### Style
- Provide detailed, thorough responses.
- Use a casual, conversational tone.
- Be creative and explore novel approaches.
- Be proactive: suggest next steps and improvements.
"""
```

Each weight has thresholds that trigger specific instructions. For example:

- `verbosity >= 0.3` → "Provide detailed, thorough responses."
- `verbosity <= -0.3` → "Be concise and direct."
- `creativity >= 0.3` → "Be creative and explore novel approaches."
- `autonomy >= 0.7` → "Act independently; make decisions without asking."

### Example: Goal State Warnings

When goal tracking detects problems, the injector adds warnings:

```python
# Brain state
goal = "Build a REST API for user authentication"
progress = 0.45
drift = 0.35
loop_detected = False

# Injected context
"""
### Goal: Build a REST API for user authentication
Progress: 45% | Drift: 0.35
CAUTION: Mild drift detected. Stay on track.
"""
```

If drift exceeds 0.5 or a loop is detected, warnings escalate to "WARNING" with explicit instructions to refocus or break the pattern.

### Example: Calibration Warnings

Calibration issues trigger direct instructions:

```python
# Brain state: Tool success predictions are overconfident
ece_scores = {"tool_success": 0.18}

# Injected context
"""
### Calibration
- Calibration warning (tool_success): Recent predictions unreliable (ECE=0.18).
  Double-check your reasoning.
"""
```

### Token Budget Management

The injector respects a token budget (default 500 tokens ≈ 2000 characters). If brain context exceeds the budget, it truncates gracefully:

1. Truncate at section boundaries (preserves complete sections)
2. Prioritize earlier sections (behavioral weights, column mode, goal state)
3. Append `[...truncated for token budget]` if truncation occurs

## BrainParameterResolver: API Parameter Mapping

While the injector communicates *what* to do, the resolver controls *how* the LLM explores the solution space by mapping cognitive state to API parameters.

### The Neuroscience Rationale

The brain does not use a fixed "temperature" when making decisions. Depending on surprise (dopamine), confidence (prefrontal calibration), attention priority (thalamic gating), creativity drive (divergent thinking), and active cortical column specialization, the brain modulates its exploration-exploitation tradeoff in real time.

The `BrainParameterResolver` implements this principle for LLM APIs.

### Resolved Parameters

The resolver maps brain state to eight LLM parameters:

| Parameter | Brain Signals | Purpose |
|-----------|---------------|---------|
| **temperature** | Process type, surprise, confidence, attention, creativity, task ceiling | Exploration vs. exploitation tradeoff |
| **top_p** | Process type (System1=0.85, System2=0.95) | Nucleus sampling threshold |
| **top_k** | Column overrides (Gemini only) | Top-k sampling threshold |
| **max_tokens** | Attention priority, resource budget, verbosity | Response length ceiling |
| **frequency_penalty** | Creativity weight (OpenAI only) | Penalize token repetition |
| **presence_penalty** | Surprise level (OpenAI only) | Encourage new vocabulary |
| **thinking_budget** | Process type, calibration health | Reasoning token budget (for o1-style models) |
| **seed** | Process type, surprise (deterministic when System1 + low surprise) | Reproducibility control |

### Temperature Resolution: Multi-Signal Fusion

Temperature is resolved through a multi-stage pipeline:

```python
# Stage 1: Base from Dual-Process Router
if process_type == "system1":
    base_temp = 0.2  # Fast, exploit known patterns
elif process_type == "system2":
    base_temp = 0.6  # Slow, explore alternatives
else:
    base_temp = 0.4  # Neutral

# Stage 2: Surprise modulation (+0.0 to +0.3)
# High surprise = unexpected outcome = explore more
surprise_boost = surprise * 0.3

# Stage 3: Confidence modulation (+0.0 to +0.2)
# Low confidence = uncertain = explore more
confidence_boost = (1.0 - confidence) * 0.2

# Stage 4: Attention modulation
if attention_priority == "critical":
    attention_adj = -0.1  # Focus, reduce exploration
elif attention_priority == "subconscious":
    attention_adj = -0.15  # Routine, minimal exploration
else:
    attention_adj = 0.0

# Stage 5: Creativity from behavioral weights (±0.15)
creativity_delta = creativity_weight * 0.15

# Combine
temp = base_temp + surprise_boost + confidence_boost + attention_adj + creativity_delta

# Stage 6: Task ceiling
temp = min(temp, TASK_CEILING[task_type])
# coding=0.5, validation=0.3, planning=0.9, conversation=1.0

# Stage 7: Clamp to valid range
temp = max(0.0, min(2.0, temp))
```

### Example: System 1 vs System 2 Parameters

```python
# System 1: Fast, confident, routine task
resolve(
    process_type="system1",
    surprise=0.05,
    confidence=0.85,
    attention_priority="foreground",
    creativity=0.0,
)
# → temperature=0.2, top_p=0.85, seed=42 (deterministic)

# System 2: Slow, uncertain, novel task
resolve(
    process_type="system2",
    surprise=0.7,
    confidence=0.4,
    attention_priority="critical",
    creativity=0.5,
)
# → temperature=0.85, top_p=0.95, thinking_budget=4096
```

### Max Tokens: Attention + Resources + Verbosity

Max tokens combines three signals:

1. **Attention priority budgets**:
   - CRITICAL: 8192 tokens
   - FOREGROUND: 4096 tokens
   - BACKGROUND: 2048 tokens
   - SUBCONSCIOUS: 1024 tokens
   - SUPPRESSED: 256 tokens

2. **Resource allocation**: `ResourceHomunculus.token_budget * 4096`

3. **Verbosity scaling**: `base_tokens * (1.0 + verbosity * 0.5)`

The final value is the minimum of attention and resource budgets, scaled by verbosity.

### Thinking Budget: Calibration-Aware Reasoning

For reasoning models (like OpenAI's o1), the resolver allocates thinking tokens based on calibration health:

```python
if process_type == "system2":
    if calibration_health == "critical":
        thinking_budget = 8192  # Need deep reasoning to recover
    elif calibration_health == "warning":
        thinking_budget = 4096
    else:
        thinking_budget = 2048  # Healthy, normal budget
```

### Gemini 3 Special Handling

Gemini 3 models (gemini-3-pro-preview, gemini-3-flash-preview) ignore the temperature parameter and always use 1.0. The resolver detects Gemini 3 models and forces `temperature=1.0` to avoid confusion.

## Priority System: Resolving Conflicts

Multiple systems can attempt to set the same parameter. The Brain-LLM Bridge uses a strict priority hierarchy:

1. **Modulator CLAMP** (highest) - `TargetedModulator` hard overrides for experiments
2. **Column override** - Active `FunctionalColumn` weight overrides
3. **Gemini 3 forced temperature** - Provider-specific constraints
4. **Brain state computation** - Multi-signal resolution (default)
5. **Task default** (lowest) - Fallback values

Example:

```python
# Column override sets temperature
active_column.weight_overrides["temperature"] = 0.3

# Brain computes temperature=0.7 from signals
# But column override wins → final temperature=0.3

# Unless modulator CLAMP is active
modulator.clamp("temperature", 0.9, turns=5)
# → final temperature=0.9 (modulator beats column)
```

## Integration in Session.run()

The Brain-LLM Bridge activates automatically in every `Session.run()` call:

```python
async def run(self, message: str) -> Response:
    # 1. Collect brain state
    attention_result = self.attention.process_turn(message, context)
    active_column = self.columns.select_column(task_type, message)
    active_concepts = self.concepts.activate([task_type] + tool_names)
    proactive_pred = self.proactive.predict_next_turn()

    # 2. Brain State Injection: compile brain context
    system_with_brain = self._build_brain_augmented_prompt(
        attention_result=attention_result,
        active_column=active_column,
        active_concepts=active_concepts,
        proactive_pred=proactive_pred,
    )

    # 3. Brain Parameter Resolution: map cognitive state to LLM params
    param_bundle = self._resolve_brain_parameters(
        task_type=task_type,
        process_type=process_type,
        attention_priority=attention_priority,
        active_column=active_column,
        resource_alloc=resource_alloc,
    )

    # 4. Generate with brain-controlled parameters
    response = await self.agent.engine.router.generate(
        messages=self._messages,
        tools=tool_defs,
        role=role,
        system_instruction=system_with_brain,  # Injected brain context
        temperature=param_bundle.temperature,
        top_p=param_bundle.top_p,
        top_k=param_bundle.top_k,
        max_tokens=param_bundle.max_tokens,
        frequency_penalty=param_bundle.frequency_penalty,
        presence_penalty=param_bundle.presence_penalty,
        thinking_budget=param_bundle.thinking_budget,
    )
```

## Brain Component → LLM Parameter Mapping Table

| Brain Component | Influences | How |
|----------------|-----------|-----|
| **DualProcessRouter** | temperature, top_p, thinking_budget | System1=low temp/tight top_p, System2=higher temp/wide top_p |
| **PredictionEngine** | temperature, presence_penalty, seed | High surprise → higher temp, more exploration |
| **CalibrationEngine** | temperature, thinking_budget | Low confidence → higher temp, more thinking tokens |
| **AttentionSystem** | temperature, max_tokens | Critical → lower temp (focus); priority → token budget |
| **BehavioralWeights** | temperature, max_tokens, frequency_penalty | creativity → temp, verbosity → max_tokens |
| **ResourceHomunculus** | max_tokens | token_budget → absolute ceiling |
| **FunctionalColumn** | ALL (via overrides) | Column can override any parameter |
| **TargetedModulator** | ALL (via CLAMP) | Modulator can hard-clamp any parameter |

## Observability

The resolver tracks which signals contributed to each parameter decision:

```python
# After resolution
stats = param_resolver.get_stats()
print(stats)
```

Output:

```python
{
    "signals": {
        "dual_process_base": 0.2,
        "surprise_boost": 0.15,
        "confidence_boost": 0.08,
        "attention_adjustment": -0.1,
        "creativity_delta": 0.05,
        "combined_raw": 0.38,
        "task_ceiling": 0.5,
        "temperature_final": 0.38,
    },
    "decisions": {
        "temperature": "brain_state_computation",
        "top_p": "system1_default",
        "max_tokens": "attention_resource_verbosity",
    }
}
```

This enables full traceability: you can see exactly why the system chose `temperature=0.38` (System1 base + surprise + confidence - attention + creativity, capped at task ceiling).

## Best Practices

### Do: Trust the Brain's Decisions

The resolver combines signals from 8+ brain components. Manual temperature overrides bypass this intelligence:

```python
# ❌ Don't override unless you have a specific reason
response = await router.generate(messages, temperature=0.7)  # Ignores brain state

# ✅ Let the brain decide
response = await session.run(message)  # Brain controls temperature
```

### Do: Use Modulators for Experiments

If you need to override parameters for testing:

```python
# ✅ Use modulator CLAMP (temporary, observable)
session.modulator.clamp("temperature", 0.9, turns=5, reason="Testing high creativity")
response = await session.run(message)

# After 5 turns, CLAMP expires and brain resumes control
```

### Do: Configure Columns for Persistent Specialization

If a task type always needs specific parameters:

```python
# ✅ Create a specialized column with overrides
coding_column = FunctionalColumn(
    name="coding",
    weight_overrides={
        "temperature": 0.3,  # Low exploration for code
        "code_density": 0.9,
        "verbosity": -0.2,
    }
)
session.columns.register_column(coding_column)
```

### Don't: Hardcode Provider-Specific Parameters

The resolver handles provider differences automatically:

```python
# ❌ Don't write provider-specific code
if provider == "openai":
    response = await router.generate(messages, frequency_penalty=0.5)
elif provider == "gemini":
    response = await router.generate(messages, top_k=40)

# ✅ Use the resolver (automatically filters by provider)
param_bundle = param_resolver.resolve(...)
kwargs = param_bundle.to_provider_kwargs(provider)
response = await router.generate(messages, **kwargs)
```

## Summary

The Brain-LLM Bridge closes the prompt gap through two complementary mechanisms:

1. **BrainStateInjector** compiles cognitive state into natural language context that tells the LLM *what* to prioritize
2. **BrainParameterResolver** maps cognitive state to API parameters that control *how* the LLM explores solutions

Together, they transform corteX from "a sophisticated control system steering a black box" into an integrated cognitive architecture where wrapper-brain and LLM-brain cooperate as a unified whole.

Every brain computation - from weight updates to prediction surprise to calibration warnings - now influences LLM behavior through explicit, traceable mechanisms.
