# Feedback Engine

The Feedback Engine infers user satisfaction from implicit signals -- without ever asking "Was this response helpful?" It detects corrections, frustration, satisfaction, brevity preferences, and speed hints from the patterns in user messages, then translates these signals into weight updates.

## What It Does

The Feedback Engine operates across four tiers, each with a different scope and timescale:

| Tier | Scope | Brain Analogy | Source |
|------|-------|---------------|--------|
| **Tier 1: Direct** | Single turn | Amygdala (immediate emotional signals) | Pattern matching on user message |
| **Tier 2: User Insights** | Cross-session | Hippocampus (episodic memory) | Accumulated Tier 1 signals |
| **Tier 3: Enterprise** | Organization | Prefrontal cortex (social norms) | Admin-configured rules |
| **Tier 4: Global** | All deployments | Collective knowledge | Opt-in cloud aggregation |

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Implicit Reward Signals"
    The brain does not wait for explicit feedback to learn. The amygdala continuously monitors the emotional valence of incoming stimuli -- is this good or bad? -- and generates rapid reward/punishment signals that modulate learning in the basal ganglia and cortex.

    A person's facial expression, tone of voice, or body language communicates volumes without a single word of explicit feedback. The Feedback Engine applies this same principle to text: word choice, message length, punctuation patterns, and conversational structure all carry implicit signals about satisfaction.

    Tier 2 mirrors the hippocampus's role in consolidating episodic memories: individual interactions are aggregated into a user profile that persists across sessions. Tier 3 mirrors the prefrontal cortex's enforcement of social norms -- organizational rules that constrain individual behavior.

## How It Works

### Tier 1: Direct Feedback Detection

Tier 1 uses regex pattern matching to detect implicit signals from user messages:

```python
from corteX.engine.feedback import FeedbackEngine
from corteX.engine.weights import WeightEngine

weights = WeightEngine()
feedback = FeedbackEngine(weights)

# Detect implicit signals
signals = feedback.process_user_message(
    "No, that's wrong. I meant the other approach."
)
# Detected: CORRECTION (strength=0.8)
# Weight updates: autonomy -0.12, risk_tolerance -0.08
```

**Signal types detected:**

| Signal | Patterns | Weight Effect |
|--------|----------|---------------|
| CORRECTION | "no, I meant...", "that's wrong", "try again" | autonomy down, risk_tolerance down |
| FRUSTRATION | "...", "just do X", "ugh", "!!" | verbosity down, explanation_depth down |
| SATISFACTION | "perfect", "exactly", "thanks" | autonomy up (reinforce current approach) |
| BREVITY | Very short messages (<=3 words) | verbosity down |
| SPEED | "quickly", "ASAP", "fast" | speed_vs_quality up |
| DETAIL | "explain", "more detail", "elaborate" | detail_level up, explanation_depth up |
| ENGAGEMENT | Long messages (>50 words) | engagement_level up |
| DISENGAGEMENT | Empty messages | engagement down |

### Tier 2: Cross-Session Insights

Tier 2 accumulates Tier 1 signals over time to build a persistent user profile:

```python
# After many interactions, Tier 2 has accumulated insights
summary = feedback.get_signal_summary()
# {
#   "total_interactions": 42,
#   "corrections": 5,
#   "satisfaction_count": 28,
#   "signal_history_length": 156,
# }
```

Accumulated insights update the `UserInsightWeights`:
- `preferred_response_length`: short / medium / long
- `domain_expertise_level`: beginner / intermediate / expert
- `frustration_level`: 0.0 (happy) to 1.0 (very frustrated)
- `correction_frequency`: how often the user corrects the agent
- `time_sensitivity`: 0.0 (patient) to 1.0 (urgent)

### Tier 3: Enterprise Rules

Enterprise admins can configure topic-based feedback rules:

```python
feedback.tier3.configure({
    "topic_safety_rules": {
        "finance": 0.3,     # Increase safety when discussing finance
        "healthcare": 0.5,  # Increase safety more for healthcare
    }
})

# When the topic matches, enterprise weight updates are generated
signals = feedback.process_user_message(
    "How should we handle patient data?",
    context={"current_topic": "healthcare data processing"},
)
# Enterprise rule fires: safety_strictness += 0.5
```

### Tier 4: Global (Opt-In)

For non-on-prem deployments where users opt in, anonymized aggregate learning is shared across corteX deployments. This is disabled by default and never activated for on-prem installations:

```python
# Disabled by default
feedback.tier4.enabled  # False

# Opt-in for cloud deployments
feedback.tier4.enable("https://api.cortex.questo.ai/v1/global")
```

### Signal-to-Weight Translation

Each detected signal maps to specific behavioral weight updates. The translation is designed to be proportional -- stronger signals produce larger weight changes:

```python
# CORRECTION signal (strength=0.8):
#   autonomy delta = -0.15 * 0.8 = -0.12
#   risk_tolerance delta = -0.1 * 0.8 = -0.08

# FRUSTRATION signal (strength=0.6):
#   verbosity delta = -0.2 * 0.6 = -0.12
#   explanation_depth delta = -0.15 * 0.6 = -0.09

# SATISFACTION signal (strength=0.7):
#   autonomy delta = +0.05 * 0.7 = +0.035
```

Note the asymmetry: negative signals (correction, frustration) produce larger deltas than positive signals (satisfaction). This reflects prospect theory's loss aversion -- the system is more responsive to problems than to praise.

## When It Activates

- **Every user message**: Tier 1 pattern matching runs on every incoming message
- **Every interaction**: Tier 2 accumulates signals and updates the user profile
- **When topic context is available**: Tier 3 enterprise rules are evaluated
- **Periodically (opt-in only)**: Tier 4 syncs anonymized metrics

## API Reference

```python
from corteX.engine.feedback import (
    FeedbackEngine,
    FeedbackSignal,
    SignalType,
    Tier1DirectFeedback,
    Tier2UserInsights,
    Tier3Enterprise,
    Tier4Global,
)
```
