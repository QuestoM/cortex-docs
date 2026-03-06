# Deploy to Production

In this tutorial you will take an agent from development to production. You will configure enterprise safety controls, set up audit logging, manage secrets through environment variables, add structured logging, monitor session statistics, consider on-premises deployment, and tune performance for high-throughput workloads.

---

## What you will build

- A production-ready agent with full enterprise configuration
- Safety controls: blocked topics, PII detection, and autonomy caps
- Audit logging to file for compliance
- Environment variable management for secrets
- Structured logging with Python's `logging` module
- A monitoring dashboard that tracks session and brain metrics
- Performance tuning for high-throughput deployments

## Prerequisites

- Python 3.11+
- corteX installed: `pip install cortex-engine[openai]`
- An OpenAI API key
- Completion of at least one earlier tutorial (any agent will work)

---

## Step 1: Set up the project

```bash
mkdir production-agent && cd production-agent
touch production.py
touch .env
```

---

## Step 2: Manage secrets with environment variables

Never hardcode API keys in source files. Use a `.env` file for local development and environment variables in production.

```bash title=".env"
OPENAI_API_KEY=sk-your-actual-key-here
CORTEX_SAFETY_LEVEL=strict
CORTEX_AUDIT_PATH=/var/log/cortex/audit.log
CORTEX_LOG_LEVEL=INFO
```

Load environment variables at the top of your script. In production, your deployment platform (Kubernetes, Docker, systemd) will inject these directly.

```python title="production.py"
import asyncio
import logging
import os
import time
import cortex

# Load from .env file in development (pip install python-dotenv)
try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass  # In production, env vars are set by the platform
```

!!! warning "Never commit .env files"
    Add `.env` to your `.gitignore` immediately. Leaked API keys are the most common security incident in AI applications.

    ```text title=".gitignore"
    .env
    *.log
    __pycache__/
    ```

---

## Step 3: Configure structured logging

Set up Python's `logging` module with a structured format that includes timestamps and log levels. This integrates with any log aggregation system (ELK, Datadog, CloudWatch).

```python title="production.py"
log_level = os.environ.get("CORTEX_LOG_LEVEL", "INFO")
logging.basicConfig(
    level=getattr(logging, log_level),
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger("cortex.production")
```

---

## Step 4: Configure enterprise safety

Enterprise safety controls enforce behavioral boundaries that the weight engine cannot override. Configure them based on your organization's requirements.

```python title="production.py"
safety_level = os.environ.get("CORTEX_SAFETY_LEVEL", "strict")

enterprise = cortex.EnterpriseConfig(
    safety_level=safety_level,                   # (1)!
    blocked_topics=[
        "competitor_info",
        "internal_salary",
        "unreleased_products",
        "legal_advice",
    ],
    audit_log=True,                              # (2)!
    pii_detection=True,                          # (3)!
    max_autonomy=0.5,                            # (4)!
    require_human_approval=["delete_account"],    # (5)!
)
```

1. Safety levels: `permissive` (dev), `moderate` (default), `strict` (production), `locked` (regulated industries). Use `strict` or `locked` in production.
2. Enable audit logging for compliance. Every tool call, safety activation, and model routing decision is recorded.
3. PII detection automatically masks Social Security numbers, credit card numbers, and other sensitive patterns in both inputs and outputs.
4. Hard cap on agent autonomy. Even if the weight engine learns `autonomy=0.9`, the effective value will be clamped to 0.5.
5. Specific tool actions that require explicit human approval before execution.

!!! tip "Safety levels in practice"
    | Environment | Recommended Level |
    |-------------|-------------------|
    | Local development | `permissive` |
    | Staging / QA | `moderate` |
    | Production (general) | `strict` |
    | Regulated (HIPAA, SOX) | `locked` |

---

## Step 5: Set up audit logging

Configure where audit logs are written. For compliance, you typically want file-based logs with long retention.

```python title="production.py"
audit_path = os.environ.get("CORTEX_AUDIT_PATH", "./audit.log")
logger.info(f"Audit log path: {audit_path}")
logger.info(f"Safety level: {safety_level}")
logger.info(f"PII detection: enabled")
logger.info(f"Blocked topics: {enterprise.blocked_topics}")
```

!!! info "Audit log contents"
    When `audit_log=True`, corteX records:

    - Every tool invocation (name, arguments, result, latency)
    - Every safety policy activation (blocked topic detected, PII masked)
    - Every model routing decision (System 1 vs. System 2, model selected)
    - Session lifecycle events (start, close, consolidation)

    Conversation content is **not** logged by default for privacy. Enable it explicitly with `log_messages=True` if your compliance framework requires it.

---

## Step 6: Build the production engine and agent

```python title="production.py"
async def main():
    logger.info("Starting production agent...")
    start_time = time.time()

    # 1. Create the engine with environment-based configuration
    engine = cortex.Engine(
        providers={
            "openai": {
                "api_key": os.environ["OPENAI_API_KEY"],  # (1)!
            },
        },
        orchestrator_model="gpt-4o",
        worker_model="gpt-4o-mini",
    )

    # 2. Define a simple tool for demonstration
    @cortex.tool(name="lookup_account", description="Look up a customer account")
    async def lookup_account(account_id: str) -> str:
        """Look up a customer account by ID."""
        logger.info(f"Tool called: lookup_account(account_id={account_id})")
        return f"Account {account_id}: Active, Plan: Enterprise, Since: 2024-01"

    # 3. Create the agent with enterprise config
    agent = engine.create_agent(
        name="production_support",
        system_prompt=(
            "You are a production customer support agent. "
            "Be professional, accurate, and concise. "
            "Never discuss competitor products, internal salaries, "
            "or unreleased features."
        ),
        tools=[lookup_account],
        goal_tracking=True,
        enterprise_config=enterprise,
    )

    startup_ms = (time.time() - start_time) * 1000
    logger.info(f"Agent initialized in {startup_ms:.0f}ms")
```

1. Use `os.environ["KEY"]` (not `.get()`) in production. If the key is missing, the application should fail loudly at startup rather than silently later.

---

## Step 7: Add session monitoring

Create a helper that logs detailed metrics after each interaction. In production, you would send these metrics to your monitoring system (Prometheus, Datadog, custom dashboard).

```python title="production.py"
    async def monitored_run(session, message: str, request_id: str):
        """Run a message with full monitoring and logging."""
        logger.info(f"[{request_id}] User message: {message[:80]}...")
        run_start = time.time()

        response = await session.run(message)

        run_ms = (time.time() - run_start) * 1000
        meta = response.metadata

        # Log performance metrics
        logger.info(
            f"[{request_id}] Completed | "
            f"model={meta.model_used} | "
            f"tokens={meta.tokens_used} | "
            f"latency={meta.latency_ms:.0f}ms | "
            f"total_time={run_ms:.0f}ms | "
            f"tools={meta.tools_called} | "
            f"goal_progress={meta.goal_progress}"
        )

        return response
```

---

## Step 8: Run monitored interactions

```python title="production.py"
    # Start a session
    session = agent.start_session(user_id="enterprise_customer_1")
    logger.info("Session started for enterprise_customer_1")

    # Run interactions with monitoring
    r1 = await monitored_run(
        session,
        "Can you look up account ACC-5001?",
        request_id="req-001",
    )
    print(f"Agent: {r1.content}")

    r2 = await monitored_run(
        session,
        "What plan are they on and when did they join?",
        request_id="req-002",
    )
    print(f"Agent: {r2.content}")

    # Test safety: blocked topic
    r3 = await monitored_run(
        session,
        "How does our product compare to the competitor's offering?",
        request_id="req-003",
    )
    print(f"Agent: {r3.content}")
    logger.info(f"[req-003] Safety test: blocked topic should be refused")
```

Expected log output:

```text
2026-02-10 14:23:01 [INFO] cortex.production: Starting production agent...
2026-02-10 14:23:01 [INFO] cortex.production: Agent initialized in 127ms
2026-02-10 14:23:01 [INFO] cortex.production: Session started for enterprise_customer_1
2026-02-10 14:23:01 [INFO] cortex.production: [req-001] User message: Can you look up account ACC-5001?...
2026-02-10 14:23:01 [INFO] cortex.production: Tool called: lookup_account(account_id=ACC-5001)
2026-02-10 14:23:02 [INFO] cortex.production: [req-001] Completed | model=gpt-4o-mini | tokens=215 | latency=847ms | total_time=892ms | tools=['lookup_account'] | goal_progress=0.4
2026-02-10 14:23:02 [INFO] cortex.production: [req-002] User message: What plan are they on and when did they join?...
2026-02-10 14:23:03 [INFO] cortex.production: [req-002] Completed | model=gpt-4o-mini | tokens=148 | latency=523ms | total_time=558ms | tools=[] | goal_progress=0.7
2026-02-10 14:23:03 [INFO] cortex.production: [req-003] User message: How does our product compare to the competitor's offering?...
2026-02-10 14:23:04 [INFO] cortex.production: [req-003] Completed | model=gpt-4o | tokens=186 | latency=1102ms | total_time=1145ms | tools=[] | goal_progress=0.7
2026-02-10 14:23:04 [INFO] cortex.production: [req-003] Safety test: blocked topic should be refused
```

---

## Step 9: Collect session statistics at close

```python title="production.py"
    # Collect final metrics
    print("\n--- Session Metrics ---")
    print(f"Goal progress:  {session.get_goal_progress()}")
    print(f"Dual-process:   {session.get_dual_process_stats()}")
    print(f"Tool reputation: {session.get_reputation_stats()}")
    print(f"Weights:        {session.get_weights()}")

    # Close the session
    stats = await session.close()
    logger.info(
        f"Session closed | turns={stats['turns']} | "
        f"total_tokens={stats['total_tokens']}"
    )
    print(f"\nSession: {stats['turns']} turns, {stats['total_tokens']} total tokens")
```

---

## Step 10: On-premises deployment considerations

corteX is designed on-prem first. For air-gapped or regulated environments, keep these points in mind.

!!! info "On-prem checklist"
    1. **Local models:** Replace cloud providers with local models (Ollama, vLLM) to keep all data on-premises.

        ```python
        engine = cortex.Engine(
            providers={
                "local": {"base_url": "http://localhost:11434/v1"},
            },
            orchestrator_model="llama-3-70b",
            worker_model="llama-3-8b",
        )
        ```

    2. **File-based persistence:** Weights, memory, and audit logs can all use local file storage. No database server required.

    3. **No network dependencies:** The core SDK (weight engine, memory fabric, all brain components) runs entirely in-process with zero network calls.

    4. **License validation:** Uses Ed25519 signatures verified locally. A 30-day grace period ensures continuity during license renewal.

---

## Step 11: Performance tuning

For high-throughput production deployments, consider these tuning strategies.

```python title="performance_tips.py"
# 1. Use the worker model for most traffic
# The System 1/2 split naturally routes ~80% to the cheaper worker model.
# Ensure your worker model is fast: gemini-2.0-flash, gpt-4o-mini, or a local 8B model.

# 2. Tune the context budget for your model
context_config = cortex.ContextManagementConfig(
    token_budget_ratio=0.80,  # Leave 20% headroom for response generation
)

# 3. Configure weight tuning for efficiency
weights = cortex.WeightConfig(
    speed_vs_quality=0.3,  # Lean toward speed in high-throughput scenarios
    verbosity=-0.4,        # Shorter responses = fewer tokens = faster
)

# 4. Monitor the System 2 ratio
# If system2_ratio > 0.30, investigate why so many requests need the orchestrator.
# Common causes: vague system prompts, missing tools, overly broad user inputs.

# 5. Use streaming for chat UIs
# session.run_stream() returns tokens as they generate,
# reducing perceived latency for end users.
```

!!! tip "Token budget optimization"
    The single highest-impact performance optimization is reducing unnecessary token usage. Set `token_budget_ratio` to 0.75-0.85, use moderate `verbosity` weights (-0.2 to -0.4), and ensure your tools return concise results. A well-tuned agent can handle 2-3x more throughput than a verbose one on the same infrastructure.

---

## Step 12: Close and run

```python title="production.py"

if __name__ == "__main__":
    asyncio.run(main())
```

```bash
export OPENAI_API_KEY="sk-..."
python production.py
```

---

## Complete code

```python title="production.py"
import asyncio
import logging
import os
import time
import cortex

try:
    from dotenv import load_dotenv
    load_dotenv()
except ImportError:
    pass

log_level = os.environ.get("CORTEX_LOG_LEVEL", "INFO")
logging.basicConfig(
    level=getattr(logging, log_level),
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger("cortex.production")

safety_level = os.environ.get("CORTEX_SAFETY_LEVEL", "strict")
enterprise = cortex.EnterpriseConfig(
    safety_level=safety_level,
    blocked_topics=["competitor_info", "internal_salary", "unreleased_products", "legal_advice"],
    audit_log=True,
    pii_detection=True,
    max_autonomy=0.5,
    require_human_approval=["delete_account"],
)


async def main():
    logger.info("Starting production agent...")
    start_time = time.time()

    engine = cortex.Engine(
        providers={"openai": {"api_key": os.environ["OPENAI_API_KEY"]}},
        orchestrator_model="gpt-4o",
        worker_model="gpt-4o-mini",
    )

    @cortex.tool(name="lookup_account", description="Look up a customer account")
    async def lookup_account(account_id: str) -> str:
        logger.info(f"Tool called: lookup_account(account_id={account_id})")
        return f"Account {account_id}: Active, Plan: Enterprise, Since: 2024-01"

    agent = engine.create_agent(
        name="production_support",
        system_prompt=(
            "You are a production customer support agent. "
            "Be professional, accurate, and concise. "
            "Never discuss competitor products, internal salaries, "
            "or unreleased features."
        ),
        tools=[lookup_account],
        goal_tracking=True,
        enterprise_config=enterprise,
    )

    startup_ms = (time.time() - start_time) * 1000
    logger.info(f"Agent initialized in {startup_ms:.0f}ms")

    async def monitored_run(session, message: str, request_id: str):
        logger.info(f"[{request_id}] User message: {message[:80]}...")
        run_start = time.time()
        response = await session.run(message)
        run_ms = (time.time() - run_start) * 1000
        meta = response.metadata
        logger.info(
            f"[{request_id}] Completed | model={meta.model_used} | "
            f"tokens={meta.tokens_used} | latency={meta.latency_ms:.0f}ms | "
            f"total={run_ms:.0f}ms | tools={meta.tools_called} | "
            f"goal={meta.goal_progress}"
        )
        return response

    session = agent.start_session(user_id="enterprise_customer_1")
    logger.info("Session started")

    r1 = await monitored_run(session, "Look up account ACC-5001.", "req-001")
    print(f"Agent: {r1.content}")

    r2 = await monitored_run(session, "What plan and join date?", "req-002")
    print(f"Agent: {r2.content}")

    r3 = await monitored_run(
        session, "How do we compare to the competitor?", "req-003"
    )
    print(f"Agent: {r3.content}")

    print("\n--- Session Metrics ---")
    print(f"Goal progress:   {session.get_goal_progress()}")
    print(f"Dual-process:    {session.get_dual_process_stats()}")
    print(f"Tool reputation: {session.get_reputation_stats()}")
    print(f"Weights:         {session.get_weights()}")

    stats = await session.close()
    logger.info(f"Session closed | turns={stats['turns']} | tokens={stats['total_tokens']}")
    print(f"\nSession: {stats['turns']} turns, {stats['total_tokens']} total tokens")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## What you learned

- How to configure `cortex.EnterpriseConfig` with safety levels, blocked topics, PII detection, and autonomy caps
- How to manage API keys and configuration through environment variables
- How to set up structured logging that integrates with production log aggregation
- How to build a monitoring wrapper that tracks performance metrics per request
- On-premises deployment considerations for air-gapped and regulated environments
- Performance tuning strategies: token budgets, System 1/2 ratio monitoring, and verbosity control

## Next steps

- [Enterprise Safety](../enterprise/safety.md) -- deep dive into safety policies, PII detection, and autonomy capping
- [Audit Logging](../enterprise/audit.md) -- detailed audit configuration including syslog and webhook destinations
- [On-Premises Deployment](../enterprise/on-prem.md) -- complete air-gapped deployment guide
- [Observability guide](../guides/advanced/observability.md) -- advanced monitoring with custom metrics
