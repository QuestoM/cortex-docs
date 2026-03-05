# Goal DNA

Goal DNA is a compact token-set fingerprint of the agent's goal that enables O(1) drift detection without any LLM calls. It extracts meaningful content tokens and character trigrams from the goal text, then computes weighted Jaccard similarity against every agent action in under 0.1ms. When similarity drops below a threshold for consecutive steps, drift is declared, giving the anti-drift system an instant, zero-cost signal.

## How It Works

At initialization, `GoalDNA` processes the goal text in two ways:

1. **Content tokens** - extracted via regex (`[a-zA-Z_][a-zA-Z0-9_]*`), lowercased, filtered to remove 100+ English stop words, and kept only if length > 2. These capture the semantic essence of the goal.
2. **Character trigrams** - every 3-character substring of the lowercased goal. These capture structural patterns that token-level analysis might miss.

For each drift check, the agent's action text is processed the same way, and similarity is computed as:

```
combined = 0.7 * jaccard(goal_tokens, action_tokens) + 0.3 * jaccard(goal_trigrams, action_trigrams)
```

The 70/30 token-to-trigram weighting ensures semantic similarity dominates while structural similarity provides a secondary signal.

### Drift Detection

`check_drift(step_number, action_text)` compares similarity against the drift threshold (default 0.15). If below:

- A `DriftEvent` is recorded with severity classification
- The consecutive drift counter increments
- When the counter exceeds `consecutive_limit` (default 3), `is_drifting()` returns True

When an action is above threshold, the consecutive counter resets to zero.

### Severity Classification

Severity is relative to the drift threshold:

| Similarity Range | Severity |
|-----------------|----------|
| >= threshold | NONE |
| >= threshold * 0.7 | LOW |
| >= threshold * 0.4 | MODERATE |
| >= threshold * 0.2 | HIGH |
| < threshold * 0.2 | CRITICAL |

## Key Features

- **Sub-millisecond drift checks** - Jaccard on frozen sets is extremely fast
- **No LLM calls** - pure Python computation, works fully offline
- **Dual similarity** - token-level (semantic) + trigram-level (structural)
- **Trend analysis** - `get_trend(window)` computes direction ("improving", "worsening", "stable"), slope, average similarity, min/max, and consecutive drift count
- **Drift history** - all drift events are recorded with step number, similarity, action text, and severity
- **Consecutive tracking** - `is_drifting()` only fires after sustained drift, not one-off outliers
- **Reset capability** - `reset_consecutive()` clears the counter after successful replanning
- **Summary reporting** - `get_summary()` returns a dict with token count, total checks, drift events, trend direction, and current state

## Integration

Goal DNA is used by multiple corteX components:

- **Drift Engine** - uses a lightweight internal GoalDNA for its goal relevance signal (one of five fusion signals)
- **Goal Reminder Injector** - when `is_drifting()` returns True, the reminder injector activates
- **Session post-turn pipeline** - `check_drift()` is called after each step, and the result flows into ResponseMetadata
- **Goal Tracker** - can use GoalDNA similarity to verify step relevance

## Usage Example

```python
from corteX.engine.goal_dna import GoalDNA

dna = GoalDNA(
    "Build a REST API for user management with JWT authentication",
    drift_threshold=0.15,
    consecutive_limit=3,
)

# Check a relevant action - no drift
sim = dna.similarity("Creating database migration for users table")
print(f"Similarity: {sim:.3f}")  # ~0.25 (above threshold)

# Check an irrelevant action - drift!
event = dna.check_drift(step_number=5, action_text="Researching quantum computing")
if event:
    print(f"Drift! Severity: {event.severity.value}")  # "critical"

# After 3+ consecutive drifts
print(f"Is drifting: {dna.is_drifting()}")  # True after 3 below-threshold steps

# Analyze trend
trend = dna.get_trend(window=10)
print(f"Direction: {trend.direction}")
print(f"Avg similarity: {trend.average_similarity:.3f}")
print(f"Slope: {trend.trend_slope:.4f}")  # negative = worsening

# After replanning
dna.reset_consecutive()
```

## See Also

- [Drift Engine](../anti-drift/drift-engine.md) - five-signal drift detection that uses GoalDNA internally
- [Goal Reminder](goal-reminder.md) - reminder injection triggered by sustained drift
- [Goal Tree](goal-tree.md) - hierarchical goal decomposition
- [Loop Detector](../anti-drift/loop-detector.md) - complementary behavioral loop detection
