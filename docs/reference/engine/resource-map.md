# Resource Map

`corteX.engine.resource_map` -- Resource Homunculus for non-uniform resource allocation inspired by the cortical homunculus.

---

## Overview

The somatosensory cortex devotes more area to body parts requiring finer discrimination
(fingertips, lips) and less to areas tolerating coarser input (trunk, back). Prof. Segev:
"The fingertips are represented by a large piece of cortex... the entire back takes up less
space than the whole face."

This module applies the same principle to AI agent resource budgets:

- Frequent tasks get more tokens, retries, and verification
- Critical but rare tasks get high verification depth
- Easy, infrequent tasks get minimal resources
- When usage patterns shift, resources reorganize (cortical reorganization)

Architecture:

- **ResourceAllocation** -- Dataclass defining the resource envelope per task type
- **UsageTracker** -- Bayesian telemetry with Beta/Gamma posteriors and temporal decay
- **ResourceHomunculus** -- Main allocation engine with cortical map and reorganization
- **AdaptiveThrottler** -- Rate-limiting based on homunculus allocations

---

## Dataclass: ResourceAllocation

The resource envelope for a particular task type.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `task_type` | `str` | (required) | Task category identifier. |
| `token_budget` | `float` | `1.0` | Relative token multiplier (0.25 to 3.0). |
| `max_tool_retries` | `int` | `2` | Maximum tool-call retries. |
| `verification_depth` | `int` | `1` | 0=none, 1=basic, 2=thorough, 3=exhaustive. |
| `model_tier` | `str` | `"worker"` | `"orchestrator"`, `"worker"`, or `"background"`. |
| `parallel_evaluations` | `int` | `1` | Population-coding parallel evaluations (1-5). |
| `context_window_ratio` | `float` | `0.5` | Fraction of context window to allocate (0.0-1.0). |
| `priority` | `float` | `0.5` | Scheduling priority (0.0-1.0). |

---

## Class: UsageTracker

Bayesian telemetry for per-task-type usage patterns. Maintains `BetaDistribution` for success
rate and `GammaDistribution` for latency, plus counters for quality and frequency.

### Constructor

```python
UsageTracker(decay_factor: float = 0.98)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `record_usage` | `(task_type: str, success: bool, quality: float, latency_ms: float) -> None` | Record a task execution outcome. |
| `get_frequency` | `(task_type: str) -> float` | Relative frequency [0.0, 1.0]. |
| `get_success_rate` | `(task_type: str) -> float` | Posterior mean success rate. |
| `get_expected_latency` | `(task_type: str) -> float` | Posterior mean latency (ms). |
| `get_quality_mean` | `(task_type: str) -> float` | Mean observed quality. |
| `get_quality_variance` | `(task_type: str) -> float` | Quality variance. High = unpredictable. |
| `get_criticality_score` | `(task_type: str) -> float` | Composite: frequency * success_impact * quality_variance_factor. |
| `apply_decay` | `() -> None` | Temporal decay on all distributions and counts. |
| `get_all_task_types` | `() -> List[str]` | All tracked types ordered by frequency. |

---

## Class: ResourceHomunculus

Main allocation engine with cortical map and reorganization support.

### Constructor

```python
ResourceHomunculus(
    total_budget: float = 100.0,
    frequency_weight: float = 0.4,
    criticality_weight: float = 0.35,
    quality_weight: float = 0.25,
    decay_factor: float = 0.98,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `allocate` | `(task_type: str) -> ResourceAllocation` | Get the allocation for a task type. Returns default if first encounter. |
| `record_outcome` | `(task_type, success, quality, latency_ms) -> None` | Record a task outcome. Call `reorganize()` periodically to rebalance. |
| `reorganize` | `() -> Dict[str, float]` | Rebalance all allocations based on current usage statistics. Returns raw scores. |
| `get_cortical_map` | `() -> Dict[str, ResourceAllocation]` | The full homunculus: task_type to ResourceAllocation. |
| `get_over_allocated` | `() -> List[str]` | Task types getting more resources than justified. |
| `get_under_allocated` | `() -> List[str]` | Task types needing more resources. |
| `set_budget` | `(total_budget: float) -> None` | Adjust the total resource budget. |
| `get_stats` | `() -> Dict[str, Any]` | Comprehensive diagnostics. |
| `to_dict` / `from_dict` | | Full serialization. |

### Allocation Formula

```
raw_score = freq_weight * frequency + crit_weight * criticality + qual_weight * quality_sensitivity
allocation = normalize(raw_score) * total_budget
```

Higher raw_score translates to: more token_budget (up to 3x), more retries (up to 5),
deeper verification (up to level 3), orchestrator model tier, and more parallel evaluations.

---

## Class: AdaptiveThrottler

Rate-limits operations based on homunculus allocations. High-allocation tasks pass freely;
low-allocation tasks face exponential backoff on failures.

### Constructor

```python
AdaptiveThrottler(
    homunculus: ResourceHomunculus,
    low_threshold: float = 0.25,
    high_threshold: float = 0.70,
    base_delay_ms: float = 100.0,
    max_delay_ms: float = 10_000.0,
    backoff_exponent: float = 1.5,
)
```

### Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `should_throttle` | `(task_type: str) -> bool` | Whether to delay this request. |
| `get_throttle_delay` | `(task_type: str) -> float` | Delay in ms. 0.0 if no throttle. |
| `record_failure` | `(task_type: str) -> None` | Increment failure streak. |
| `record_success` | `(task_type: str) -> None` | Reset failure streak. |
| `get_effective_retries` | `(task_type: str) -> int` | Retries minus failure penalty. |
| `get_effective_token_budget` | `(task_type: str) -> float` | Budget reduced by failure streak (15%/step). |

---

## Example

```python
from corteX.engine.resource_map import ResourceHomunculus, AdaptiveThrottler

homunculus = ResourceHomunculus(total_budget=100.0)

# Record task outcomes
for _ in range(20):
    homunculus.record_outcome("code_edit", success=True, quality=0.8, latency_ms=500)
    homunculus.record_outcome("web_search", success=True, quality=0.6, latency_ms=2000)
    homunculus.record_outcome("planning", success=False, quality=0.3, latency_ms=800)

# Reorganize the cortical map
raw_scores = homunculus.reorganize()

# Get allocations
code_alloc = homunculus.allocate("code_edit")
print(f"code_edit: budget={code_alloc.token_budget:.2f}, tier={code_alloc.model_tier}")

# Adaptive throttling
throttler = AdaptiveThrottler(homunculus)
throttler.record_failure("planning")
throttler.record_failure("planning")
print(f"Throttle planning? {throttler.should_throttle('planning')}")
print(f"Delay: {throttler.get_throttle_delay('planning'):.0f}ms")
```
