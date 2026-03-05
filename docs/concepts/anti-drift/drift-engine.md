# Drift Engine

The Drift Engine monitors whether an agent is staying aligned with its original goal using five-signal fusion. It combines goal relevance, token budget consumption, topic divergence, output quality trend, and prediction surprise into a single drift score, then maps that score to a graduated severity level with a recommended corrective action. No LLM call is needed - all computation is pure Python using Jaccard similarity and entity extraction.

## How It Works

At each step, `assess_step()` computes five raw signals and combines them with weighted fusion:

| Signal | Weight | What It Measures |
|--------|--------|-----------------|
| Goal relevance | 35% | Jaccard similarity between the step's text and the goal's token fingerprint (GoalDNA). Low similarity = drifting away. |
| Token budget ratio | 15% | Fraction of the total token budget consumed. High consumption without proportional progress = risk. |
| Topic divergence | 20% | Fraction of entities in the current step that are new (not in the goal). Many new entities = topic shift. |
| Output quality trend | 15% | Whether output length is declining relative to the recent average. Sharp drops suggest the agent is producing less useful output. |
| Prediction surprise | 15% | Accumulated negative surprise from the prediction engine. High surprise = the agent's actions are unexpected, possibly off-track. |

A bonus penalty (+0.15) is applied when the agent has had consecutive low-similarity steps (default threshold: 3), catching sustained drift that each individual step might not trigger.

### Severity Levels and Actions

| Score Range | Severity | Recommended Action |
|-------------|----------|-------------------|
| < 0.1 | NONE | `CONTINUE` - stay on course |
| 0.1 - 0.3 | LOW | `CONTINUE` - minor deviation, self-correcting |
| 0.3 - 0.5 | MODERATE | `INJECT_REMINDER` - add goal reminder to context |
| 0.5 - 0.7 | HIGH | `SUMMARIZE_REPLAN` - summarize progress and replan |
| 0.7 - 0.85 | CRITICAL | `CHECKPOINT_RESET` - save checkpoint and reset approach |
| >= 0.85 | EMERGENCY | `ASK_USER` - halt and request user guidance |

## Key Features

- **Five-signal weighted fusion** - no single signal dominates; robust multi-dimensional drift detection
- **GoalDNA fingerprinting** - compact token-set + trigram representation of the goal for O(1) similarity checks
- **Consecutive drift tracking** - penalty for sustained low-similarity runs
- **Entity-based topic analysis** - extracts capitalized words, underscored identifiers, and long words as entity proxies
- **Surprise accumulator** - exponentially weighted moving average of negative prediction surprise (decay factor 0.7)
- **Drift trend** - `get_drift_trend(last_n)` returns average drift score over recent assessments
- **Bounded history** - assessment history capped at 100 entries, step history configurable

## Integration

The Drift Engine is a central component of the corteX anti-drift system:

- **Goal Reminder Injector** - when severity is MODERATE, the injector adds a goal reminder to the LLM context
- **Loop Detector** - drift and loops are complementary signals; both feed into the session's recovery logic
- **Prediction Engine** - supplies the `prediction_surprise` signal via `record_negative_surprise()`
- **Adaptive Budget** - budget consumption feeds into the token budget ratio signal
- **Session post-turn pipeline** - `assess_step()` is called after every agent step, and the drift score is surfaced in `ResponseMetadata.drift_score`

## Usage Example

```python
from corteX.engine.drift_engine import DriftEngine, DriftAction

engine = DriftEngine(
    goal="Build a REST API for user management with JWT auth",
    token_budget=50000,
)

# Assess each step
assessment = engine.assess_step(
    description="Create database migration for users table",
    output="Generated migration file with id, email, password_hash columns",
    tokens_used=1200,
    prediction_surprise=0.1,
)
print(f"Drift score: {assessment.score:.2f}")  # ~0.15 (low - on topic)
print(f"Severity: {assessment.severity.value}")  # "low"
print(f"Action: {assessment.recommended_action.value}")  # "continue"

# A drifting step
assessment = engine.assess_step(
    description="Researching blockchain consensus algorithms",
    output="Proof of stake vs proof of work comparison...",
    tokens_used=2000,
)
print(f"Drift score: {assessment.score:.2f}")  # ~0.65 (high - off topic)
print(f"Action: {assessment.recommended_action.value}")  # "summarize_replan"

# Check trend
trend = engine.get_drift_trend(last_n=5)
print(f"Average drift: {trend:.2f}")
```

## See Also

- [Goal DNA](../goal-intelligence/goal-dna.md) - the fingerprinting system used by the drift engine
- [Goal Reminder](../goal-intelligence/goal-reminder.md) - reminder injection triggered by moderate drift
- [Loop Detector](loop-detector.md) - complementary behavioral loop detection
- [Adaptive Budget](adaptive-budget.md) - budget-aware resource management
