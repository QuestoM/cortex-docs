# Model Mosaic API Reference

## Module: `corteX.engine.model_mosaic`

Multi-model collaboration within a single logical turn. Prepares prompts and merges results for multi-stage model patterns: single model, draft-polish, plan-critique-refine, and ensemble voting.

Brain analogy: Cortical columns (parallel processing), Broca-Wernicke circuit (draft-then-refine), prefrontal ensemble (multiple experts voting).

---

## Classes

### `MosaicPattern`

**Type**: `str, Enum`

Multi-model collaboration patterns available in the mosaic system.

| Value | Description |
|-------|-------------|
| `SINGLE_MODEL` | Direct single-model execution |
| `DRAFT_POLISH` | Two-stage: fast draft, then refined polish |
| `PLAN_CRITIQUE_REFINE` | Three-stage: plan, critique, refine |
| `ENSEMBLE_VOTE` | Three parallel experts with consensus voting |

---

### `MosaicResult`

**Type**: `@dataclass`

Unified result from a multi-model pattern execution.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `content` | `str` | Final merged content from the pattern |
| `pattern_used` | `MosaicPattern` | Which pattern was executed |
| `models_involved` | `List[str]` | Model IDs used across all stages |
| `cost_tokens` | `int` | Total tokens consumed across all stages |
| `confidence` | `float` | Confidence in the result [0.0, 1.0] |
| `stage_count` | `int` | Number of stages executed (default: 1) |
| `merge_strategy` | `str` | How results were merged (e.g., `"direct"`, `"consensus"`) |
| `latency_ms` | `float` | Total latency in milliseconds |
| `metadata` | `Dict[str, Any]` | Additional metadata |

---

### `MosaicStagePrompt`

**Type**: `@dataclass`

Prompt prepared for one stage of a mosaic pattern. The SDK orchestrator routes each stage prompt to the appropriate model.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `stage_name` | `str` | Name of this stage (e.g., `"draft"`, `"critique"`) |
| `stage_index` | `int` | Zero-based index in the stage sequence |
| `role_hint` | `str` | Suggested model role (e.g., `"orchestrator"`, `"judge"`, `"worker"`) |
| `system_prompt` | `str` | System prompt for this stage |
| `user_prompt` | `str` | User prompt for this stage |
| `temperature_hint` | `float` | Suggested temperature (default: 0.7) |
| `max_tokens_hint` | `int` | Suggested max tokens (default: 4096) |
| `requires_previous` | `bool` | Whether this stage needs the previous stage's output |

---

### `ModelMosaic`

Prepares multi-model collaboration prompts and merges results. This class does NOT call LLMs itself -- it builds stage prompts and merges stage results. The SDK orchestrator is responsible for routing each stage prompt to the appropriate model.

#### Constructor

```python
ModelMosaic()
```

No parameters. Initializes execution counters and pattern statistics.

#### Methods

##### `select_pattern`

```python
def select_pattern(
    self,
    complexity: float,
    budget_remaining_ratio: float = 1.0,
    allow_ensemble: bool = True,
    allow_multi_stage: bool = True,
) -> MosaicPattern
```

Choose a mosaic pattern based on task complexity and remaining budget.

**Parameters**:

- `complexity` (float): Task complexity score [0.0, 1.0]
- `budget_remaining_ratio` (float): Fraction of budget remaining [0.0, 1.0]
- `allow_ensemble` (bool): Whether ensemble voting is permitted
- `allow_multi_stage` (bool): Whether multi-stage patterns are permitted

**Returns**: `MosaicPattern` -- The selected pattern. Selection thresholds:

- `ENSEMBLE_VOTE`: complexity >= 0.8, budget > 0.5
- `PLAN_CRITIQUE_REFINE`: complexity >= 0.6, budget > 0.3
- `DRAFT_POLISH`: complexity >= 0.4, budget > 0.25
- `SINGLE_MODEL`: default / low budget

**Example**:

```python
from corteX.engine.model_mosaic import ModelMosaic

mosaic = ModelMosaic()
pattern = mosaic.select_pattern(complexity=0.75, budget_remaining_ratio=0.6)
# pattern = MosaicPattern.PLAN_CRITIQUE_REFINE
```

##### `build_mosaic_prompts`

```python
def build_mosaic_prompts(
    self,
    pattern: MosaicPattern,
    messages: List[Dict[str, str]],
    tools: Optional[List[Dict[str, Any]]] = None,
    goal_context: str = "",
) -> List[MosaicStagePrompt]
```

Build prompts for each stage of the selected mosaic pattern.

**Parameters**:

- `pattern` (MosaicPattern): The mosaic pattern to build prompts for
- `messages` (List[Dict[str, str]]): Conversation messages (role/content dicts)
- `tools` (Optional[List[Dict]]): Available tool definitions
- `goal_context` (str): Optional goal reminder text

**Returns**: `List[MosaicStagePrompt]` -- Ordered list of stage prompts to execute

##### `merge_results`

```python
def merge_results(
    self,
    pattern: MosaicPattern,
    stage_results: List[str],
    models_used: Optional[List[str]] = None,
    total_tokens: int = 0,
) -> MosaicResult
```

Merge stage results into a unified `MosaicResult`.

**Parameters**:

- `pattern` (MosaicPattern): The pattern that was executed
- `stage_results` (List[str]): Outputs from each stage (in order)
- `models_used` (Optional[List[str]]): Model IDs used for each stage
- `total_tokens` (int): Total tokens consumed across all stages

**Returns**: `MosaicResult` -- Merged result with confidence score. Confidence by pattern: SINGLE_MODEL=0.7, DRAFT_POLISH=0.8, PLAN_CRITIQUE_REFINE=0.85, ENSEMBLE_VOTE=0.6+agreement*0.35

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

**Returns**: `Dict` with `total_executions` count and `pattern_usage` breakdown.

---

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `_ENSEMBLE_THRESHOLD` | `0.8` | Minimum complexity for ensemble voting |
| `_PCR_THRESHOLD` | `0.6` | Minimum complexity for plan-critique-refine |
| `_DRAFT_POLISH_THRESHOLD` | `0.4` | Minimum complexity for draft-polish |

---

## Usage Example

```python
from corteX.engine.model_mosaic import ModelMosaic, MosaicPattern

mosaic = ModelMosaic()

# 1. Select pattern based on task complexity
pattern = mosaic.select_pattern(complexity=0.7, budget_remaining_ratio=0.8)

# 2. Build prompts for each stage
messages = [{"role": "user", "content": "Implement a REST API for user management"}]
prompts = mosaic.build_mosaic_prompts(pattern, messages, goal_context="Build API")

# 3. Execute each prompt via SDK (external)
stage_results = []
for prompt in prompts:
    response = await llm.generate(prompt.system_prompt, prompt.user_prompt)
    stage_results.append(response)

# 4. Merge results
result = mosaic.merge_results(pattern, stage_results, total_tokens=5000)
print(result.content)
print(f"Confidence: {result.confidence}, Strategy: {result.merge_strategy}")
```

---

## See Also

- [Intelligent Model Routing Concept](../../concepts/model-routing.md)
- [Dual-Process Routing](../../concepts/brain/dual-process.md)
