# Structured Output API Reference

## Module: `corteX.engine.structured_output`

Extracts self-assessed signals from LLM responses (confidence, difficulty, escalation needs, per-tool confidence) and combines them with brain signals for unified quality assessment. Three components: Injector (prompts LLM), Parser (extracts signals), Aggregator (fuses all sources).

---

## Classes

### `DifficultyLevel`

**Type**: `str, Enum`

Task difficulty as self-assessed by the LLM.

| Value | Description |
|-------|-------------|
| `TRIVIAL` | Straightforward task |
| `EASY` | Simple with clear path |
| `MEDIUM` | Moderate complexity |
| `HARD` | Challenging task |
| `EXTREME` | Very difficult, may need escalation |

**Parsing**: `DifficultyLevel.from_string()` handles synonyms (`"simple"` -> EASY, `"challenging"` -> HARD, `"impossible"` -> EXTREME).

### `StructuredSignals`

**Type**: `@dataclass`

Self-assessed signals extracted from an LLM response.

| Attribute | Type | Description |
|-----------|------|-------------|
| `confidence` | `float` | Self-assessed confidence [0.0, 1.0] |
| `difficulty` | `DifficultyLevel` | Task difficulty assessment |
| `escalation_needed` | `bool` | Whether more compute/context/human help needed |
| `escalation_reason` | `Optional[str]` | Why escalation is needed |
| `reasoning_steps` | `int` | Number of reasoning steps taken |
| `tools_confidence` | `Dict[str, float]` | Per-tool confidence scores |
| `raw_json` | `Optional[Dict]` | Raw JSON from LLM |
| `source` | `str` | `"default"`, `"cortex_block"`, `"generic_json"`, `"keyword_scan"`, `"parsed"` |

**Methods**: `to_dict()`, `from_dict()`, `defaults()`.

### `AggregatedQuality`

**Type**: `@dataclass`

Unified quality assessment combining LLM + brain signals.

| Attribute | Type | Description |
|-----------|------|-------------|
| `composite_confidence` | `float` | Weighted composite [0.0, 1.0] |
| `confidence_agreement` | `float` | Agreement between LLM and brain signals |
| `recommended_action` | `str` | `"proceed"`, `"proceed_confident"`, `"verify_output"`, `"retry_with_stronger_model"`, `"escalate_to_system2"`, `"escalate_to_human"` |
| `escalation_urgency` | `float` | Urgency of escalation [0.0, 1.0] |
| `signal_richness` | `float` | How many signal sources contributed |
| `details` | `Dict[str, Any]` | Detailed breakdown |

---

### `StructuredOutputInjector`

Generates instruction text for LLM system prompts requesting structured signals.

#### Constructor

```python
StructuredOutputInjector(verbosity: str = "full")
```

#### Methods

##### `generate_instruction`

```python
def generate_instruction(
    self, tool_names: Optional[List[str]] = None,
    verbosity: Optional[str] = None,
) -> str
```

Generate instruction text to append to system prompt. Two modes:

- **`"full"`**: Detailed instructions with confidence guidelines (~100 tokens)
- **`"compact"`**: One-line instruction (~30 tokens)

Requests JSON in `cortex-signals` code block format.

---

### `StructuredOutputParser`

Extracts structured signals from LLM response text with multi-strategy fallback.

#### Methods

##### `parse`

```python
def parse(self, response_text: str) -> StructuredSignals
```

Parse signals using cascading strategies:

1. **cortex-block**: Extract from ` ```cortex-signals {...} ``` ` block
2. **generic JSON**: Find any JSON with `"confidence"` key
3. **keyword scan**: Regex for `confidence:`, `difficulty:`, `escalation` keywords
4. **defaults**: Return default signals if nothing found

##### `strip_signal_block`

```python
def strip_signal_block(self, response_text: str) -> str
```

Remove cortex-signals block from response, returning clean user-facing content.

---

### `SignalAggregator`

Combines structured LLM signals with brain signals (surprise, ECE, population).

#### Constructor

```python
SignalAggregator(
    llm_weight: float = 0.30,
    population_weight: float = 0.30,
    calibration_weight: float = 0.25,
    surprise_weight: float = 0.15,
)
```

Weights are auto-normalized to sum to 1.0.

#### Methods

##### `aggregate`

```python
def aggregate(
    self, signals: StructuredSignals,
    prediction_surprise: Optional[float] = None,
    calibration_ece: Optional[float] = None,
    population_quality: Optional[float] = None,
    population_agreement: Optional[float] = None,
) -> AggregatedQuality
```

Combine all signal sources into unified quality assessment. Missing signals default to 0.5.

**Recommended action logic**:

| Condition | Action |
|-----------|--------|
| urgency >= 0.7 | `escalate_to_human` |
| urgency >= 0.5 | `escalate_to_system2` |
| confidence < 0.3 | `retry_with_stronger_model` |
| agreement < 0.4 | `verify_output` |
| confidence >= 0.8 and agreement >= 0.7 | `proceed_confident` |
| Default | `proceed` |

---

## Usage Example

```python
from corteX.engine.structured_output import (
    StructuredOutputInjector, StructuredOutputParser, SignalAggregator,
)

# 1. Inject instructions into system prompt
injector = StructuredOutputInjector(verbosity="full")
instruction = injector.generate_instruction(tool_names=["code_writer", "test_runner"])
system_prompt += "\n\n" + instruction

# 2. Parse signals from LLM response
parser = StructuredOutputParser()
signals = parser.parse(llm_response)
clean_response = parser.strip_signal_block(llm_response)

# 3. Aggregate with brain signals
aggregator = SignalAggregator()
quality = aggregator.aggregate(
    signals,
    prediction_surprise=0.2,
    calibration_ece=0.1,
    population_quality=0.85,
)

print(f"Confidence: {quality.composite_confidence:.2f}")
print(f"Action: {quality.recommended_action}")
```

---

## See Also

- [Content Prediction API](./content-prediction.md)
- [Reflection Engine API](./reflection.md)
