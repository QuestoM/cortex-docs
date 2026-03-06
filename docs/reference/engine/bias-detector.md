# Bias Detector API Reference

## Module: `corteX.engine.bias_detector_core` + `corteX.engine.bias_detector_types`

Detects when LLM pre-training defaults override retrieved context instructions. Uses heuristic analysis to compare explicit instructions against the LLM response, checking format compliance, length constraints, style matching, and factual grounding.

Brain analogy: The prefrontal cortex overriding habitual responses when they conflict with current task goals. The detector identifies when the model's "habits" (training data defaults) are winning over the explicit instructions.

---

## Classes

### `BiasType`

**Type**: `str, Enum`

Category of detected pre-training bias.

| Value | Description |
|-------|-------------|
| `FORMAT` | Response format does not match requested format (JSON, bullets, table, code, XML). |
| `STYLE` | Response tone does not match requested style (formal, casual, simple). |
| `FACTUAL` | Response uses training knowledge instead of provided context facts. |
| `BEHAVIORAL` | Reserved for behavioral bias detection. |
| `LENGTH` | Response length does not match length constraints (brief, detailed, exact count). |

---

### `BiasDetection`

**Type**: `@dataclass`

A detected instance of training bias overriding context.

| Attribute | Type | Description |
|-----------|------|-------------|
| `bias_type` | `BiasType` | Category of the detected bias. |
| `confidence` | `float` | Detection confidence (0.0-1.0, clamped). |
| `instruction` | `str` | The instruction that was violated. |
| `evidence` | `str` | Evidence of the violation in the response. |
| `suggestion` | `str` | Suggested prompt change to fix the bias. |

---

### `BiasReport`

**Type**: `@dataclass`

Aggregated bias detection results for a single response.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `detections` | `List[BiasDetection]` | `[]` | All detected bias instances. |
| `overall_compliance` | `float` | `1.0` | Compliance score (1.0 = fully compliant, 0.0 = fully non-compliant). |

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `has_bias` | `bool` | `True` if any detections exist. |
| `max_confidence` | `float` | Highest confidence among all detections (0.0 if none). |

---

### `BiasDetector`

**Type**: `class`

Detects pre-training bias overriding context instructions. Uses lightweight heuristics (no extra LLM call) to compare instructions vs response across four dimensions: format, length, style, and factual grounding.

### Methods

#### `detect(instructions, response, context_facts=None) -> BiasReport`

Detect training biases in a response.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `instructions` | `List[str]` | (required) | Explicit instructions from system prompt or context. |
| `response` | `str` | (required) | The LLM's response text to analyze. |
| `context_facts` | `Optional[List[str]]` | `None` | Facts provided in context to check factual grounding. |

**Returns:** `BiasReport` with all detected bias instances and overall compliance score.

**Detection logic:**

| Check | What it detects | Typical confidence |
|-------|-----------------|-------------------|
| Format compliance | JSON, bullets, numbered list, table, code, XML format mismatch | 0.65-0.85 |
| Length compliance | Too verbose when "brief" requested, too short when "detailed" requested, exact count violations | 0.50-0.95 |
| Style compliance | Casual markers in formal context, formal markers in casual context, technical jargon when simple requested | 0.40-0.90 |
| Factual grounding | Context facts not reflected in response (< 50% usage) | up to 0.95 |

**Compliance scoring:** `1.0 - (sum of confidences) / (count * 1.5)`, clamped to [0.0, 1.0].

---

## Format Detection Patterns

The module defines regex patterns used for format, length, and style detection:

| Pattern | Matches |
|---------|---------|
| `FORMAT_INSTRUCTIONS` | Dict mapping format keys ("json", "bullet", "numbered", "table", "code", "xml") to regex patterns and labels. |
| `LENGTH_SHORT` | Words like "brief", "concise", "short", "one-liner", "2-3 sentences". |
| `LENGTH_LONG` | Words like "detailed", "comprehensive", "thorough", "exhaustive". |
| `LENGTH_EXACT` | Patterns like "100 words", "3 sentences", "5 paragraphs". |
| `STYLE_FORMAL` | Words like "formal", "professional", "academic", "corporate". |
| `STYLE_CASUAL` | Words like "casual", "informal", "conversational", "friendly". |
| `STYLE_SIMPLE` | Words like "simple", "plain", "easy to understand", "ELI5". |

---

## Usage Example

```python
from corteX.engine.bias_detector_core import BiasDetector

detector = BiasDetector()

report = detector.detect(
    instructions=["Respond in JSON format", "Be concise, max 2 sentences"],
    response="Here is a detailed explanation of the topic. The concept involves "
             "multiple layers of complexity that require careful consideration. "
             "Let me walk you through each aspect in detail...",
)

print(f"Compliance: {report.overall_compliance:.0%}")
if report.has_bias:
    for d in report.detections:
        print(f"  [{d.bias_type.value}] {d.evidence}")
        print(f"    Suggestion: {d.suggestion}")

# With factual grounding check
report = detector.detect(
    instructions=["Answer using only the provided facts"],
    response="The capital of France is Paris.",
    context_facts=["France has 67 million inhabitants", "Paris is the capital"],
)
```

---

## See Also

- [Context Pinning API](./context-pin.md)
- [Cross Verifier API](./cross-verifier.md)
- [Production Readiness API](./readiness.md)
