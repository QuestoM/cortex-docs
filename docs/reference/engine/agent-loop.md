# Agent Loop API Reference

## Module: `corteX.engine.agent_loop`

Thin orchestrator driving multi-step task execution. Yields `LoopAction` objects via Python's generator-send protocol. The caller (Session) handles LLM calls, tool execution, and user interaction. This module NEVER calls LLMs or external APIs directly.

Integrates Goal Intelligence (GoalDNA + GoalReminderInjector) for drift detection and goal reminder injection into every LLM call.

---

## Classes

### `LoopActionType`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `LLM_CALL` | Request an LLM generation |
| `TOOL_CALL` | Request a tool execution |
| `USER_INTERACTION` | Ask user a question |
| `COMPACTION` | Context needs compaction |
| `PLAN_GENERATION` | Generate or revise a plan |
| `REFLECTION` | Reflect on output quality |
| `IMPROVEMENT` | Improve a response |
| `COMPLETE` | Goal achieved -- loop finished |
| `ABORT` | Execution aborted due to errors |

### `LoopAction`

**Type**: `@dataclass`

A single action yielded by the loop for external execution.

| Attribute | Type | Description |
|-----------|------|-------------|
| `action_type` | `LoopActionType` | Type of action requested |
| `messages` | `List[Dict[str, str]]` | Messages for LLM/plan prompts |
| `params` | `Dict[str, Any]` | Brain parameters and config |
| `tool_name` | `Optional[str]` | Tool to execute (for TOOL_CALL) |
| `tool_args` | `Optional[Dict]` | Tool arguments |
| `interaction_request` | `Optional[InteractionRequest]` | For USER_INTERACTION |
| `reason` | `str` | Human-readable reason for this action |

### `LoopState`

**Type**: `@dataclass`

Snapshot of current loop state for persistence and monitoring.

| Attribute | Type | Description |
|-----------|------|-------------|
| `goal` | `str` | Current goal |
| `step_count` | `int` | Steps executed so far |
| `max_steps` | `int` | Maximum allowed steps |
| `plan` | `Optional[ExecutionPlan]` | Current execution plan |
| `current_step` | `Optional[PlanStep]` | Step being executed |
| `last_response` | `str` | Most recent LLM response |
| `status` | `str` | `"running"`, `"completed"`, `"aborted"`, `"idle"` |
| `drift_score` | `float` | Current GoalDNA drift score |
| `drift_events` | `int` | Total drift events detected |
| `is_drifting` | `bool` | Whether sustained drift is active |

---

### `AgentLoop`

Thin orchestrator via generator. Yields `LoopAction`, receives results via `send()`.

#### Constructor

```python
AgentLoop(
    context_compiler: ContextCompiler,
    planner: PlanningEngine,
    reflector: ReflectionEngine,
    recovery: RecoveryEngine,
    interaction: InteractionManager,
    policy: PolicyEngine,
    sub_agents: SubAgentManager,
    max_steps: int = 100,
    goal_dna: Optional[GoalDNA] = None,
    goal_reminder: Optional[GoalReminderInjector] = None,
)
```

All sub-engines are injected. The loop composes them into a coherent execution flow.

#### Methods

##### `start`

```python
def start(
    self, goal: str, system_prompt: str,
    brain_params: Optional[Dict[str, Any]] = None,
    tools: Optional[List[Dict[str, Any]]] = None,
    user_preferences: Optional[Dict[str, Any]] = None,
    policies: Optional[List[str]] = None,
) -> Generator[LoopAction, Any, LoopState]
```

Generator driving multi-step execution. The caller iterates with `send()`:

1. Optionally generates a plan (if goal is complex enough)
2. Iterates through plan steps, yielding LLM_CALL actions
3. Checks drift via GoalDNA after each step
4. Injects goal reminders into LLM context
5. Reflects on output quality when triggered
6. Handles errors via RecoveryEngine
7. Replans on sustained drift or policy violations
8. Yields COMPLETE or ABORT when finished

**Returns**: `Generator[LoopAction, Any, LoopState]`

##### `handle_error`

```python
def handle_error(
    self, error: Exception, step: Optional[PlanStep] = None,
    model_name: Optional[str] = None,
) -> LoopAction
```

Use recovery engine to determine next action after an error. Returns a `LoopAction` with the appropriate recovery strategy (retry, escalate, or abort).

##### `get_state`

```python
def get_state(self) -> LoopState
```

Get current loop state snapshot.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Aggregate stats from all sub-engines: context_compiler, planner, reflection, recovery, interaction, policy, sub_agents, goal_dna, goal_reminder.

---

## Execution Flow

```
start(goal) --> maybe_plan --> [loop]
                                 |
                           get_next_step
                                 |
                           compile_context
                                 |
                        inject_goal_reminder
                                 |
                         yield LLM_CALL -----> caller sends response
                                 |
                          check_drift (GoalDNA)
                                 |
                       evaluate_policy (PolicyEngine)
                                 |
                     maybe_reflect (ReflectionEngine)
                                 |
                       mark_step_completed
                                 |
                           [next step]
```

---

## See Also

- [Agentic Engine Architecture](../../concepts/engine-v2.md)
- [Context Compiler API](./context-compiler.md)
- [Planning Engine API](./planner.md)
- [Goal DNA API](./goal-dna.md)
