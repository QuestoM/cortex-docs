# Sub-Agent Manager API Reference

## Module: `corteX.engine.sub_agent`

Manages delegated work via sub-agents with isolated context windows and shared token budgets. Each sub-agent gets its own context space, preventing cross-contamination of focus.

Brain analogy: Prefrontal cortex delegates subtasks to specialized regions, each with its own working memory. Thalamus summarizes results back to parent. Does NOT call LLM directly -- builds prompts and parses responses.

---

## Classes

### `SubAgentTask`

**Type**: `@dataclass`

A single sub-agent task with its own context window.

| Attribute | Type | Description |
|-----------|------|-------------|
| `task_id` | `str` | Unique task identifier (format: `sub_<hex>`) |
| `goal` | `str` | Task goal description |
| `context_hint` | `str` | Context for the sub-agent |
| `max_steps` | `int` | Maximum steps allowed (default: 10) |
| `token_budget` | `int` | Token budget allocated (default: 10000) |
| `status` | `str` | `"pending"`, `"running"`, `"completed"`, `"failed"`, `"cancelled"` |
| `result` | `Optional[str]` | Execution result |
| `artifacts` | `List[Dict[str, Any]]` | Produced artifacts |
| `started_at` | `Optional[float]` | Start timestamp |
| `completed_at` | `Optional[float]` | Completion timestamp |
| `tokens_used` | `int` | Actual tokens consumed |
| `steps_taken` | `int` | Actual steps taken |

### `SubAgentResult`

**Type**: `@dataclass`

Result returned after sub-agent task completion.

| Attribute | Type | Description |
|-----------|------|-------------|
| `task_id` | `str` | Task identifier |
| `success` | `bool` | Whether task succeeded |
| `summary` | `str` | Result summary |
| `artifacts` | `List[Dict[str, Any]]` | Produced artifacts |
| `tokens_used` | `int` | Tokens consumed |
| `steps_taken` | `int` | Steps taken |
| `duration_ms` | `float` | Execution duration |

---

### `SubAgentManager`

Manages sub-agents with isolated context windows and shared token budget.

#### Constructor

```python
SubAgentManager(
    max_concurrent: int = 3,
    max_sub_agent_steps: int = 10,
    max_summary_tokens: int = 2000,
    total_token_budget: int = 50000,
)
```

**Parameters**:

- `max_concurrent` (int): Maximum sub-agents running at once. Default: 3
- `max_sub_agent_steps` (int): Default step limit per sub-agent. Default: 10
- `max_summary_tokens` (int): Max tokens for result summaries. Default: 2000
- `total_token_budget` (int): Shared budget across all sub-agents. Default: 50000

All parameters must be >= 1.

#### Methods

##### `create_task`

```python
def create_task(
    self, goal: str, context_hint: str = "",
    max_steps: Optional[int] = None, token_budget: Optional[int] = None,
) -> SubAgentTask
```

Create a new sub-agent task. Budget is allocated from the shared pool. Raises `ValueError` if goal is empty.

##### `build_sub_agent_prompt`

```python
def build_sub_agent_prompt(
    self, task: SubAgentTask, available_tools: Optional[List[str]] = None,
    parent_goal: str = "",
) -> str
```

Build the initial prompt for the sub-agent. Includes task goal, context, parent goal reference, available tools, step limit, and token budget.

##### `build_summary_prompt`

```python
def build_summary_prompt(self, task: SubAgentTask, full_output: str) -> str
```

Build a prompt to summarize sub-agent output to `max_summary_tokens`. Input is truncated to 20,000 characters.

##### `parse_summary`

```python
def parse_summary(self, llm_response: str) -> str
```

Parse and truncate summary to max_summary_tokens (chars as proxy, 4x multiplier).

##### `mark_running` / `mark_completed` / `mark_failed` / `cancel_task`

State transition methods for sub-agent tasks.

```python
def mark_running(self, task_id: str) -> None
def mark_completed(self, task_id: str, result: str, artifacts=None, tokens_used=0, steps_taken=0) -> SubAgentResult
def mark_failed(self, task_id: str, error: str) -> SubAgentResult
def cancel_task(self, task_id: str) -> None
```

##### `can_spawn`

```python
def can_spawn(self) -> bool
```

Check if another sub-agent can be spawned (concurrency limit + budget remaining).

##### `allocate_budget`

```python
def allocate_budget(self, requested: int) -> int
```

Allocate token budget from remaining pool. Returns actual allocation (may be less than requested).

##### `get_task` / `get_running_tasks` / `get_completed_results` / `get_all_tasks`

Task query methods.

##### `get_stats`

Returns: total_tasks, running, completed, failed, cancelled, pending, tokens_spent, tokens_remaining, max_concurrent, can_spawn.

---

## Usage Example

```python
from corteX.engine.sub_agent import SubAgentManager

manager = SubAgentManager(max_concurrent=3, total_token_budget=50000)

# Create and execute a sub-agent task
task = manager.create_task(
    goal="Write unit tests for the user model",
    context_hint="User model is in models/user.py with fields: id, name, email",
    max_steps=8,
)

prompt = manager.build_sub_agent_prompt(
    task,
    available_tools=["code_writer", "test_runner"],
    parent_goal="Build a REST API for user management",
)

manager.mark_running(task.task_id)

# ... LLM execution happens here ...

result = manager.mark_completed(
    task.task_id, result="Created 5 unit tests",
    tokens_used=3000, steps_taken=4,
)

# Summarize for parent context
summary_prompt = manager.build_summary_prompt(task, full_output)
summary = manager.parse_summary(llm_summary_response)

print(f"Can spawn more: {manager.can_spawn()}")
print(f"Stats: {manager.get_stats()}")
```

---

## See Also

- [Agent Loop API](./agent-loop.md)
- [Context Compiler API](./context-compiler.md)
