# Assess Production Readiness Before Deploying

The production readiness scorer analyzes your engine and agent configuration and returns a 0-100 score with a letter grade, category breakdowns, and actionable recommendations. Run it before every deployment.

---

## Quick check

```python
from corteX.sdk import Engine
from corteX.sdk_config import EnterpriseConfig

engine = Engine(
    providers={"openai": {"api_key": "sk-..."}},
    enterprise_config=EnterpriseConfig(
        safety_level="strict",
        audit_log=True,
        max_tokens_per_session=50000,
    ),
)

result = engine.readiness_score(
    agent_config={
        "goal_tracking": True,
        "loop_prevention": True,
        "system_prompt": "You are a support agent for Acme Corp.",
        "tools": ["search_kb", "create_ticket"],
    }
)

print(f"Score: {result['score']}/100 ({result['grade']})")
print(f"Checks: {result['checks_passed']}/{result['checks_total']} passed")
print(f"Categories: {result['category_scores']}")
for rec in result["top_recommendations"]:
    print(f"  - {rec}")
```

## What it checks

The scorer evaluates 20 checks across 6 categories:

| Category | Weight | What it checks |
|---|---|---|
| **Safety** | 25% | Safety level, blocked topics, tool restrictions, model allowlist, token budget |
| **Reliability** | 20% | Goal tracking, loop prevention, provider configuration |
| **Observability** | 15% | Audit logging, session recording, weight persistence |
| **Performance** | 15% | Context management, weight tuning, multi-provider routing |
| **Resilience** | 15% | Fallback providers, system prompt, feedback collection |
| **Compliance** | 10% | Compliance frameworks, data retention, audit trail |

## Grade scale

| Grade | Score | Meaning |
|---|---|---|
| **A** | 90-100 | Production-ready |
| **B** | 80-89 | Good, minor improvements suggested |
| **C** | 70-79 | Needs attention before production |
| **D** | 60-69 | Significant gaps |
| **F** | 0-59 | Not ready for production |

## Reaching an A grade

The most common gaps and how to close them:

```python
engine = Engine(
    providers={
        "openai": {"api_key": "sk-..."},
        "gemini": {"api_key": "AIza..."},       # Fallback provider
    },
    enterprise_config=EnterpriseConfig(
        safety_level="strict",                    # Not "permissive"
        audit_log=True,                           # Enable audit trail
        compliance=["soc2", "gdpr"],              # Declare frameworks
        data_retention="session",                 # Not "none"
        max_tokens_per_session=100000,            # Set a budget
        allowed_models=["gpt-4o", "gemini-2.5-pro"],
        blocked_topics=["weapons", "illegal"],
        feedback_collection="enterprise",
    ),
    weight_persistence_path="/var/cortex/weights.json",
)

agent = engine.create_agent(
    name="production_agent",
    system_prompt="You are a customer support agent for Acme Corp.",
    tools=[search_kb, create_ticket],              # Explicit tool list
    goal_tracking=True,
    loop_prevention=True,
    enable_session_recording=True,
    weight_config=WeightConfig(autonomy=0.5),
    context_config=ContextManagementConfig(enabled=True),
)
```

!!! tip "Run in CI/CD"
    Add a readiness check to your deployment pipeline. Fail the deploy if the score drops below your threshold (e.g., 80 for staging, 90 for production).

---

## Next steps

- [Handle Service Failures Gracefully](graceful-degradation.md) - improve your resilience score
- [Monitor Your Agent](observability.md) - track runtime metrics after deployment
