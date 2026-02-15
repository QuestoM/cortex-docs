# Content Prediction API Reference

## Module: `corteX.engine.content_prediction`

LLM-powered tool prediction, response evaluation, and sentiment classification. Generates prompts and parses responses -- never makes LLM calls directly. The SDK pipeline handles the actual LLM calls, keeping this module testable.

Brain analogy: Tool prediction = prefrontal cortex mental rehearsal. Parallel evaluation = ACC conflict monitoring. Sentiment classification = social cognition circuits.

---

## Classes

### `ContentPredictionConfig`

**Type**: `@dataclass`

Configuration for content-aware prediction features.

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `enable_tool_prediction` | `bool` | `True` | Enable tool call prediction |
| `enable_parallel_eval` | `bool` | `True` | Enable parallel evaluation |
| `enable_llm_sentiment` | `bool` | `True` | Enable sentiment classification |
| `cheap_model` | `str` | `"gemini-3-flash-preview"` | Model for cheap predictions |
| `max_prediction_tokens` | `int` | `256` | Max tokens for predictions |
| `cache_ttl_seconds` | `int` | `120` | Cache TTL in seconds |

### `ContentAwarePrediction`

**Type**: `@dataclass`

Result of an LLM-based content prediction for a tool call.

| Attribute | Type | Description |
|-----------|------|-------------|
| `tool_name` | `str` | Tool being predicted |
| `predicted_success` | `float` | Predicted success probability [0.0, 1.0] |
| `predicted_quality` | `float` | Predicted output quality [0.0, 1.0] |
| `risk_factors` | `List[str]` | Identified risk factors |
| `suggested_alternatives` | `List[str]` | Alternative tools to consider |

### `PredictionCache`

**Type**: Class

Simple TTL cache keyed by hashed string parts.

| Method | Description |
|--------|-------------|
| `get(*key_parts)` | Return cached value if present and not expired |
| `put(value, *key_parts)` | Store a value with current timestamp |
| `clear()` | Clear all entries |
| `prune_expired()` | Remove expired entries, return count pruned |
| `size` (property) | Number of cached entries |

---

### `ContentPredictor`

Prompt-builder + response-parser for content-aware predictions. Three capabilities, each with a `build_*_prompt()` and `parse_*_response()` pair.

#### Constructor

```python
ContentPredictor(config: Optional[ContentPredictionConfig] = None)
```

#### Capability 1: Tool Call Prediction

##### `build_tool_prediction_prompt`

```python
def build_tool_prediction_prompt(
    self, tool_name: str, tool_args: Dict[str, Any],
    context: Optional[Dict[str, Any]] = None,
    recent_history: Optional[List[Dict[str, Any]]] = None,
) -> str
```

Build a prompt asking the LLM to predict tool call outcome. Includes tool name, args (truncated to 500 chars), context (goal, step, task_type), and recent history (last 5 entries).

##### `parse_tool_prediction_response`

```python
def parse_tool_prediction_response(
    self, response_text: str, tool_name: str,
) -> ContentAwarePrediction
```

Parse LLM response into `ContentAwarePrediction`. Gracefully handles parse errors with 0.5 defaults.

#### Capability 2: Response Quality Evaluation

##### `build_evaluation_prompt`

```python
def build_evaluation_prompt(
    self, response_text: str, goal: str,
    context: Optional[Dict[str, Any]] = None,
) -> str
```

Build a prompt for parallel evaluation of response quality against the stated goal.

##### `parse_evaluation_response`

```python
def parse_evaluation_response(self, response_text: str) -> Dict[str, Any]
```

Parse evaluation response. Returns `quality_score`, `goal_alignment`, `completeness` (all [0.0, 1.0]), and `issues` list.

#### Capability 3: Sentiment Classification

##### `build_sentiment_prompt`

```python
def build_sentiment_prompt(self, text: str) -> str
```

Build a prompt for LLM-based sentiment classification of user messages.

##### `parse_sentiment_response`

```python
def parse_sentiment_response(self, response_text: str) -> Dict[str, float]
```

Parse sentiment response. Returns `satisfaction`, `frustration`, `confusion`, `urgency` (all [0.0, 1.0]). Defaults to 0.5 on failure.

#### Cache Methods

##### `get_cached_prediction` / `cache_prediction`

```python
def get_cached_prediction(self, tool_name: str, tool_args: Dict[str, Any]) -> Optional[ContentAwarePrediction]
def cache_prediction(self, prediction: ContentAwarePrediction, tool_args: Dict[str, Any]) -> None
```

Check cache for existing prediction or store a new one. Keyed by tool name + serialized args.

---

## Usage Example

```python
from corteX.engine.content_prediction import ContentPredictor

predictor = ContentPredictor()

# 1. Tool prediction
prompt = predictor.build_tool_prediction_prompt(
    "code_writer", {"language": "python", "task": "create user model"},
    context={"goal": "Build REST API", "current_step": "3/10"},
)
llm_response = await cheap_llm.generate(prompt)
prediction = predictor.parse_tool_prediction_response(llm_response, "code_writer")
print(f"Predicted success: {prediction.predicted_success:.2f}")
print(f"Risk factors: {prediction.risk_factors}")

# 2. Response evaluation
eval_prompt = predictor.build_evaluation_prompt(
    agent_response, goal="Build REST API",
)
eval_response = await cheap_llm.generate(eval_prompt)
evaluation = predictor.parse_evaluation_response(eval_response)
print(f"Quality: {evaluation['quality_score']:.2f}")

# 3. Sentiment classification
sentiment_prompt = predictor.build_sentiment_prompt("This isn't working, I'm frustrated!")
sentiment_response = await cheap_llm.generate(sentiment_prompt)
sentiment = predictor.parse_sentiment_response(sentiment_response)
print(f"Frustration: {sentiment['frustration']:.2f}")
```

---

## See Also

- [Structured Output API](./structured-output.md)
- [Reflection Engine API](./reflection.md)
