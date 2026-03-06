# Reflection Engine API Reference

## Module: `corteX.engine.reflection`

Post-generation quality verification with learning. Reviews agent outputs, identifies issues, triggers corrections, and stores learned lessons. Never calls LLM directly -- builds prompts and parses responses.

Brain analogy: Metacognition (prefrontal cortex), error monitoring (ACC), episodic memory (hippocampus recalls past failures to avoid repeating them).

---

## Classes

### `ReflectionTrigger`

**Type**: `str, Enum`

| Value | Condition |
|-------|-----------|
| `LOW_CONFIDENCE` | Confidence below quality threshold |
| `HIGH_RISK` | Risk level > 0.7 |
| `GOAL_DRIFT` | Goal drift > 0.4 |
| `TOOL_FAILURE` | Tool failures > 0 |
| `USER_CRITICAL` | Final step of plan |
| `PERIODIC` | Every N steps |

### `ReflectionResult`

**Type**: `@dataclass`

| Attribute | Type | Description |
|-----------|------|-------------|
| `triggered_by` | `ReflectionTrigger` | What triggered this reflection |
| `passes_quality_gate` | `bool` | Whether output passes quality threshold |
| `quality_score` | `float` | Quality score [0.0, 1.0] |
| `issues_found` | `List[str]` | Identified issues |
| `suggested_fix` | `Optional[str]` | How to fix issues |
| `should_retry` | `bool` | Whether to retry with improvements |
| `retry_guidance` | `str` | Guidance for retry |
| `reflection_time_ms` | `float` | Time taken for reflection in milliseconds (default: 0.0) |

### `ReflectionLesson`

**Type**: `@dataclass`

A learned lesson from past mistakes, with effectiveness tracking.

| Attribute | Type | Description |
|-----------|------|-------------|
| `lesson_id` | `str` | Unique identifier |
| `task_type` | `str` | Task category |
| `mistake` | `str` | What went wrong |
| `correction` | `str` | How to fix it |
| `effectiveness` | `float` | Learned effectiveness [0.0, 1.0] |
| `context_hash` | `str` | SHA-256 hash for context similarity matching (required) |
| `created_at` | `float` | Timestamp of lesson creation (default: `time.time()`) |
| `times_applied` | `int` | Number of times this lesson has been applied (default: 0) |

---

### `ReflectionEngine`

#### Constructor

```python
ReflectionEngine(
    quality_threshold: float = 0.5,
    max_reflections_per_step: int = 2,
    reflection_bank_size: int = 100,
    periodic_interval: int = 5,
)
```

#### Methods

##### `should_reflect`

```python
def should_reflect(
    self, confidence: float, risk_level: float = 0.0,
    goal_drift: float = 0.0, tool_failures: int = 0,
    step_number: int = 0, is_final_step: bool = False,
) -> Tuple[bool, Optional[ReflectionTrigger]]
```

Decide whether reflection is needed. Returns `(flag, trigger)`. Respects `max_reflections_per_step` limit.

##### `build_reflection_prompt`

```python
def build_reflection_prompt(
    self, goal: str, step_description: str, response: str,
    tool_results: List[Dict[str, Any]],
    relevant_lessons: Optional[List[ReflectionLesson]] = None,
) -> str
```

Build prompt for LLM quality verification. Requests JSON output with `quality_score`, `issues`, and `suggested_fix`.

##### `parse_reflection`

```python
def parse_reflection(self, llm_response: str, trigger: ReflectionTrigger) -> ReflectionResult
```

Parse LLM reflection response. Tries JSON parsing first, then keyword-based text fallback.

##### `build_improvement_prompt`

```python
def build_improvement_prompt(self, original_response: str, issues: List[str], goal: str) -> str
```

Build prompt to improve a response based on identified issues.

##### `store_lesson` / `get_relevant_lessons` / `record_lesson_outcome`

Lesson management: store mistakes and corrections, retrieve relevant lessons ranked by task_type and effectiveness, update effectiveness based on outcomes.

##### `get_stats`

```python
def get_stats() -> Dict[str, Any]
```

Return reflection statistics: `total_reflections`, `retries_triggered`, `improvements_made`, `lessons_stored`, `bank_capacity`, `average_lesson_effectiveness`, `quality_threshold`.

##### `reset_step_counts`

```python
def reset_step_counts() -> None
```

Reset per-step reflection counts for a new task. Called from `AgentLoop.start()` at the beginning of each goal.

---

## Usage Example

```python
from corteX.engine.reflection import ReflectionEngine

reflector = ReflectionEngine(quality_threshold=0.5)

should, trigger = reflector.should_reflect(confidence=0.3)
if should:
    prompt = reflector.build_reflection_prompt(
        goal="Build API", step_description="Create endpoints",
        response=agent_output, tool_results=[],
    )
    llm_response = await llm.generate(prompt)
    result = reflector.parse_reflection(llm_response, trigger)

    if not result.passes_quality_gate:
        fix_prompt = reflector.build_improvement_prompt(
            agent_output, result.issues_found, "Build API"
        )
        improved = await llm.generate(fix_prompt)

    # Store lesson for future reference
    reflector.store_lesson("api_design", "forgot auth", "always add auth middleware")
```

---

## See Also

- [Agent Loop API](./agent-loop.md)
- [Decision Log API](./decision-log.md)
