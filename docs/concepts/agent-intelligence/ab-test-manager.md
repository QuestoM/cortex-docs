# A/B Test Manager

The A/B Test Manager enables deterministic, statistically rigorous comparison of model variants or agent configurations. It uses hash-based variant assignment for stable traffic splitting and the Mann-Whitney U test for non-parametric significance testing - no scipy dependency required. This lets developers answer questions like "Does gemini-2.5-pro produce better code than gemini-2.5-flash for my use case?" with real statistical confidence.

## How It Works

An A/B test is defined by two model variants (A and B), a traffic split ratio, and a maximum sample size. The flow is:

1. **Create a test** with `create_test("code_quality", model_a="gemini-2.5-pro", model_b="gemini-2.5-flash")`
2. **Assign variants deterministically** - `get_variant(test_name, request_id)` hashes the request ID with SHA-256 to produce a stable bucket assignment. The same request always gets the same variant.
3. **Record outcomes** - after each request, call `record_outcome()` with an `ABMetrics` dataclass containing `quality_score`, `latency_ms`, `cost_usd`, and `error_rate`
4. **Evaluate** - `evaluate()` runs the Mann-Whitney U test on quality scores and returns an `ABTestResult` with p-value, effect size (rank-biserial r), and a winner if statistically significant

Tests auto-complete when both sides reach `max_samples`. They can also be paused or cancelled manually.

## Key Features

- **Zero external dependencies** - Mann-Whitney U test and p-value approximation implemented in pure Python
- **Deterministic assignment** - SHA-256 hash of `"{request_id}:{test_name}"` ensures repeatable variant selection
- **Configurable significance level** - default alpha = 0.05, adjustable per manager instance
- **Effect size reporting** - rank-biserial correlation r in [-1, 1] quantifies how large the difference is
- **Multi-metric tracking** - records quality, latency, cost, and error rate per outcome
- **Auto-completion** - test transitions to COMPLETED when both variants reach max samples
- **Lifecycle management** - tests can be ACTIVE, PAUSED, COMPLETED, or CANCELLED
- **Minimum sample guard** - evaluation requires at least 5 samples per variant before running statistics

## Integration

The A/B Test Manager works alongside:

- **Model Mosaic** - test single-model vs. multi-model patterns
- **Provider Health** - variant assignment can be overridden if a provider is unhealthy
- **Decision Log** - A/B assignment decisions are recorded with `DecisionType.PATTERN_SELECTION`
- **Session Orchestrator** - variant assignment happens during model selection in the turn pipeline

## Usage Example

```python
from corteX.engine.ab_test_manager import ABTestManager, ABMetrics

manager = ABTestManager(significance_level=0.05)

# Create a test
test = manager.create_test(
    name="code_gen_model",
    model_a="gemini-2.5-pro",
    model_b="gemini-2.5-flash",
    split=0.5,
    max_samples=100,
)

# For each request, get deterministic variant
variant = manager.get_variant("code_gen_model", request_id="req-abc123")
# variant = "gemini-2.5-pro" or "gemini-2.5-flash" (stable per request)

# After execution, record outcome
manager.record_outcome("code_gen_model", variant, ABMetrics(
    quality_score=0.85,
    latency_ms=1200,
    cost_usd=0.003,
    error_rate=0.0,
))

# After enough samples, evaluate
result = manager.evaluate("code_gen_model")
print(f"Winner: {result.winner}")        # e.g., "gemini-2.5-pro"
print(f"p-value: {result.p_value:.4f}")  # e.g., 0.0231
print(f"Significant: {result.significant}")  # True
print(result.explanation)
```

## See Also

- [Model Mosaic](model-mosaic.md) - multi-model patterns that can be A/B tested
- [Provider Health](provider-health.md) - health data for failover during tests
- [Decision Log](decision-log.md) - audit trail for variant assignments
