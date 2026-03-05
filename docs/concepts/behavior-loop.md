# The Behavior Loop

The Behavior Loop is corteX's closed-loop learning cycle. Every action an agent takes produces an outcome, that outcome generates feedback signals, those signals update synaptic weights through plasticity rules, and the updated weights shape the next action. This is how corteX agents get better over time - not through retraining, but through continuous, real-time adaptation.

## The Cycle

```
Action -> Outcome -> Feedback -> Plasticity -> Weight Update -> Better Action
  ^                                                                  |
  +------------------------------------------------------------------+
```

1. The agent takes an **action** (selects a tool, generates a response, picks a model).
2. The action produces an **outcome** (success/failure, user reaction, latency).
3. The `FeedbackEngine` detects **implicit signals** from the user's next message.
4. The `PlasticityManager` translates signals into **weight changes** using brain-inspired learning rules.
5. The `WeightEngine` stores the updated weights, which influence the **next action**.

No explicit "thumbs up/down" is required. The system reads between the lines.

## Stage 1: Feedback Detection

The `FeedbackEngine` coordinates a 4-tier feedback system, each operating at a different scope and timescale.

### Tier 1 - Direct Signals (`Tier1DirectFeedback`)

Immediate, per-message analysis. The `analyze()` method scans each user message for implicit cues using pattern matching with context-aware confidence scoring:

| Signal Type | What It Detects | Example Triggers |
|-------------|----------------|------------------|
| `CORRECTION` | User corrects the agent | "No, I meant...", "That's wrong" |
| `FRUSTRATION` | User shows impatience | "Just do it", "...", multiple `!!` |
| `SATISFACTION` | User is happy | "Perfect!", "Thanks", "Well done" |
| `CONFUSION` | User is lost | "I don't understand", "What do you mean" |
| `BREVITY_PREFERENCE` | User wants shorter responses | Very short messages (<=3 words) |
| `DETAIL_PREFERENCE` | User wants depth | "Explain", "More detail", "Elaborate" |
| `SPEED_PREFERENCE` | User values speed | "Quickly", "ASAP", "Fast" |

Each detected signal becomes a `FeedbackSignal` with a `signal_type`, `strength` (0.0-1.0), `source`, and `evidence` string.

Ambiguous words like "great" or "nice" are scored with context-aware confidence. A short message like "Great!" scores high confidence (0.4 base + 0.3 short-message boost + 0.2 position boost = 0.9). The same word in "The Great Wall is amazing" scores low (topic indicator penalty of -0.3) and is filtered by the `CONFIDENCE_THRESHOLD` of 0.7.

### Tier 2 - User Insights (`Tier2UserInsights`)

Cross-session preference aggregation. The `accumulate()` method tracks signal history and computes rolling metrics like `correction_frequency`, `frustration_level`, and `preferred_response_length`. These feed into the `UserInsightWeights` stored in the `WeightEngine`.

### Tier 3 - Enterprise Rules (`Tier3Enterprise`)

Admin-configured rules that apply organizational constraints. The `evaluate()` method takes a context dict and produces `WeightUpdate` objects - for example, increasing `safety_strictness` whenever the conversation topic matches "finance". Users can override some enterprise settings (those listed in `_user_overridable`), but not all.

### Tier 4 - Global Learning (`Tier4Global`)

Opt-in aggregate learning across corteX deployments. Only active for non-on-prem installations where the user explicitly calls `enable()` with a cloud endpoint. On-prem deployments operate fully independently.

### Orchestration

The `FeedbackEngine.process_user_message()` method runs all four tiers in sequence:

```python
from corteX.engine.feedback import FeedbackEngine
from corteX.engine.weights import WeightEngine

weights = WeightEngine()
feedback = FeedbackEngine(weights)

signals = feedback.process_user_message(
    message="Thanks, that's exactly what I needed!",
    context={"current_topic": "deployment"},
)
# Tier 1 detects SATISFACTION (high confidence)
# Tier 2 accumulates the signal into user insights
# Tier 3 evaluates enterprise rules against the context
# Weight engine is automatically updated
```

Weight deltas from Tier 1 signals are modulated by a **surprise signal**: unexpected feedback produces larger updates, while predictable feedback produces smaller ones. The modulation factor ranges from 0.5 (zero surprise) to 1.5 (maximum surprise), computed via `WeightEngine.compute_surprise_signal()`.

## Stage 2: Plasticity Rules

Detected feedback must be translated into weight changes. The `PlasticityManager` coordinates five brain-inspired rules after each agent step via `on_step_complete()`:

**Hebbian Learning** (`HebbianRule.apply()`): When a tool+task combination co-activates with a good outcome, their association strengthens. Bad outcomes weaken it. The delta is `(outcome_quality - 0.5) * 2`, mapping the 0-1 quality range to a -1 to +1 learning signal.

**Long-Term Potentiation** (`LTPRule.apply()`): After 3+ consecutive successes with the same pattern (e.g., `code_interpreter+coding`), an exponential bonus kicks in: `min(0.2, 0.05 * log(1 + streak - 2))`. This is how the system locks in reliably successful strategies.

**Long-Term Depression** (`LTDRule.apply()`): After 2+ consecutive failures, an exponential penalty fires: `min(0.3, 0.1 * log(1 + streak - 1))`. The lower threshold (2 vs. 3) means the system is quicker to abandon failing strategies than to commit to new ones - mirroring the brain's negativity bias.

**Homeostatic Regulation** (`HomeostaticRegulation.apply()`): Runs every 10 interactions. Behavioral weights above 0.8 are pulled back toward center. Model scores above 0.95 are redistributed to prevent monopolies. Configured with `target_mean=0.5` and `regulation_strength=0.02`.

**Critical Period Modulation** (`CriticalPeriodModulator.get_plasticity_multiplier()`): Early in a session, learning rates start at 2.0x and decay to 1.0x over the first 10 turns. After the critical period, they slowly decline to a floor of 0.5x. Call `plasticity.new_session()` to reset.

When the `PredictionEngine` reports a `SurpriseSignal`, all plasticity is amplified. The effective multiplier is `critical_period_multiplier * (1.0 + surprise.learning_signal_strength)`.

## Stage 3: Weight Storage and Retrieval

The `WeightEngine` is the central store for all learned preferences, organized into seven categories with independent learning rates:

| Category | Class | What It Tracks | Learning Rate |
|----------|-------|---------------|---------------|
| Behavioral | `BehavioralWeights` | verbosity, autonomy, risk_tolerance, creativity, etc. | 0.12 |
| Tool Preference | `ToolPreferenceWeights` | Per-tool success rates, latency, preference scores | 0.08 |
| Model Selection | `ModelSelectionWeights` | Which LLM works best for which task type | 0.04 |
| Goal Alignment | (dict on `WeightEngine`) | current_progress, drift_score, loop_risk | 0.15 |
| User Insights | `UserInsightWeights` | Preferred length, technical depth, frustration | 0.03 |
| Enterprise | `EnterpriseWeights` | safety_strictness, compliance_rules, allowed_tools | 0.02 |
| Global | `GlobalWeights` | Cross-deployment aggregate patterns | 0.01 |

Lower learning rates mean slower, more stable adaptation. Enterprise and global weights change slowly because they represent organizational knowledge, not individual preferences.

Every update passes through `WeightEngine.apply_update()`, which applies the category's `LearningRates`, records the change in a history trace (capped at 1,000 entries via `_max_history`), and returns the actual delta applied. `BehavioralWeights.update()` includes momentum (0.7 factor) and a homeostatic centering force (`-0.01 * current_value`) that gently resists extreme values. All weights are clamped to [-1.0, 1.0] for behavioral or [0.0, 1.0] for scores.

Tool selection is enhanced with Bayesian foundations: `ToolPreferenceWeights` maintains parallel Bayesian posteriors (`BayesianToolSelector`) alongside fast EMA heuristics. Use `get_best_tool_thompson()` for production selection (Thompson Sampling balances exploration vs. exploitation) or `get_best_tool()` for a deterministic fallback.

### Persistence

Weights survive across sessions via `WeightEngine.save(path)` and `WeightEngine.load(path)`, which serialize to JSON. A 0.9 exponential decay is applied at load time through `load_merged()`, preventing weights from drifting too far from baseline over many sessions.

At session end, `consolidate()` performs sleep-like cleanup: behavioral weights smaller than 0.05 snap to zero, tool failure counts decay by 0.8, and momentum is halved. This mirrors the brain's sleep consolidation, where noise is pruned and important patterns are preserved.

## The Self-Improving Agent

Putting it all together:

```python
from corteX.engine.weights import WeightEngine
from corteX.engine.feedback import FeedbackEngine
from corteX.engine.plasticity import PlasticityManager

weights = WeightEngine()
feedback = FeedbackEngine(weights)
plasticity = PlasticityManager(weights)

# Turn 1: Agent gives a verbose response
# User replies with frustration
signals = feedback.process_user_message("Too long. Just give me the answer.")
# -> FRUSTRATION + BREVITY_PREFERENCE detected
# -> behavioral weights "verbosity" and "explanation_depth" decrease

# Turn 2: Agent gives a concise response
# User is satisfied
signals = feedback.process_user_message("Perfect, thanks!")
# -> SATISFACTION detected (confidence >= 0.7)
# -> behavioral weight "autonomy" slightly increases

# After tool execution
plasticity.on_step_complete(
    tool="code_interpreter", task_type="coding",
    model="gemini-flash", success=True, quality=0.9,
)
# -> Hebbian: code_interpreter+coding association strengthened
# -> LTP: if streak >= 3, exponential bonus applied

# Session end
weights.consolidate()
weights.save("~/.cortex/weights/agent.json")
```

The next session loads these weights (decayed by 0.9) and the agent starts with learned preferences: this user prefers brevity, `code_interpreter` works well for coding, and `gemini-flash` is reliable for this workload. No explicit configuration required.

## See Also

- [Plasticity Manager](brain/plasticity.md) - deep dive into each learning rule
- [Feedback Engine](brain/feedback.md) - full signal detection reference
- [Weight Engine](brain/weights.md) - all seven weight categories
- [Brain-to-Behavior Loop](brain/behavior-loop.md) - the 14 brain components that close the execution loop
- [Prediction Engine](brain/prediction.md) - surprise signals that modulate learning
