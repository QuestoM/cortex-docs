# Planning Engine API Reference

## Module: `corteX.engine.planner`

Goal decomposition and multi-step execution planning. Decomposes goals into executable plans, tracks execution state, and supports replanning on failure. Builds prompts and parses responses only -- never calls LLM directly.

Brain analogy: Prefrontal cortex (decomposition), basal ganglia (sequencing), anterior cingulate (replanning), motor cortex (dependency scheduling).

---

## Classes

### `PlanStepStatus`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `PENDING` | Not yet started |
| `RUNNING` | Currently executing |
| `COMPLETED` | Successfully completed |
| `FAILED` | Failed with error |
| `SKIPPED` | Intentionally skipped |

### `PlanStep`

**Type**: `@dataclass`

A single executable step within an execution plan.

| Attribute | Type | Description |
|-----------|------|-------------|
| `step_id` | `str` | Unique step identifier |
| `description` | `str` | What this step does |
| `expected_outcome` | `str` | Expected result |
| `tools_needed` | `List[str]` | Tools required for this step |
| `dependencies` | `List[str]` | Step IDs that must complete first |
| `status` | `PlanStepStatus` | Current status |
| `retry_count` | `int` | How many times this step has been retried |
| `max_retries` | `int` | Maximum retry attempts (default: 2) |
| `result` | `Optional[str]` | Execution result |
| `error` | `Optional[str]` | Error message if failed |

### `ExecutionPlan`

**Type**: `@dataclass`

A full execution plan decomposed from a goal.

| Attribute | Type | Description |
|-----------|------|-------------|
| `plan_id` | `str` | Unique plan identifier |
| `goal` | `str` | Original goal text |
| `steps` | `List[PlanStep]` | Ordered list of steps |
| `created_at` | `float` | Creation timestamp |
| `revised_count` | `int` | Number of times plan was revised |

**Properties**: `current_step_index`, `is_complete`, `progress` (0.0-1.0)

**Methods**: `get_completed_steps()`, `get_failed_steps()`, `to_summary()`

---

### `PlanningEngine`

Decomposes goals into executable plans using LLM prompt/parse pattern.

#### Constructor

```python
PlanningEngine(max_steps: int = 20, min_steps: int = 1)
```

**Parameters**:

- `max_steps` (int): Maximum steps in a plan. Default: 20
- `min_steps` (int): Minimum steps in a plan. Default: 1

#### Methods

##### `should_plan`

```python
def should_plan(self, goal: str, threshold: float = 0.3) -> bool
```

Decide if this goal warrants planning vs single-step execution. Uses `estimate_complexity()` against the threshold.

##### `estimate_complexity`

```python
def estimate_complexity(self, goal: str) -> float
```

Heuristic complexity estimate (0.0-1.0) based on goal text. Factors: length (25%), action verbs (30%), multi-action keywords (30%), structure (15%).

##### `build_plan_prompt`

```python
def build_plan_prompt(self, goal: str, available_tools: List[str], context: str = "") -> str
```

Build the prompt for LLM to generate a plan. Returns prompt text requesting JSON array output.

##### `parse_plan`

```python
def parse_plan(self, goal: str, llm_response: str) -> ExecutionPlan
```

Parse LLM response into an `ExecutionPlan`. Tries JSON parsing first, then numbered/bulleted text, then single-step fallback.

##### `build_replan_prompt` / `parse_replan`

```python
def build_replan_prompt(self, plan: ExecutionPlan, failure_reason: str, context: str = "") -> str
def parse_replan(self, original_plan: ExecutionPlan, llm_response: str) -> ExecutionPlan
```

Build prompt for replanning after failure, preserving completed steps.

##### `get_next_step`

```python
def get_next_step(self, plan: ExecutionPlan) -> Optional[PlanStep]
```

Get next executable step respecting dependencies. Returns `None` if all steps are complete.

##### `mark_step_running` / `mark_step_completed` / `mark_step_failed`

State transition methods for plan steps.

##### `create_single_step_plan`

```python
def create_single_step_plan(self, goal: str) -> ExecutionPlan
```

Fallback: create a simple one-step plan for non-complex goals.

---

## Usage Example

```python
from corteX.engine.planner import PlanningEngine

planner = PlanningEngine(max_steps=15)

if planner.should_plan("Build a REST API with auth and testing"):
    prompt = planner.build_plan_prompt(
        "Build a REST API with auth and testing",
        available_tools=["code_writer", "test_runner"]
    )
    llm_response = await llm.generate(prompt)
    plan = planner.parse_plan("Build a REST API", llm_response)

    while step := planner.get_next_step(plan):
        planner.mark_step_running(plan, step.step_id)
        result = await execute_step(step)
        planner.mark_step_completed(plan, step.step_id, result)

    print(f"Plan complete: {plan.progress:.0%}")
```

---

## See Also

- [Goal Tracking Concept](../../concepts/brain/goal-tracking.md)
- [Agent Loop API](./agent-loop.md)
