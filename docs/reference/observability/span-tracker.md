# Span Tracker API Reference

## Module: `corteX.dashboard.span_tracker`

Tracks which context spans the LLM actually used in its response. When context is packed (system prompt, memories, retrieved documents, recent turns), we cannot know which chunks the LLM actually attended to. SpanTracker analyzes responses with lightweight heuristics (no extra LLM call) to estimate which context items were referenced, enabling token waste optimization.

Brain analogy: Meta-cognition about attention - knowing what you paid attention to and what you ignored.

---

## Classes

### `SpanUsage`

**Type**: `@dataclass`

Usage data for a single context span.

| Attribute | Type | Description |
|-----------|------|-------------|
| `span_id` | `str` | Unique identifier for this span. |
| `span_type` | `str` | Category: "system_prompt", "memory", "retrieved", "recent_turn", "pinned". |
| `tokens` | `int` | Token count for this span. |
| `referenced` | `bool` | Whether the span was detected as used. |
| `reference_score` | `float` | Confidence score (0.0-1.0) that the span was used. |
| `reference_evidence` | `str` | Brief explanation (e.g. "keyword overlap 45%, entity match 30%"). |

### Methods

#### `to_dict() -> Dict[str, Any]`

Serialize to dictionary. Rounds `reference_score` to 4 decimal places.

---

### `SpanReport`

**Type**: `@dataclass`

Full span usage report for a single turn.

| Attribute | Type | Description |
|-----------|------|-------------|
| `total_context_tokens` | `int` | Total tokens across all context items. |
| `used_spans` | `List[SpanUsage]` | Spans classified as referenced. |
| `unused_spans` | `List[SpanUsage]` | Spans classified as unreferenced. |
| `utilization_ratio` | `float` | Ratio of used tokens to total tokens (0.0-1.0). |
| `waste_tokens` | `int` | Total tokens in unused spans. |
| `timestamp` | `float` | When the report was generated (default: `time.time()`). |

### Methods

#### `to_dict() -> Dict[str, Any]`

Serialize to dictionary. Includes `used_count` and `unused_count` convenience fields.

---

### `SpanTracker`

**Type**: `class`

Analyzes LLM responses to estimate which context items were actually referenced. Uses three heuristic signals combined with configurable weights.

#### Constructor

```python
SpanTracker(
    threshold: float = 0.15,
    keyword_weight: float = 0.4,
    entity_weight: float = 0.35,
    structural_weight: float = 0.25,
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `threshold` | `float` | `0.15` | Minimum combined score for a span to be classified as "referenced". |
| `keyword_weight` | `float` | `0.4` | Weight for keyword overlap signal. |
| `entity_weight` | `float` | `0.35` | Weight for entity match signal. |
| `structural_weight` | `float` | `0.25` | Weight for structural reference signal. |

### Methods

#### `analyze(context_items, response) -> SpanReport`

Analyze which context items were used in the response.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `context_items` | `List[Dict[str, Any]]` | (required) | Context items, each with keys: `span_id` (str), `span_type` (str), `content` (str), `tokens` (int). |
| `response` | `str` | (required) | The LLM's response text. |

**Returns:** `SpanReport` with used/unused classification, utilization ratio, and waste metrics.

**Heuristic signals:**

| Signal | Weight | How it works |
|--------|--------|-------------|
| Keyword overlap | 0.4 | Ratio of shared meaningful words (stopwords excluded) between span and response. |
| Entity matching | 0.35 | Shared entities - names, numbers, URLs, and emails found in both span and response. |
| Structural references | 0.25 | Phrases like "as mentioned", "based on", "according to" plus trigram quote detection. |

**Classification:** A span is "referenced" when its combined weighted score >= `threshold` (default 0.15).

---

## Usage Example

```python
from corteX.dashboard.span_tracker import SpanTracker

tracker = SpanTracker()

context_items = [
    {
        "span_id": "sys-prompt",
        "span_type": "system_prompt",
        "content": "You are a financial analyst specializing in NASDAQ stocks.",
        "tokens": 12,
    },
    {
        "span_id": "mem-1",
        "span_type": "memory",
        "content": "User portfolio contains AAPL, GOOGL, and MSFT positions.",
        "tokens": 15,
    },
    {
        "span_id": "doc-1",
        "span_type": "retrieved",
        "content": "Q3 earnings report: AAPL revenue $89.5B, up 8% YoY.",
        "tokens": 20,
    },
]

response = "Based on the Q3 earnings, AAPL revenue of $89.5B shows strong growth."

report = tracker.analyze(context_items, response)
print(f"Utilization: {report.utilization_ratio:.0%}")
print(f"Wasted tokens: {report.waste_tokens}")

for span in report.used_spans:
    print(f"  USED: {span.span_id} ({span.reference_evidence})")
for span in report.unused_spans:
    print(f"  UNUSED: {span.span_id}")
```

---

## See Also

- [Context Pinning API](../engine/context-pin.md)
- [Context Scoring API](../engine/context-scoring.md)
