# A/B Test Manager API Reference

## Module: `corteX.engine.ab_test_manager`

Deterministic model comparison with statistical rigor. Uses Mann-Whitney U test for non-parametric significance testing. Hash-based assignment ensures stable variant selection.

Brain analogy: Basal ganglia (action selection via competition), dopamine system (reward comparison).

---

## Classes

### `ABTestStatus`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `ACTIVE` | Test is running and accepting samples |
| `PAUSED` | Test is paused (no new samples) |
| `COMPLETED` | Both variants reached max_samples |
| `CANCELLED` | Test was manually cancelled |

### `ABTest`

**Type**: `@dataclass`

Definition of an A/B test between two models.

| Attribute | Type | Description |
|-----------|------|-------------|
| `test_name` | `str` | Unique test identifier |
| `model_a` | `str` | Control model ID |
| `model_b` | `str` | Challenger model ID |
| `traffic_split` | `float` | Fraction routed to model A (default: 0.5) |
| `start_time` | `float` | Test start timestamp |
| `max_samples` | `int` | Maximum samples per variant (default: 200) |
| `status` | `ABTestStatus` | Current test status |
| `description` | `str` | Test description |

### `ABMetrics`

**Type**: `@dataclass`

Outcome metrics for a single request.

| Attribute | Type | Default |
|-----------|------|---------|
| `quality_score` | `float` | 0.0 |
| `latency_ms` | `float` | 0.0 |
| `cost_usd` | `float` | 0.0 |
| `error_rate` | `float` | 0.0 |
| `timestamp` | `float` | current time |

### `ABTestResult`

**Type**: `@dataclass`

Statistical evaluation of an A/B test.

| Attribute | Type | Description |
|-----------|------|-------------|
| `test_name` | `str` | Test identifier |
| `winner` | `Optional[str]` | Winning model ID (None if not significant) |
| `p_value` | `float` | Statistical p-value |
| `effect_size` | `float` | Rank-biserial effect size [-1, 1] |
| `samples_a` | `int` | Samples for model A |
| `samples_b` | `int` | Samples for model B |
| `significant` | `bool` | Whether result is statistically significant |
| `metric_summary_a` | `Dict[str, float]` | Mean metrics for model A |
| `metric_summary_b` | `Dict[str, float]` | Mean metrics for model B |
| `explanation` | `str` | Human-readable explanation |

---

### `ABTestManager`

Manages A/B tests between model variants with deterministic hash-based assignment and Mann-Whitney U significance testing.

#### Constructor

```python
ABTestManager(significance_level: float = 0.05)
```

**Parameters**:

- `significance_level` (float): P-value threshold for significance. Default: 0.05

#### Methods

##### `create_test`

```python
def create_test(
    self, name: str, model_a: str, model_b: str,
    split: float = 0.5, max_samples: int = 200,
    description: str = "",
) -> ABTest
```

Create a new A/B test. Traffic split is clamped to [0.1, 0.9].

##### `get_variant`

```python
def get_variant(self, test_name: str, request_id: str) -> str
```

Deterministically assign a variant via `hash(request_id:test_name)`. Same request_id always gets the same variant.

##### `record_outcome`

```python
def record_outcome(self, test_name: str, model: str, metrics: ABMetrics) -> None
```

Record an outcome for a model variant. Auto-completes the test when both sides reach `max_samples`.

##### `evaluate`

```python
def evaluate(self, test_name: str) -> ABTestResult
```

Evaluate using Mann-Whitney U test on `quality_score`. Requires >= 5 samples per variant.

##### `pause_test` / `cancel_test`

```python
def pause_test(self, test_name: str) -> bool
def cancel_test(self, test_name: str) -> bool
```

Pause or cancel a test. Returns `True` if status was changed.

---

## Usage Example

```python
from corteX.engine.ab_test_manager import ABTestManager, ABMetrics

manager = ABTestManager(significance_level=0.05)

# Create test
manager.create_test("flash_vs_pro", "gemini-3-flash", "gemini-3-pro")

# Route requests
for request_id in request_ids:
    model = manager.get_variant("flash_vs_pro", request_id)
    result = await call_model(model, request)
    manager.record_outcome("flash_vs_pro", model, ABMetrics(
        quality_score=result.quality, latency_ms=result.latency,
    ))

# Evaluate
result = manager.evaluate("flash_vs_pro")
if result.significant:
    print(f"Winner: {result.winner} (p={result.p_value:.4f})")
```

---

## See Also

- [Model Routing Concept](../../concepts/model-routing.md)
- [Provider Health Monitor API](./provider-health.md)
