# Feedback Engine

`corteX.engine.feedback`

The 4-tier feedback engine infers user satisfaction from implicit signals and updates weights accordingly -- without ever asking "are you satisfied?" directly. Detects corrections, frustration, satisfaction, engagement, brevity/detail preferences, and speed urgency from natural conversation patterns.

---

## SignalType

Enum of implicit feedback signal types.

```python
class SignalType(str, Enum):
    CORRECTION = "correction"
    FRUSTRATION = "frustration"
    SATISFACTION = "satisfaction"
    DISENGAGEMENT = "disengagement"
    ENGAGEMENT = "engagement"
    TOPIC_INTEREST = "topic_interest"
    BREVITY_PREFERENCE = "brevity"
    DETAIL_PREFERENCE = "detail"
    SPEED_PREFERENCE = "speed"
    QUALITY_PREFERENCE = "quality"
```

---

## FeedbackSignal

A detected feedback signal from user interaction.

| Attribute | Type | Description |
|-----------|------|-------------|
| `signal_type` | `SignalType` | Category of the detected signal |
| `strength` | `float` | Signal intensity (`0.0` to `1.0`) |
| `source` | `str` | Which detector tier found this signal |
| `evidence` | `str` | What triggered the detection |
| `timestamp` | `float` | Unix timestamp (auto-populated) |

---

## Tier1DirectFeedback

Tier 1: Immediate conversation signal detection. Analyzes user messages for implicit feedback using regex pattern matching with context-aware confidence scoring.

### Class Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `CONFIDENCE_THRESHOLD` | `float` | Minimum confidence (`0.7`) to emit a signal |
| `CORRECTION_PATTERNS` | `List[str]` | Regex patterns for correction detection |
| `FRUSTRATION_PATTERNS` | `List[str]` | Regex patterns for frustration detection |
| `SATISFACTION_PATTERNS_WEIGHTED` | `List[Tuple[str, float]]` | Weighted patterns with base confidence |
| `SPEED_PATTERNS` | `List[str]` | Regex patterns for speed/urgency detection |
| `DETAIL_PATTERNS` | `List[str]` | Regex patterns for detail-seeking behavior |

### Methods

#### `analyze(user_message: str, response_time_ms: float = 0) -> List[FeedbackSignal]`

Analyzes a user message for implicit feedback signals. Uses context-aware confidence scoring to reduce false positives -- ambiguous words like "great" in descriptive sentences are filtered unless confidence exceeds the threshold.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `user_message` | `str` | The user's raw message text |
| `response_time_ms` | `float` | Time taken for the previous response |

**Returns:** List of detected `FeedbackSignal` instances.

**Detection rules:**

- **Correction**: Patterns like "no, I meant...", "that's wrong", "try again"
- **Frustration**: Patterns like "...", "just do X", "why can't you", multiple `!?`
- **Satisfaction**: Context-aware -- "perfect", "exactly", "thanks" scored by position and message length
- **Brevity**: Messages with 3 or fewer words (when not satisfaction)
- **Speed**: Keywords like "quickly", "ASAP", "fast"
- **Detail**: 2+ patterns like "explain", "why", "elaborate"
- **Engagement**: Messages over 50 words
- **Disengagement**: Empty messages

---

## Tier2UserInsights

Tier 2: Cross-session preference learning. Aggregates Tier 1 signals over time to build a persistent user profile.

### Methods

#### `accumulate(signals: List[FeedbackSignal]) -> Dict[str, Any]`

Accumulates signals and computes insight updates. Returns a dict of weight updates to apply to `UserInsightWeights`.

**Tracked insights:** `correction_frequency`, `frustration_level`, `preferred_response_length`, `technical_depth`, `time_sensitivity`, `engagement_level`.

---

## Tier3Enterprise

Tier 3: Enterprise-configurable feedback rules. Admin-defined rules that generate weight updates based on context (e.g., topic-based safety adjustments).

### Methods

#### `configure(rules: Dict[str, Any]) -> None`

Admin configures enterprise feedback rules and enables the tier.

#### `evaluate(context: Dict[str, Any]) -> List[WeightUpdate]`

Applies enterprise rules to generate weight updates. Supports `topic_safety_rules` mapping topic patterns to safety adjustments.

---

## Tier4Global

Tier 4: Aggregate learning across corteX deployments (opt-in, non-on-prem only).

### Methods

#### `enable(endpoint: str) -> None`

Enables global sync with a cloud endpoint.

#### `async sync(weight_engine: WeightEngine) -> None`

Syncs anonymized metrics with the corteX cloud. Currently a placeholder.

---

## FeedbackEngine

Main feedback engine coordinating all 4 tiers. Processes user messages through signal detection, cross-session accumulation, enterprise rules, and global sync.

### Constructor

```python
FeedbackEngine(weight_engine: WeightEngine)
```

| Attribute | Type | Description |
|-----------|------|-------------|
| `weights` | `WeightEngine` | The weight engine to update |
| `tier1` | `Tier1DirectFeedback` | Immediate signal detection |
| `tier2` | `Tier2UserInsights` | Cross-session learning |
| `tier3` | `Tier3Enterprise` | Enterprise rules |
| `tier4` | `Tier4Global` | Global aggregate learning |

### Methods

#### `process_user_message(message: str, response_time_ms: float = 0, context: Optional[Dict[str, Any]] = None) -> List[FeedbackSignal]`

Processes a user message through all feedback tiers. Automatically detects signals and updates the weight engine.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `message` | `str` | The user's message text |
| `response_time_ms` | `float` | Response time for the previous turn |
| `context` | `Optional[Dict]` | Additional context for enterprise rules |

**Returns:** List of detected `FeedbackSignal` instances.

**Weight updates by signal type:**

| Signal | Weight Updates |
|--------|---------------|
| `CORRECTION` | `autonomy` -0.15, `risk_tolerance` -0.1 |
| `FRUSTRATION` | `verbosity` -0.2, `explanation_depth` -0.15 |
| `SATISFACTION` | `autonomy` +0.05 |
| `BREVITY_PREFERENCE` | `verbosity` -0.15 |
| `DETAIL_PREFERENCE` | `detail_level` +0.2, `explanation_depth` +0.15 |
| `SPEED_PREFERENCE` | `speed_vs_quality` +0.2 |

#### `get_signal_summary() -> Dict[str, Any]`

Returns summary with: `total_interactions`, `corrections`, `satisfaction_count`, `signal_history_length`, `enterprise_enabled`, `global_enabled`.

### Example

```python
from corteX.engine.weights import WeightEngine
from corteX.engine.feedback import FeedbackEngine

weights = WeightEngine()
feedback = FeedbackEngine(weights)

# Process user messages -- weights update automatically
signals = feedback.process_user_message("Thanks, that's perfect!")
# -> [FeedbackSignal(SATISFACTION, strength=0.35)]
# -> autonomy weight increases by 0.05 * 0.35

signals = feedback.process_user_message("No, that's wrong. Try again.")
# -> [FeedbackSignal(CORRECTION, strength=0.8)]
# -> autonomy decreases, risk_tolerance decreases

# Enterprise rules
feedback.tier3.configure({
    "topic_safety_rules": {"finance": 0.3, "medical": 0.5}
})
signals = feedback.process_user_message(
    "Show me the report",
    context={"current_topic": "financial analysis"},
)
```
