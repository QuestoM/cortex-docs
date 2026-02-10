# Safety Policies

The `SafetyPolicy` is the executive control layer of corteX -- the "prefrontal cortex" that enforces behavioral boundaries on agents regardless of what the weight engine has learned.

## Safety Levels

Four levels provide increasing restriction:

| Level | Description | User Overrides |
|-------|-------------|---------------|
| `PERMISSIVE` | Minimal restrictions. Suitable for internal dev tools. | Allowed |
| `MODERATE` | Default. PII detection and content filtering enabled. | Allowed |
| `STRICT` | All protections enabled. Human approval required for high-risk actions. | Limited |
| `LOCKED` | No user overrides permitted. Full admin control. | None |

```python
from corteX.enterprise.config import SafetyPolicy, SafetyLevel

policy = SafetyPolicy(
    level=SafetyLevel.STRICT,
    blocked_topics=["competitor_analysis", "employee_salary"],
    blocked_patterns=[r"\b\d{3}-\d{2}-\d{4}\b"],  # SSN pattern
    max_autonomy=0.5,
    require_human_approval=["file_delete", "deploy_production"],
    pii_detection=True,
    content_filtering=True,
    injection_protection=True,
)
```

## Content Filtering

When `content_filtering=True`, the agent's outputs are checked against blocked topics and blocked patterns before being delivered:

```python
# Topic blocking
policy.allows_topic("quarterly revenue")      # True
policy.allows_topic("competitor analysis")    # False

# Pattern blocking (regex)
policy.blocked_patterns = [r"\b\d{3}-\d{2}-\d{4}\b"]  # Block SSN patterns
```

## PII Detection

When `pii_detection=True`, the system automatically detects and masks personally identifiable information in both inputs and outputs. This includes:

- Social Security Numbers
- Credit card numbers
- Email addresses (configurable)
- Phone numbers (configurable)

## Prompt Injection Protection

When `injection_protection=True`, the system guards against prompt injection attacks -- attempts by malicious input to override the agent's system instructions.

## Autonomy Capping

The `max_autonomy` field (0.0-1.0) places a hard cap on how independently the agent can act, regardless of what the weight engine's behavioral autonomy weight says:

```python
# Weight engine might learn autonomy=0.9, but enterprise caps it
effective_autonomy = min(
    weight_engine.behavioral.get("autonomy"),  # Learned: 0.9
    safety_policy.max_autonomy,                 # Cap: 0.5
)
# Result: 0.5
```

This is implemented in `WeightEngine.get_effective_autonomy()`.

## Human Approval Gates

Specific actions can require explicit human approval before execution:

```python
policy = SafetyPolicy(
    require_human_approval=[
        "file_delete",
        "deploy_production",
        "database_write",
    ]
)
```

The orchestrator checks this list before executing any action and routes to a `BLOCKING` approval flow when a match is found.

## Integration with the Weight Engine

The `EnterpriseWeights` tier in the weight engine reflects admin-configured safety rules. These weights are *set*, not *learned* -- the brain engine does not override admin intent:

```python
engine = WeightEngine()
engine.enterprise.set("safety_strictness", 0.9)
engine.enterprise.set("max_autonomy_level", 0.5)
engine.enterprise.set("blocked_topics", ["salary_info"])

# Users can only override keys in the _user_overridable set
engine.enterprise.user_override("safety_strictness", 0.3)  # May be allowed
engine.enterprise.user_override("blocked_topics", [])       # Denied
```
