# Agentic Engine Architecture

The corteX Agentic Engine transforms the SDK from a reactive single-turn chat wrapper into a **goal-driven multi-step agent loop**. Built on patterns from Claude Code, OpenClaw (IBM), CUGA, Cursor, and Manus, the Agentic Engine introduces planning, reflection, recovery, policy enforcement, and sub-agent delegation -- all while keeping the existing 20-component brain engine intact.

---

## Why the Agentic Engine?

The base engine processes one message at a time: the user sends a message, the 14-step pipeline runs once, and a response comes back. This works for conversational use cases, but enterprise workflows demand multi-step execution where the agent decomposes a goal, executes steps autonomously, reflects on quality, and recovers from errors -- all without manual intervention.

The Agentic Engine adds an **agentic loop** on top of the existing pipeline. The `session.run()` API remains unchanged. The `session.run_agentic()` API drives multi-step execution.

---

## Architecture

The agentic loop is a **thin orchestrator**. It yields actions to the caller, which handles LLM calls and tool execution. This separation keeps the loop testable and provider-agnostic.

```text
                    +---------------------------+
                    |      run_agentic()        |
                    |   (goal-driven loop)      |
                    +-------------+-------------+
                                  |
                    +-------------v-------------+
                    |     Context Compiler      |
                    |   (4-zone architecture)   |
                    +-------------+-------------+
                                  |
              +-------------------+-------------------+
              |                   |                   |
     +--------v-------+  +-------v--------+  +-------v--------+
     | Planning Engine |  | Policy Engine  |  | Recovery Engine|
     | (decompose goal)|  | (guardrails)   |  | (error handling)|
     +--------+-------+  +-------+--------+  +-------+--------+
              |                   |                   |
              +-------------------+-------------------+
                                  |
                    +-------------v-------------+
                    |     LLM Generation        |
                    |  (via existing pipeline)   |
                    +-------------+-------------+
                                  |
                    +-------------v-------------+
                    |   Reflection Engine        |
                    |  (post-generation QA)      |
                    +-------------+-------------+
                                  |
                    +-------------v-------------+
                    |   Interaction Manager      |
                    |  (autonomy levels L1-L5)   |
                    +---------------------------+
```

---

## Core Modules

### Context Compiler (4-Zone Architecture)

The Context Compiler assembles the LLM prompt from four zones, each with a target budget as a percentage of the context window:

| Zone | Budget | Contents | Caching Strategy |
|------|--------|----------|------------------|
| **System Zone** | ~12% | Agent identity, policies, tool definitions, user preferences | Stable across turns -- KV-cache friendly |
| **Persistent Zone** | ~8% | Current goal, brain state digest, plan summary | Append-only within a session |
| **Working Zone** | ~40% | Compressed conversation history, scratchpad notes | Summarized when budget is exceeded |
| **Recent Zone** | ~40% | Latest messages, current task state, goal restatement | Rotated every turn |

!!! note "Brain Science: Working Memory Zones"
    The 4-zone architecture mirrors how the brain organizes working memory. The prefrontal cortex maintains stable goal representations (Persistent Zone) while sensory cortices hold rapidly-changing perceptual information (Recent Zone). The hippocampus compresses and consolidates older information (Working Zone), and long-term procedural memory provides stable defaults (System Zone).

The zone budgets are configurable. When the total token count exceeds the context window, the Context Compiler compresses the Working Zone first (using the existing `ContextSummarizer`), then trims the Recent Zone from the oldest messages inward.

### Planning Engine

The Planning Engine decomposes a high-level goal into executable steps with dependency tracking.

**When planning activates:**

- The user provides a goal via `run_agentic()`
- Complexity estimation exceeds the `planning_threshold` (configurable)
- The Reflection Engine requests a replan after detecting quality issues
- Goal drift exceeds the `drift_threshold` from the Goal Tracker

**Plan structure:**

```python
Plan(
    goal="Analyze customer complaints and create a summary report",
    steps=[
        Step(id="1", action="Fetch complaints from database", deps=[]),
        Step(id="2", action="Categorize complaints by type", deps=["1"]),
        Step(id="3", action="Calculate severity distribution", deps=["1"]),
        Step(id="4", action="Generate summary report", deps=["2", "3"]),
    ],
    estimated_steps=4,
    complexity_score=0.65,
)
```

Plans are stored in the Persistent Zone and updated as steps complete. The Planning Engine respects dependency ordering -- step 4 above will not execute until steps 2 and 3 are both complete.

### Reflection Engine

The Reflection Engine runs **after** LLM generation to verify output quality. It catches hallucinations, incomplete answers, policy violations, and goal drift before the response reaches the user.

**6 trigger types:**

| Trigger | When It Fires | Action |
|---------|---------------|--------|
| **Always** | Every step | Baseline quality check |
| **Tool failure** | A tool call returned an error | Verify the agent handled the error correctly |
| **High surprise** | PredictionEngine surprise > threshold | Check if the unexpected result is valid |
| **Goal drift** | GoalTracker drift > threshold | Verify step still serves the original goal |
| **Long sequence** | Step count > configurable limit | Check for loops and wasted work |
| **User-facing** | Step produces a final response | Extra scrutiny on tone, completeness, accuracy |

**Lesson bank:** The Reflection Engine maintains a bank of lessons learned from past mistakes. When a reflection catches an error, the correction is stored and injected into future prompts as a "do not repeat" directive. Lessons decay over time if they are not triggered again.

### Recovery Engine

The Recovery Engine classifies errors and applies appropriate retry strategies.

**Error classification:**

| Class | Examples | Strategy |
|-------|----------|----------|
| **Transient** | Rate limit, timeout, network error | Exponential backoff with jitter, up to 3 retries |
| **Permanent** | Invalid API key, missing permission | Abort immediately, surface to user |
| **Context** | Context window exceeded, malformed tool output | Compress context or reformulate the request |
| **Fatal** | Provider down, budget exhausted | Abort the entire agentic run |

The Recovery Engine tracks cumulative errors across steps. If the error rate exceeds the `abort_threshold` (default: 5 errors in 10 steps), the agentic loop terminates with a detailed error report.

### Interaction Manager

The Interaction Manager controls how much autonomy the agent has during agentic execution.

**5 autonomy levels:**

| Level | Name | Behavior |
|-------|------|----------|
| **L1** | Full confirmation | Ask user before every action |
| **L2** | Confirm destructive | Ask before destructive or expensive actions |
| **L3** | Inform and proceed | Notify user but continue unless stopped |
| **L4** | Silent execution | Execute silently, report at the end |
| **L5** | Full autonomy | No interaction, no reporting mid-run |

**Smart timeouts:** If the user does not respond within the configured timeout (default: 30 seconds), the Interaction Manager can auto-decide based on the action's risk level. Low-risk actions proceed; high-risk actions are skipped with a note in the plan.

### Policy Engine

The Policy Engine enforces enterprise guardrails throughout the agentic loop. Policies are evaluated before tool execution, after LLM generation, and before user-facing responses.

**5 policy types:**

| Policy | Purpose | Example |
|--------|---------|---------|
| **IntentGuard** | Block disallowed intents | Reject requests to delete production data |
| **Playbook** | Enforce workflow templates | Customer complaints must follow the 5-step escalation process |
| **ToolApproval** | Gate tool access | `database_write` requires L2+ approval |
| **ToolGuide** | Shape tool usage | Always use parameterized queries for SQL tools |
| **OutputFormatter** | Enforce output standards | All reports must include a summary section |

Policies are defined as configuration and loaded at session creation. They integrate with the existing enterprise Safety Controls system.

### Sub-Agent Manager

The Sub-Agent Manager delegates work to isolated sub-agents, each with its own context window and tool set.

**When sub-agents are used:**

- A plan step requires a specialized tool set (e.g., code analysis vs. database queries)
- The main context window is approaching its limit
- Parallel execution would speed up independent plan steps

**Token budget allocation:** The Sub-Agent Manager divides the total token budget across active sub-agents. Each sub-agent receives a proportional share based on the estimated complexity of its assigned step. Results are summarized and injected back into the main agent's Working Zone.

---

## Usage

### Single-turn (existing v1 API, unchanged)

```python
import cortex_ai as cortex

engine = cortex.Engine(provider="gemini", api_key="...")
agent = cortex.Agent(engine=engine, system_prompt="You are a helpful assistant")
session = agent.session()

result = await session.run("What is the capital of France?")
print(result.text)
```

### Multi-step goal-driven (new in v2)

```python
import cortex_ai as cortex

engine = cortex.Engine(provider="gemini", api_key="...")
agent = cortex.Agent(
    engine=engine,
    system_prompt="You are a data analyst with access to the complaints database",
    tools=[fetch_complaints, categorize, generate_report],
)
session = agent.session()

result = await session.run_agentic(
    goal="Analyze customer complaints and create a summary report",
    max_steps=50,
    autonomy_level=3,  # L3: inform and proceed
)

print(result["final_response"])
print(result["plan_summary"])
print(result["steps_executed"])
print(result["tokens_used"])
```

### Configuring the agentic loop

```python
from cortex_ai.config import AgenticConfig

config = AgenticConfig(
    max_steps=100,
    autonomy_level=3,
    planning_threshold=0.3,      # complexity score that triggers planning
    drift_threshold=0.4,         # goal drift that triggers replanning
    abort_threshold=5,           # max errors before abort
    reflection_enabled=True,
    sub_agents_enabled=True,
    context_zones={
        "system": 0.12,
        "persistent": 0.08,
        "working": 0.40,
        "recent": 0.40,
    },
)

result = await session.run_agentic(
    goal="...",
    config=config,
)
```

---

## How v2 Integrates with the Brain Engine

The Agentic Engine does not replace the 20 brain components -- it builds on top of them. Every step in the agentic loop runs through the existing 14-step pipeline:

| Brain Component | Role in v2 |
|----------------|------------|
| **GoalTracker** | Tracks progress across all agentic steps, triggers replanning on drift |
| **PredictionEngine** | Predicts step outcomes, surprise signals trigger reflection |
| **DualProcessRouter** | Routes each step to System 1 or System 2 based on complexity |
| **WeightEngine** | Bayesian posteriors for tool and model selection across steps |
| **PlasticityManager** | Adjusts weights after each step, not just each turn |
| **FeedbackEngine** | Processes implicit signals from tool results and intermediate outputs |
| **ReputationSystem** | Quarantines tools that fail during agentic execution |
| **CorticalContextEngine** | Manages the 4-zone context window across all steps |
| **MemoryFabric** | Stores step results in working memory for future reference |
| **AdaptationFilter** | Habituates to repetitive step patterns, amplifies novel results |

---

## Design Principles

1. **Thin orchestrator**: The agentic loop yields actions; the caller executes them. This keeps the loop testable without live LLM calls.
2. **Provider-agnostic**: The Agentic Engine works with any LLM provider supported by corteX (OpenAI, Gemini, Anthropic, local models).
3. **Backward-compatible**: `session.run()` continues to work exactly as before. The Agentic Engine is opt-in via `session.run_agentic()`.
4. **Enterprise-first**: Policy enforcement, audit logging, and safety controls are built into every step of the agentic loop.
5. **On-prem ready**: No external service dependencies. The entire agentic loop runs locally.

---

## Learn More

- [Architecture (v1 pipeline)](architecture.md) -- the 14-step `run()` pipeline that v2 builds on.
- [Goal Tracking](brain/goal-tracking.md) -- how goal drift and loop prevention work.
- [Dual-Process Routing](brain/dual-process.md) -- System 1/2 routing used in each agentic step.
- [Context Engine](context/context-engine.md) -- the context management system extended by v2's 4-zone compiler.
