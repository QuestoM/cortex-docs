# Cost Predictor API Reference

## Module: `corteX.observability.cost_predictor`

Estimate total plan cost BEFORE execution. Brain analogy: prefrontal cortex forecasting energy expenditure before action. Uses tenant DNA (average tokens per task type) combined with model pricing tables to produce cost predictions with confidence intervals. If predicted cost exceeds budget, the agent can ask for approval before proceeding.

## Classes

### `StepEstimate`

**Type**: `@dataclass`

Cost estimate for a single plan step.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `step_description` | `str` | Description of the plan step |
| `estimated_tokens` | `int` | Predicted token consumption |
| `estimated_cost_usd` | `float` | Predicted cost in USD |
| `model` | `str` | Model to be used |
| `confidence` | `float` | Confidence in this estimate (0.0-1.0) |
| `metadata` | `Dict[str, Any]` | Additional metadata |

---

### `CostPrediction`

**Type**: `@dataclass`

Full cost prediction for a plan with confidence interval.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `estimated_tokens` | `int` | Total predicted tokens |
| `estimated_cost_usd` | `float` | Total predicted cost in USD |
| `confidence_interval` | `Tuple[float, float]` | (low, high) cost bounds in USD |
| `breakdown` | `List[StepEstimate]` | Per-step cost breakdown |
| `over_budget` | `bool` | Whether estimated cost exceeds budget |
| `budget_remaining` | `float` | Remaining budget after predicted cost |
| `model_breakdown` | `Dict[str, float]` | Cost per model |

#### Methods

##### `summary`

```python
def summary(self) -> str
```

Human-readable prediction summary.

**Example output**: `"Predicted: 12,500 tokens, $0.0250 USD (within budget). CI: [$0.0175, $0.0375]. Budget remaining: $0.4750"`

---

### `CostPredictor`

Predict total plan cost before execution using step heuristics and tenant DNA.

#### Constructor

```python
CostPredictor()
```

#### Methods

##### `predict`

```python
def predict(
    self, plan_steps: List[Dict[str, Any]],
    tenant_dna: Optional[Dict[str, Any]] = None,
    model_pricing: Optional[Dict[str, float]] = None,
    budget: float = float("inf")
) -> CostPrediction
```

Predict cost for a plan before execution.

**Parameters**:

- `plan_steps` (`List[Dict[str, Any]]`): Steps with `description` and optional `model`, `complexity`, `estimated_tokens` keys.
- `tenant_dna` (`Optional[Dict[str, Any]]`): Tenant DNA with `avg_tokens_per_task`, `avg_task_complexity` keys.
- `model_pricing` (`Optional[Dict[str, float]]`): Model ID to price per 1K tokens. Default: $0.002/1K.
- `budget` (`float`): Total budget in USD. Used to compute `over_budget` flag.

**Returns**: `CostPrediction` with breakdown and confidence interval.

**Token estimation heuristics**:

| Complexity | Keywords | Base Tokens |
|-----------|----------|-------------|
| Simple | `lookup`, `get`, `read`, `check`, `list`, `count`, `echo` | 500 |
| Medium | (no match) | 1,500 |
| Complex | `analyze`, `generate`, `write`, `create`, `plan`, `summarize`, `compare`, `refactor`, `debug`, `transform`, `synthesize` | 4,000 |

Final estimate blends heuristic (60%) with tenant DNA average (40%).

**Confidence interval**:

| Condition | Low Multiplier | High Multiplier |
|-----------|---------------|----------------|
| No tenant DNA | 0.6x | 1.8x |
| With tenant DNA | 0.7x | 1.5x |

##### `is_within_budget`

```python
def is_within_budget(self, prediction: CostPrediction, budget: float) -> bool
```

Check if a prediction is within the given budget.

##### `get_calibration_stats`

```python
def get_calibration_stats(self) -> Dict[str, Any]
```

**Returns**: Dict with `history_size` and `avg_estimated_tokens`.

---

## Model Pricing Lookup

Price lookup follows a fallback chain:

1. Exact match: `pricing["gemini-3-pro-preview"]`
2. Prefix match: `pricing["gemini"]` matches `"gemini-3-pro-preview"`
3. Default: `$0.002` per 1K tokens

---

## Example

```python
from corteX.observability.cost_predictor import CostPredictor

predictor = CostPredictor()

prediction = predictor.predict(
    plan_steps=[
        {"description": "Read user configuration", "model": "gemini-3-flash-preview"},
        {"description": "Analyze database schema", "model": "gemini-3-pro-preview"},
        {"description": "Generate migration code", "model": "gemini-3-pro-preview"},
        {"description": "Write unit tests", "model": "gemini-3-flash-preview"},
    ],
    tenant_dna={
        "avg_tokens_per_task": 3000,
        "avg_task_complexity": 0.7,
    },
    model_pricing={
        "gemini-3-pro-preview": 0.005,
        "gemini-3-flash-preview": 0.001,
    },
    budget=0.50,
)

print(prediction.summary())
print(f"\nBreakdown:")
for step in prediction.breakdown:
    print(f"  {step.step_description}: {step.estimated_tokens} tokens, "
          f"${step.estimated_cost_usd:.4f} ({step.model})")

print(f"\nModel costs: {prediction.model_breakdown}")

if prediction.over_budget:
    print("WARNING: Plan exceeds budget! Seek approval.")
```

---

## See Also

- [Tenant DNA](../tenancy/dna.md) -- Provides `avg_tokens_per_task` for estimation
- [Quota Tracker](../tenancy/quota.md) -- Enforces token budgets at runtime
- [Metrics Collector](./metrics.md) -- Records actual costs for calibration
- [Decision Tracer](./tracer.md) -- Traces cost-related decisions
