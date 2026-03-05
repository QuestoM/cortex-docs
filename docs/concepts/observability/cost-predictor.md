# Cost Predictor

The Cost Predictor estimates total plan cost before execution begins,
acting like the prefrontal cortex forecasting energy expenditure
before committing to an action. If predicted cost exceeds budget, the
agent can request approval rather than running up unexpected bills.

## How It Works

The `CostPredictor` takes a list of plan steps and produces a
`CostPrediction` with per-step breakdowns and confidence intervals.
For each step, token count is estimated through a three-tier process:

1. **Explicit tokens** - if the plan step specifies `estimated_tokens`,
   that value is used directly (confidence: 0.95)
2. **Keyword classification** - step descriptions are matched against
   keyword sets to determine complexity:
   - Simple (500 tokens): lookup, get, read, check, list, count, echo
   - Complex (4,000 tokens): analyze, generate, write, create, plan, summarize, compare, refactor, debug
   - Medium (1,500 tokens): everything else
3. **Tenant DNA blending** - the heuristic estimate is blended 60/40
   with the tenant's historical average tokens per task

Cost is then computed by multiplying tokens by model pricing (per
1,000 tokens). The predictor uses exact match, prefix match, or a
conservative default of $0.002/1K tokens.

Confidence intervals narrow when tenant DNA is available (0.7x-1.5x)
versus heuristic-only estimates (0.6x-1.8x).

## Key Features

- **Pre-execution prediction** - know the cost before spending anything
- **Per-step breakdown** - `StepEstimate` for each plan step with tokens, cost, model, and confidence
- **Confidence intervals** - range tightens with tenant DNA availability
- **Budget checking** - `over_budget` flag and `budget_remaining` computed automatically
- **Model-level breakdown** - `model_breakdown` shows cost per model
- **Tenant DNA integration** - blends historical averages with heuristic estimates
- **Calibration history** - records estimates for future accuracy improvement
- **Human-readable summary** - `prediction.summary()` returns a formatted cost report

## Integration

The Cost Predictor is called by the orchestrator during plan
evaluation, before any LLM calls are made. The prediction feeds into
the adaptive budget system - if the predicted cost exceeds the
session budget, the agent can pause for approval or re-plan with
cheaper models.

Tenant DNA (historical token averages) is sourced from the enterprise
feedback system. Over time, predictions become more accurate as the
calibration history grows with actual execution data.

## Usage Example

```python
from corteX.observability.cost_predictor import CostPredictor

predictor = CostPredictor()

# Define plan steps
steps = [
    {"description": "read user preferences", "model": "gemini-2.5-flash"},
    {"description": "analyze quarterly report and summarize findings",
     "model": "gemini-2.5-pro"},
    {"description": "generate email draft", "model": "gemini-2.5-pro"},
]

# Predict cost with model pricing and budget
prediction = predictor.predict(
    plan_steps=steps,
    tenant_dna={"avg_tokens_per_task": 2000},
    model_pricing={
        "gemini-2.5-flash": 0.0001,   # $0.0001 per 1K tokens
        "gemini-2.5-pro": 0.00125,    # $0.00125 per 1K tokens
    },
    budget=0.05,  # $0.05 budget
)

print(prediction.summary())
# "Predicted: 6,200 tokens, $0.0065 USD (within budget). CI: [$0.0046, $0.0098]..."

# Check budget
if prediction.over_budget:
    print("Plan exceeds budget - requesting approval")

# Inspect per-step breakdown
for step in prediction.breakdown:
    print(f"  {step.step_description}: {step.estimated_tokens} tokens, "
          f"${step.estimated_cost_usd:.4f} (confidence: {step.confidence})")

# Calibration stats
stats = predictor.get_calibration_stats()
# {"history_size": 3, "avg_estimated_tokens": 2067}
```

## See Also

- [Decision Tracer](decision-tracer.md) - traces why decisions were made
- [Metrics Collector](metrics-collector.md) - collects actual runtime costs for calibration
- [Tenant Audit Stream](audit-stream.md) - logs cost events for compliance
