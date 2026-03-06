# Handle Service Failures Gracefully

When LLM providers go down, rate limits hit, or tools fail, your agent shouldn't crash silently. The degradation chain provides a 5-level fallback from full capability to an honest failure message - so your users always get a response.

---

## The 5 degradation levels

| Level | Name | What happens |
|---|---|---|
| 1 | **FULL** | All features active (normal operation) |
| 2 | **REDUCED_TOOLS** | Non-essential tools disabled, core tools only |
| 3 | **CACHED** | Use cached/pre-computed responses where possible |
| 4 | **TEMPLATE** | Fall back to templated responses |
| 5 | **HONEST_FAILURE** | Tell the user the system is degraded |

## Check current state

```python
from corteX.sdk import Engine

engine = Engine(providers={"openai": {"api_key": "sk-..."}})
agent = engine.create_agent(name="support", system_prompt="You help users.")
session = agent.start_session(user_id="user-1")

state = session.get_degradation_state()
print(f"Level: {state['current_level']} ({state['current_level_name']})")
```

## Manual degradation control

For testing or when you detect issues externally:

```python
# Degrade one level (e.g., provider having issues)
state = session.force_degrade("OpenAI returning 429 rate limit errors")
print(f"Now at: {state['current_level_name']}")  # REDUCED_TOOLS

# Degrade again
state = session.force_degrade("Rate limits persisting")
print(f"Now at: {state['current_level_name']}")  # CACHED

# Recover one level when things improve
state = session.force_recover()
print(f"Recovered to: {state['current_level_name']}")  # REDUCED_TOOLS
```

## Integration with circuit breakers

The degradation chain works alongside corteX's built-in circuit breaker (`corteX.resilience.AgentCircuitBreaker`). When the circuit breaker trips after consecutive failures, it can trigger automatic degradation:

```python
# In your error handling
try:
    response = await session.run(user_message)
except ProviderError:
    session.force_degrade("Provider error detected")
    # Agent continues at reduced capability
```

## Best practices

!!! tip "Log every degradation"
    Always pass a descriptive reason to `force_degrade()`. This creates an audit trail for post-incident analysis.

!!! tip "Test degradation in staging"
    Deliberately trigger each level in staging to verify your user experience at every degradation state. Users should never see a blank screen or cryptic error.

!!! warning "Recovery is manual"
    The chain doesn't auto-recover. Your monitoring system should call `force_recover()` when health checks pass again.

---

## Next steps

- [Assess Production Readiness](readiness-check.md) - the readiness scorer checks your resilience configuration
- [Monitor Your Agent](observability.md) - detect when degradation is needed
