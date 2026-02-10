# Dual-Process Router

The Dual-Process Router implements Daniel Kahneman's System 1 / System 2 framework for AI agent decision-making. It dynamically routes each decision through either a fast heuristic path or a slow deliberate path, based on real-time assessment of uncertainty, novelty, and risk.

## What It Does

For every decision the agent must make -- which tool to use, which model to call, how to respond -- the router evaluates whether the situation is routine enough for fast processing or requires careful deliberation:

| Path | Speed | Cost | When Used |
|------|-------|------|-----------|
| **System 1** (fast) | Low latency | Cheap model, cached patterns | Familiar tasks, high agreement, low risk |
| **System 2** (slow) | Higher latency | Quality model, full reasoning | Novel tasks, disagreement, errors, high stakes |

## Why: The Decision Theory Inspiration

!!! note "Brain Science: Kahneman's Dual Process Theory"
    Daniel Kahneman's *Thinking, Fast and Slow* describes two modes of cognition:

    - **System 1**: Fast, automatic, effortless. Pattern matching, heuristic-based. Operates unconsciously. Responsible for most of our daily decisions.
    - **System 2**: Slow, deliberate, effortful. Logical reasoning, step-by-step analysis. Engages when System 1 encounters something unexpected.

    The Anterior Cingulate Cortex (ACC) acts as the conflict detector -- when automatic responses conflict with the required task, the ACC triggers System 2 engagement. This is exactly what the DualProcessRouter implements: it monitors for conflict signals and escalates when necessary.

The key insight is that not every agent turn requires the same computational budget. A simple "yes, continue" from the user does not need the same processing as "Actually, I changed my mind -- let's take a completely different approach."

## How It Works

### Escalation Context

The router evaluates seven signals to decide which system to engage:

```python
from corteX.engine.game_theory import DualProcessRouter, EscalationContext

router = DualProcessRouter(
    surprise_threshold=0.6,     # Prediction was wrong
    agreement_threshold=0.4,    # Population evaluators disagree
    novelty_threshold=0.7,      # Unfamiliar task pattern
    safety_threshold=0.8,       # Enterprise risk level
    drift_threshold=0.4,        # Goal tracker reports drift
)

context = EscalationContext(
    surprise_magnitude=0.8,      # High surprise (prediction error)
    population_agreement=0.3,    # Low agreement (evaluators disagree)
    task_novelty=0.2,            # Familiar task
    enterprise_safety=0.1,       # Low risk
    user_explicit_request=False,
    error_in_last_step=False,
    goal_drift=0.1,
)

process = router.route(context)
# ProcessType.SYSTEM2 -- escalated because surprise is high and agreement is low
```

### Escalation Triggers

Any single trigger is sufficient to escalate to System 2:

| Trigger | Threshold | Signal Source |
|---------|-----------|---------------|
| High surprise | > 0.6 | Prediction Engine |
| Low agreement | < 0.4 | Population Decoder |
| High novelty | > 0.7 | Task Classifier |
| High safety | > 0.8 | Enterprise Config |
| User request | boolean | User message ("think carefully") |
| Previous error | boolean | Last step status |
| Goal drift | > 0.4 | Goal Tracker |

### What Each System Does

**System 1 (fast path):**
- Uses `PopulationDecoder` for tool selection
- Uses `ToolPreferenceWeights.get_best_tool()` for quick deterministic picks
- Simpler prompt, lower temperature
- Speed-optimized model (worker tier)

**System 2 (slow path):**
- Full LLM reasoning with `thinking=True`
- `GoalTracker.verify_step()` with LLM verification
- Complex prompt, higher temperature
- Quality-optimized model (orchestrator tier)

### Monitoring the System 2 Ratio

```python
stats = router.get_stats()
# {
#   "system1_count": 45,
#   "system2_count": 12,
#   "system2_ratio": 0.21,
#   "total_decisions": 57,
# }

# If system2_ratio is too high, the agent is over-thinking
# If too low, it may be missing edge cases
```

### Integration with the Attentional Filter

The DualProcessRouter works in tandem with the `AttentionalFilter`. The attentional filter classifies the *incoming message* priority (CRITICAL, FOREGROUND, BACKGROUND, SUBCONSCIOUS). The dual-process router classifies the *decision-making process* for that message. Together, they determine both *how much attention* and *how much reasoning* each turn receives.

## When It Activates

The Dual-Process Router is consulted at every decision point in the agent loop:

1. **Before tool selection**: Should we use quick weight lookup (System 1) or Thompson Sampling with full evaluator ensemble (System 2)?
2. **Before model routing**: Should we use the worker model (System 1) or the orchestrator (System 2)?
3. **Before replanning**: Should we continue on the current plan (System 1) or replan from scratch (System 2)?
4. **After errors**: Errors always trigger System 2 for the recovery step.

## API Reference

```python
from corteX.engine.game_theory import (
    DualProcessRouter,
    ProcessType,
    EscalationContext,
)
```
