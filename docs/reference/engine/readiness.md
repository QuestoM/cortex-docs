# Production Readiness Scorer API Reference

## Module: `corteX.engine.readiness` + `corteX.engine.readiness_checks`

Automated agent deployment readiness assessment. Analyzes engine, agent, and session configuration to score how ready an agent is for production, returning a graded report with actionable recommendations across 6 categories.

Brain analogy: The anterior cingulate cortex monitors ongoing performance and detects when the system is not meeting its own standards. The readiness scorer is a pre-deployment version - checking standards BEFORE the agent encounters real users.

---

## Classes

### `ReadinessCategory`

**Type**: `str, Enum`

Categories of production readiness checks.

| Value | Description |
|-------|-------------|
| `RELIABILITY` | Goal tracking, loop prevention, provider configuration |
| `OBSERVABILITY` | Audit logging, session recording, weight persistence |
| `SAFETY` | Safety level, blocked topics, tool restrictions, model allowlist, token budget |
| `PERFORMANCE` | Context management, weight tuning, multi-provider routing |
| `RESILIENCE` | Fallback providers, system prompt, feedback collection |
| `COMPLIANCE` | Compliance frameworks, data retention, audit trail |

---

### `ReadinessCheck`

**Type**: `@dataclass`

A single readiness check with pass/fail/warning status.

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Machine-readable check name (e.g. `"goal_tracking_enabled"`). |
| `category` | `ReadinessCategory` | Which category this check belongs to. |
| `status` | `str` | One of `"pass"`, `"fail"`, or `"warning"`. |
| `message` | `str` | Human-readable description of what this check validates. |
| `weight` | `float` | Contribution to category score (0.0-1.0). |
| `recommendation` | `str` | Actionable fix if not passing. |

---

### `ReadinessReport`

**Type**: `@dataclass`

Full readiness assessment result.

| Attribute | Type | Description |
|-----------|------|-------------|
| `score` | `int` | Overall score 0-100. |
| `grade` | `str` | Letter grade: A (>=90), B (>=80), C (>=70), D (>=60), F (<60). |
| `checks` | `List[ReadinessCheck]` | All individual checks that were run. |
| `category_scores` | `Dict[ReadinessCategory, float]` | Per-category scores (0.0-1.0). |
| `top_recommendations` | `List[str]` | Top 5 recommendations sorted by impact. |

---

### `ProductionReadinessScorer`

**Type**: `class`

Scores agent production readiness on a 0-100 scale with weighted categories.

#### Category Weights

| Category | Weight | Description |
|----------|--------|-------------|
| Safety | 25% | Guardrails, tool restrictions, PII, model allowlist |
| Reliability | 20% | Goal tracking, loop prevention, providers |
| Observability | 15% | Audit logging, session recording, weight persistence |
| Performance | 15% | Context management, weight tuning, multi-provider |
| Resilience | 15% | Fallback providers, system prompt, feedback |
| Compliance | 10% | Frameworks, data retention, audit trail |

### Methods

#### `score(engine_config, agent_config, session_config=None) -> ReadinessReport`

Run all readiness checks and return a scored report.

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `engine_config` | `Dict[str, Any]` | (required) | Engine-level config (providers, enterprise_config, etc.). |
| `agent_config` | `Dict[str, Any]` | (required) | Agent-level config (goal_tracking, tools, etc.). |
| `session_config` | `Optional[Dict[str, Any]]` | `None` | Optional session-level config overrides. |

**Returns:** `ReadinessReport` with score, grade, checks, and recommendations.

**Scoring logic:**
- `"pass"` status contributes full weight to category score.
- `"warning"` status contributes 50% of weight.
- `"fail"` status contributes 0%.
- Category scores are weighted-averaged into the overall 0-100 score.

---

## Check Functions (readiness_checks module)

Six category-specific check functions, each returning `List[ReadinessCheck]`:

| Function | Category | Checks |
|----------|----------|--------|
| `check_reliability(cfg)` | RELIABILITY | goal_tracking, loop_prevention, providers_configured |
| `check_observability(cfg)` | OBSERVABILITY | audit_logging, session_recording, weight_persistence |
| `check_safety(cfg)` | SAFETY | safety_level, blocked_topics, tools_set, allowed_models, token_budget |
| `check_performance(cfg)` | PERFORMANCE | context_management, weight_config, multi_provider |
| `check_resilience(cfg)` | RESILIENCE | fallback_provider, system_prompt, feedback_collection |
| `check_compliance(cfg)` | COMPLIANCE | compliance_frameworks, data_retention, audit_trail |

---

## Usage Example

```python
from corteX.engine.readiness import ProductionReadinessScorer

scorer = ProductionReadinessScorer()
report = scorer.score(
    engine_config={
        "providers": {"openai": {"api_key": "sk-..."}},
        "enterprise_config": {
            "audit_log": True,
            "safety_level": "strict",
            "max_tokens_per_session": 50000,
        },
    },
    agent_config={
        "goal_tracking": True,
        "loop_prevention": True,
        "tools": ["search", "calculator"],
        "system_prompt": "You are a helpful assistant.",
    },
)

print(f"Score: {report.score}/100 ({report.grade})")
for cat, score in report.category_scores.items():
    print(f"  {cat.value}: {score:.0%}")

if report.top_recommendations:
    print("\nTop recommendations:")
    for rec in report.top_recommendations:
        print(f"  - {rec}")
```

---

## See Also

- [Bias Detector API](./bias-detector.md)
- [Adaptive Budget API](./adaptive-budget.md)
- [Degradation Chain API](./degradation.md)
