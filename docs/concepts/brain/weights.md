# Weight Engine

The Weight Engine is the central nervous system of adaptive behavior in corteX. It maintains seven categories of synaptic-like weights that collectively control how the agent behaves, which tools it prefers, which models it routes to, and how it balances exploration against exploitation.

## What It Does

The Weight Engine tracks learned preferences across seven dimensions, each operating at a different timescale and scope:

| Category | Timescale | Scope | Brain Analogy |
|----------|-----------|-------|---------------|
| **Behavioral** | Per-turn | Session | Fast synaptic transmission |
| **Tool Preference** | Per-execution | Cross-session | Procedural memory (motor cortex) |
| **Model Selection** | Per-generation | Cross-session | Associative memory |
| **Goal Alignment** | Per-goal | Session | Prefrontal goal maintenance |
| **User Insights** | Cross-session | User | Hippocampal declarative memory |
| **Enterprise** | Admin-set | Organization | Prefrontal executive control |
| **Global** | Opt-in | All deployments | Cultural/collective knowledge |

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Synaptic Weights"
    In the biological brain, learning is encoded as changes in synaptic strength -- the efficiency with which one neuron activates another. There are approximately 100 trillion synapses in the human brain, each with an individually tuned weight. Eric Kandel received the Nobel Prize for demonstrating the molecular mechanisms of synaptic weight change in *Aplysia*.

    The corteX Weight Engine translates this principle: every aspect of agent behavior is governed by numerical weights that are updated through experience, not hard-coded rules. Like biological synapses, these weights have momentum (inertia), homeostatic regulation (resistance to extremes), and different learning rates depending on the category.

## How It Works

### Behavioral Weights

Behavioral weights control the agent's conversational style. They update every turn based on immediate signals:

```python
from corteX.engine.weights import WeightEngine

engine = WeightEngine()

# 10 behavioral dimensions, each on [-1, 1]
engine.behavioral.weights
# {
#   "verbosity": 0.0,          # -1=terse, 1=verbose
#   "formality": 0.0,          # -1=casual, 1=formal
#   "initiative": 0.3,         # -1=passive, 1=proactive
#   "detail_level": 0.3,       # -1=summary, 1=exhaustive
#   "creativity": 0.0,         # -1=conservative, 1=creative
#   "speed_vs_quality": 0.0,   # -1=quality, 1=speed
#   "autonomy": 0.5,           # -1=ask permission, 1=just do it
#   "explanation_depth": 0.3,  # -1=just results, 1=full reasoning
#   "code_density": 0.0,       # -1=pseudocode, 1=production code
#   "risk_tolerance": 0.0,     # -1=safe, 1=experimental
# }

# Updates use momentum and homeostatic clamping
engine.behavioral.update("verbosity", delta=0.3, lr=0.12)
```

Each update incorporates:
- **Momentum** (0.7 factor): smooths out rapid changes, like neural inertia
- **Homeostatic pull**: a gentle centering force (-0.01 * current_value) that resists extremes

### Tool Preference Weights

Tool preferences combine fast heuristic tracking (EMA) with principled Bayesian posteriors:

```python
# Record tool execution
engine.tools.record_use("code_interpreter", success=True, latency_ms=1500)

# Deterministic selection (fast)
best = engine.tools.get_best_tool(["code_interpreter", "web_search"])

# Thompson Sampling selection (Bayesian exploration/exploitation)
best = engine.tools.get_best_tool_thompson(["code_interpreter", "web_search"])

# Factor in latency for speed-sensitive tasks
best = engine.tools.get_best_tool_with_latency(
    ["code_interpreter", "web_search"],
    speed_weight=0.3,
)
```

!!! note "Brain Science: Thompson Sampling and the Explore/Exploit Tradeoff"
    The brain constantly balances exploitation (using what works) against exploration (trying alternatives). The dopaminergic system modulates this tradeoff: dopamine bursts drive exploration, while stable dopamine encourages exploitation.

    Thompson Sampling implements this mathematically. Each tool maintains a Beta distribution posterior over its success rate. To select a tool, a sample is drawn from each posterior, and the tool with the highest sample wins. Uncertain tools (wide distributions) occasionally produce high samples, driving exploration. Well-known tools (narrow distributions) reliably produce their true value.

Tool preferences also incorporate:
- **Prospect-theoretic updates**: failures hurt more than successes help (loss aversion lambda=2.25, from Kahneman-Tversky)
- **LTP/LTD**: consecutive successes trigger exponential strengthening; consecutive failures trigger exponential weakening
- **Anchor management**: informed initialization for known tools
- **Availability filtering**: controlled recency bias

### Model Selection Weights

Model selection weights learn which LLM performs best for each task type:

```python
# 7 built-in task types
engine.models._weights.keys()
# ["planning", "coding", "summarization", "validation",
#  "conversation", "tool_use", "reasoning"]

# Update after generation
engine.models.update("coding", "gemini-3-flash-preview", delta=0.3, lr=0.04)

# Get scores for routing
scores = engine.models.get_scores("coding")
# {"gemini-3-flash-preview": 0.72, "claude-sonnet": 0.65, ...}
```

### Enterprise Weights

Enterprise weights are set by admins, not learned. Some can be overridden by users:

```python
# Admin sets enterprise policy
engine.enterprise.set("safety_strictness", 0.8)
engine.enterprise.set("max_autonomy_level", 0.5)

# User tries to override
allowed = engine.enterprise.user_override("max_autonomy_level", 0.7)
# True -- this key is in the user-overridable set

allowed = engine.enterprise.user_override("data_sensitivity", 0.1)
# False -- data_sensitivity is admin-only
```

### Effective Autonomy

The effective autonomy level considers all tiers -- enterprise caps always override behavioral preferences:

```python
autonomy = engine.get_effective_autonomy()
# min(behavioral_autonomy, enterprise_max_autonomy_level)
```

### Surprise-Driven Learning Rate Modulation

When the Prediction Engine reports a surprise (prediction was wrong), the Weight Engine amplifies the learning signal:

```python
signal = engine.compute_surprise_signal(
    prediction_quality=0.9,  # Expected high quality
    actual_quality=0.4,      # Got low quality
)
# signal ≈ 0.85 (high surprise -> learn faster)
```

### Consolidation

At session end, a sleep-like consolidation process cleans up noise:

```python
consolidated = engine.consolidate()
# - Small behavioral weights (< 0.05) reset to 0
# - Failure counts decay by 0.8
# - Momentum terms halved
```

### Persistence

All weights serialize to JSON and can be saved/loaded for cross-session continuity:

```python
engine.save("~/.cortex/weights.json")
# Later:
engine.load("~/.cortex/weights.json")
```

## When It Activates

The Weight Engine is always active. Every component in corteX reads from it or writes to it:

- **FeedbackEngine** writes to behavioral and user insight weights
- **PlasticityManager** writes to tool and model weights via Hebbian/LTP/LTD rules
- **Orchestrator** reads behavioral weights to configure LLM prompts
- **ToolFramework** reads tool preference weights for selection
- **Enterprise config** writes to enterprise weights at startup
- **GoalTracker** writes to goal alignment weights

## API Reference

```python
from corteX.engine.weights import (
    WeightEngine,
    WeightCategory,
    WeightUpdate,
    LearningRates,
    BehavioralWeights,
    ToolPreferenceWeights,
    ModelSelectionWeights,
    UserInsightWeights,
    EnterpriseWeights,
    GlobalWeights,
)
```
