# Memory Management

corteX sessions accumulate conversation history as agents run. Without management, long-running agents can exhaust provider context windows or consume excessive RAM. The memory management system provides budget enforcement, automatic compaction, and multi-agent pool monitoring to keep sessions healthy at any scale.

---

## Overview

Three layers compose the memory management architecture:

| Layer | Class | Module | Responsibility |
|---|---|---|---|
| **1 - Budget** | `ContextBudgetEnforcer` | `corteX.engine.context_budget` | Tracks message count + token estimate per session; signals when compaction is needed |
| **2 - Compaction** | `ConversationCompactor` | `corteX.memory.compactor` | Compresses conversation history using one of three strategies when budget is exceeded |
| **3 - Pool** | `AgentPoolMemoryMonitor` | `corteX.runtime.memory_monitor` | Monitors process-level RAM across all agents and recommends which to suspend under pressure |

The layers are independent by design. You can use the enforcer alone (to detect when to compact), wire it together with the compactor (for automatic reduction), and optionally add the pool monitor when running many agents in one process.

The `MemoryManagementMixin` in `corteX.session.memory_mixin` wires layers 1 and 2 directly into a Session subclass so they activate automatically before each turn.

```
User turn
    |
    v
[ContextBudgetEnforcer]  ---> BudgetStatus (OK / WARNING / CRITICAL / EXCEEDED)
    |
    | (if WARNING or above)
    v
[ConversationCompactor]  ---> compacted message list + CompactionResult
    |
    v
[LLM generation continues with compacted history]

(In parallel, from the orchestrator)
[AgentPoolMemoryMonitor] ---> MemoryPressure (OK / WARNING / CRITICAL)
    |
    | (if WARNING or above)
    v
recommend_suspensions() ---> list of agent IDs to suspend
```

---

## ContextBudgetEnforcer

`corteX.engine.context_budget`

The enforcer tracks two dimensions simultaneously: message count and estimated token count. It takes the maximum of both percentages to compute `pct_used`, so either dimension alone can trigger a status escalation.

Token estimation uses a word-count heuristic (`words * 1.3`) rather than a real tokenizer. This keeps the module dependency-free and fully on-prem compatible while erring on the conservative side.

### BudgetStatus

| Value | Threshold | Meaning |
|---|---|---|
| `OK` | `pct_used < 0.80` | No action needed |
| `WARNING` | `>= 0.80` | Compaction recommended |
| `CRITICAL` | `>= 0.95` | Compaction strongly recommended |
| `EXCEEDED` | `>= 1.00` | Context limit reached; compaction required immediately |

### Configuration

| Parameter | Default | Description |
|---|---|---|
| `max_messages` | `200` | Maximum message count before signalling compaction |
| `max_tokens` | `100_000` | Estimated token ceiling for the conversation history |
| `warning_pct` | `0.80` | Fraction of the limit that triggers `WARNING` |
| `critical_pct` | `0.95` | Fraction of the limit that triggers `CRITICAL` |

### Basic usage

```python
from corteX.engine.context_budget import get_budget_enforcer, BudgetStatus

enforcer = get_budget_enforcer()

snapshot = enforcer.check_budget(messages, session_id="my-session")
print(snapshot.status)       # BudgetStatus.OK
print(snapshot.pct_used)     # 0.23
print(snapshot.message_count)
print(snapshot.estimated_tokens)

if enforcer.should_compact(messages):
    # trigger compaction
    pass
```

### Enforcement with logging

`enforce()` wraps `check_budget()` and automatically emits `WARNING`-level log entries for WARNING/CRITICAL and `ERROR`-level log entries for EXCEEDED:

```python
snapshot, needs_compaction = enforcer.enforce(messages, session_id="my-session")

if needs_compaction:
    # compact and continue
    pass
```

### Custom limits

The singleton returned by `get_budget_enforcer()` uses default limits. For custom limits, instantiate directly:

```python
from corteX.engine.context_budget import ContextBudgetEnforcer

enforcer = ContextBudgetEnforcer(
    max_messages=50,
    max_tokens=32_000,
    warning_pct=0.75,
    critical_pct=0.90,
)
```

---

## ConversationCompactor

`corteX.memory.compactor`

The compactor reduces conversation history using one of three strategies. Choose the strategy based on how much continuity your agent needs.

### Strategies

| Strategy | Behavior | Best for |
|---|---|---|
| `SUMMARIZE` (default) | Replaces old messages with a `[HISTORY SUMMARY: N messages compacted]` marker, keeps the most recent N messages verbatim | Long-running agents that need continuity (C-suite, research agents) |
| `SLIDING_WINDOW` | Discards everything except the last N messages | Stateless agents where each turn is mostly independent (support bots, FAQ agents) |
| `DROP_OLDEST` | Drops oldest messages until the list fits within `max_messages` | High-throughput agents where throughput matters more than continuity |

### Configuration

| Parameter | Default | Description |
|---|---|---|
| `strategy` | `SUMMARIZE` | Compaction strategy to use |
| `max_messages` | `100` | Maximum messages to keep after compaction |
| `keep_recent` | `20` | Number of recent messages to preserve verbatim (SUMMARIZE only) |
| `summary_role` | `"system"` | Role assigned to the inserted summary marker (SUMMARIZE only) |

### Basic usage

```python
from corteX.memory.compactor import (
    ConversationCompactor,
    CompactionConfig,
    CompactionStrategy,
)

compactor = ConversationCompactor(CompactionConfig(
    strategy=CompactionStrategy.SUMMARIZE,
    max_messages=100,
    keep_recent=20,
))

compacted, result = compactor.compact(messages, session_id="my-session")
print(f"Compressed {result.messages_removed} messages")
# result.original_count, result.compacted_count, result.strategy_used
```

If the message list is already within budget, `compact()` returns the original list unchanged and a `CompactionResult` with `messages_removed == 0`.

### Using the default singleton

For zero-config usage, the module exposes a singleton:

```python
from corteX.memory.compactor import get_compactor

compacted, result = get_compactor().compact(messages)
```

---

## MemoryManagementMixin

`corteX.session.memory_mixin`

The mixin wires the budget enforcer and compactor directly into a Session subclass so you do not need to call them manually on every turn.

### Adding the mixin to a Session

```python
from corteX.session.memory_mixin import MemoryManagementMixin
from corteX.engine.context_budget import ContextBudgetEnforcer
from corteX.memory.compactor import ConversationCompactor, CompactionConfig, CompactionStrategy


class MySession(MemoryManagementMixin):
    def __init__(self, session_id: str) -> None:
        self.session_id = session_id
        self.messages: list[dict] = []

        # Wire in default enforcer + compactor
        self.init_memory_management()

        # Or wire in custom instances:
        # self.init_memory_management(
        #     enforcer=ContextBudgetEnforcer(max_messages=50, max_tokens=32_000),
        #     compactor=ConversationCompactor(CompactionConfig(
        #         strategy=CompactionStrategy.SLIDING_WINDOW,
        #     )),
        # )

    async def run(self, message: str) -> str:
        self.messages.append({"role": "user", "content": message})

        # Compact before sending to the LLM
        self.messages, result = self.compact_if_needed(
            self.messages, session_id=self.session_id
        )
        if result:
            print(f"Compacted: {result.messages_removed} messages removed")

        # ... send self.messages to LLM ...
        return "response"
```

### Checking memory status

`get_memory_status()` returns a plain dict suitable for health checks and monitoring dashboards:

```python
status = session.get_memory_status(session.messages, session_id="my-session")
# {
#   "session_id": "my-session",
#   "message_count": 47,
#   "estimated_tokens": 8312,
#   "pct_used": 0.234,
#   "status": "ok",
#   "needs_compaction": False,
# }
```

### API summary

| Method | Returns | Description |
|---|---|---|
| `init_memory_management(enforcer, compactor)` | `None` | Wire in enforcer and compactor; call from `__init__` |
| `check_memory_budget(messages, session_id)` | `BudgetSnapshot` | Check budget without modifying messages |
| `compact_if_needed(messages, session_id)` | `(list[dict], CompactionResult \| None)` | Compact only if budget signals compaction is needed |
| `get_memory_status(messages, session_id)` | `dict` | Health-check-friendly summary of memory state |

---

## AgentPoolMemoryMonitor

`corteX.runtime.memory_monitor`

For multi-agent orchestrators running several agents in a single process, the pool monitor tracks process-level RAM and recommends which agents to suspend when memory pressure is detected.

!!! note "psutil dependency"
    The monitor uses `psutil` to read the current process RSS. If `psutil` is not installed, RSS readings fall back to `0.0` and `MemoryPressure` stays at `OK`. Install it with `pip install psutil` to enable real monitoring.

### MemoryPressure

| Value | Threshold | Meaning |
|---|---|---|
| `OK` | `pct_used < 0.70` | No action needed |
| `WARNING` | `>= 0.70` | Consider suspending low-priority agents |
| `CRITICAL` | `>= 0.90` | Suspend agents immediately |

### Configuration

| Parameter | Default | Description |
|---|---|---|
| `max_mb` | `2048.0` | Memory ceiling in megabytes (2 GB) |
| `warning_pct` | `0.70` | Fraction of `max_mb` that triggers `WARNING` |
| `critical_pct` | `0.90` | Fraction of `max_mb` that triggers `CRITICAL` |
| `min_priority_to_keep` | `5` | Agents at or above this priority are never recommended for suspension |

### Agent priority

When registering an agent, assign a priority from 1 (lowest) to 5 (highest). The monitor sorts suspension candidates by priority (lowest first) and then by registration order (oldest first) when priorities are equal.

| Priority | Meaning |
|---|---|
| 1 | Expendable workers - suspended first |
| 2-4 | Mid-tier agents |
| 5 | Critical agents (C-suite equivalent) - never suspended by default |

### Basic usage

```python
from corteX.runtime.memory_monitor import get_memory_monitor, MemoryPressure

monitor = get_memory_monitor()

monitor.register_agent("agent-001", priority=1)
monitor.register_agent("agent-002", priority=2)
monitor.register_agent("ceo-agent", priority=5)

snapshot = monitor.take_snapshot()
print(snapshot.rss_mb)       # current process RSS in MB
print(snapshot.pct_used)     # fraction of max_mb used
print(snapshot.pressure)     # MemoryPressure.OK / WARNING / CRITICAL
print(snapshot.agent_count)  # number of active (non-suspended) agents

if snapshot.pressure != MemoryPressure.OK:
    to_suspend = monitor.recommend_suspensions(target_count=3)
    for agent_id in to_suspend:
        # suspend the agent in your orchestrator
        monitor.mark_suspended(agent_id)
```

### Resuming agents

When memory pressure drops, mark agents active again:

```python
snapshot = monitor.take_snapshot()
if snapshot.pressure == MemoryPressure.OK:
    monitor.mark_active("agent-001")
```

### Custom instance

```python
from corteX.runtime.memory_monitor import AgentPoolMemoryMonitor

monitor = AgentPoolMemoryMonitor(
    max_mb=4096.0,
    warning_pct=0.65,
    critical_pct=0.85,
    min_priority_to_keep=4,
)
```

---

## Best Practices

!!! tip "Set budgets based on your LLM tier"
    For Gemini Tier-1 (25 RPM, 250 RPD), keep sessions compact. A `max_messages=50` / `max_tokens=32_000` budget ensures each session stays well within the model's context window and leaves headroom for tool calls.

!!! tip "Match compaction strategy to agent type"
    - **C-suite agents** (CEO, CFO, CISO) need continuity across many turns. Use `SUMMARIZE` so they retain a high-level summary of earlier context.
    - **Support or FAQ agents** are stateless by nature. Use `SLIDING_WINDOW` and keep only the last 10-20 messages.
    - **High-throughput pipeline agents** processing many short tasks should use `DROP_OLDEST` for the lowest overhead.

!!! tip "Register all agents with explicit priority"
    Always pass a `priority` when calling `monitor.register_agent()`. Unregistered or default-priority agents create ambiguity in suspension ordering. Assigning deterministic priorities makes orchestrator behaviour predictable under pressure.

!!! tip "Emit memory status to your observability platform"
    Call `session.get_memory_status()` after each turn and ship the dict to your metrics backend (Datadog, Prometheus, OpenTelemetry). Alert when `status` reaches `critical` or `needs_compaction` is `True` for multiple consecutive turns.

---

## Next steps

- [Monitor Your Agent](observability.md) - full session stats and metrics export.
- [Context Summarization](../../concepts/advanced/context-summarizer.md) - the context summarizer that powers SUMMARIZE compaction.
- [Memory Fabric](../../concepts/context/memory.md) - working, episodic, and semantic memory across sessions.
