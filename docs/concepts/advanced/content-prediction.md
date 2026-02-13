# Content-Aware Predictions

The Content-Aware Prediction engine uses the LLM itself to predict tool outcomes, evaluate response quality, and classify user sentiment. It generates prompts and parses responses -- it never makes LLM calls directly. The SDK pipeline handles the actual inference.

## What It Does

Three prediction capabilities, each with a `build_*_prompt()` / `parse_*_response()` pair:

1. **Tool Prediction**: Mental rehearsal before tool execution -- predicts success probability, quality, risk factors, and alternatives
2. **Response Evaluation**: Parallel quality assessment against the stated goal -- scores quality, goal alignment, and completeness
3. **Sentiment Classification**: LLM-based sentiment analysis replacing regex patterns -- scores satisfaction, frustration, confusion, urgency

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Mental Rehearsal"
    The prefrontal cortex runs "mental simulations" of actions before executing them. Before reaching for a cup, your motor cortex rehearses the movement and predicts the outcome. If the prediction suggests failure (the cup is too far), the plan is revised before execution.

    The Content-Aware Prediction engine implements this: before executing a tool call, the system asks a cheap LLM model to predict the outcome. If the prediction suggests high risk or low quality, the orchestrator can choose a different tool or approach.

## How It Works

### Tool Prediction (Mental Rehearsal)

```python
from corteX.engine.content_prediction import ContentPredictor

predictor = ContentPredictor()

# Build a prompt asking the LLM to predict tool outcome
prompt = predictor.build_tool_prediction_prompt(
    tool_name="code_interpreter",
    tool_args={"code": "import pandas as pd; df.head()"},
    context={"goal": "Analyze sales data", "task_type": "coding"},
    recent_history=[{"tool": "search", "outcome": "found dataset"}],
)

# After sending prompt to cheap model and getting response:
prediction = predictor.parse_tool_prediction_response(
    response_text=llm_response, tool_name="code_interpreter"
)
# ContentAwarePrediction(
#   predicted_success=0.85, predicted_quality=0.8,
#   risk_factors=[], suggested_alternatives=[]
# )
```

### Response Evaluation (Conflict Monitoring)

```python
# Evaluate how well a response achieves the goal
eval_prompt = predictor.build_evaluation_prompt(
    response_text=agent_response,
    goal="Summarize Q4 financial results",
    context={"step_number": 3, "tools_used": ["search", "calculator"]},
)

# Parse the evaluation
evaluation = predictor.parse_evaluation_response(eval_response)
# {"quality_score": 0.82, "goal_alignment": 0.9,
#  "completeness": 0.75, "issues": ["Missing revenue breakdown"]}
```

### Sentiment Classification

```python
# Classify user sentiment via LLM instead of regex
sentiment_prompt = predictor.build_sentiment_prompt(
    "This is taking forever and I still don't have an answer"
)

sentiment = predictor.parse_sentiment_response(llm_response)
# {"satisfaction": 0.15, "frustration": 0.85,
#  "confusion": 0.3, "urgency": 0.7}
```

### Prediction Caching

Results are cached with a configurable TTL to avoid redundant predictions:

```python
# Check cache before building a new prompt
cached = predictor.get_cached_prediction("code_interpreter", tool_args)
if cached is None:
    # Build prompt, call LLM, cache result
    predictor.cache_prediction(prediction, tool_args)
```

## When It Activates

- **Before tool execution**: Tool prediction prompt is sent to a cheap model in parallel
- **After step completion**: Response evaluation runs in parallel with the main flow
- **On user messages**: Sentiment classification replaces regex-based detection
- **Cache hits**: Skip prediction when identical tool+args were recently predicted

## API Reference

```python
from corteX.engine.content_prediction import (
    ContentPredictor,
    ContentPredictionConfig,
    ContentAwarePrediction,
    PredictionCache,
)
```
