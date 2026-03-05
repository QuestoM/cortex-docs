# Model Mosaic

Model Mosaic enables multi-model collaboration within a single logical turn. Instead of routing every request to one LLM, it orchestrates parallel or sequential model patterns - draft-then-polish, plan-critique-refine, or ensemble voting - to produce higher-quality results. The `ModelMosaic` class does not call LLMs itself; it builds stage prompts and merges stage results, leaving actual LLM routing to the SDK orchestrator.

## How It Works

The system selects a collaboration pattern based on task complexity and remaining budget, then generates stage-specific prompts for each model in the pipeline. After all stages execute, results are merged into a single `MosaicResult` with a confidence score.

**Pattern selection logic:**

- Complexity >= 0.8 and budget > 50% - `ENSEMBLE_VOTE` (three independent experts, consensus extraction)
- Complexity >= 0.6 and budget > 30% - `PLAN_CRITIQUE_REFINE` (generate, critique, revise)
- Complexity >= 0.4 and budget > 25% - `DRAFT_POLISH` (fast draft, then careful polish)
- Below thresholds or budget < 20% - `SINGLE_MODEL` (direct pass-through)

The merge step varies by pattern. Ensemble voting uses token-set Jaccard overlap to find the response with highest agreement across all experts. Draft-polish and plan-critique-refine simply take the final refined output.

## Key Features

- **Four collaboration patterns** via `MosaicPattern` enum: `SINGLE_MODEL`, `DRAFT_POLISH`, `PLAN_CRITIQUE_REFINE`, `ENSEMBLE_VOTE`
- **Budget-aware selection** - automatically falls back to single-model when budget is low
- **Stage prompts with role hints** - each `MosaicStagePrompt` carries a `role_hint` ("orchestrator", "judge", "worker", "creative") and `temperature_hint`
- **Token-set consensus** for ensemble voting - no external dependencies, pure Python Jaccard overlap scoring
- **Execution statistics** tracking pattern usage counts via `get_stats()`
- **Goal context injection** - optional goal reminder text woven into stage prompts

## Integration

Model Mosaic connects with several corteX brain components:

- **Adaptive Budget** - the `budget_remaining_ratio` parameter ensures expensive multi-model patterns are only used when the budget allows
- **Goal Reminder** - goal context can be passed into `build_mosaic_prompts()` so every stage stays aligned with the original objective
- **Decision Log** - pattern selection decisions are recorded for compliance and analysis
- **Provider Health** - the orchestrator uses health data to assign stages to healthy providers

## Usage Example

```python
from corteX.engine.model_mosaic import ModelMosaic, MosaicPattern

mosaic = ModelMosaic()

# 1. Select pattern based on task complexity and budget
pattern = mosaic.select_pattern(complexity=0.75, budget_remaining_ratio=0.6)
# Returns MosaicPattern.PLAN_CRITIQUE_REFINE

# 2. Build stage prompts
prompts = mosaic.build_mosaic_prompts(
    pattern=pattern,
    messages=[{"role": "user", "content": "Implement a rate limiter"}],
    tools=[{"name": "code_edit"}],
    goal_context="Build a production-ready API",
)
# Returns 3 MosaicStagePrompt objects: plan, critique, refine

# 3. Execute each prompt via LLM (SDK handles routing)
stage_results = []
for prompt in prompts:
    result = await router.generate(prompt)  # your LLM call
    stage_results.append(result)

# 4. Merge results
final = mosaic.merge_results(
    pattern=pattern,
    stage_results=stage_results,
    models_used=["gemini-2.5-pro", "gemini-2.5-flash", "gemini-2.5-pro"],
    total_tokens=8500,
)
print(final.content)       # Refined final output
print(final.confidence)    # 0.85 for plan-critique-refine
print(final.merge_strategy)  # "refined_output"
```

## See Also

- [Adaptive Budget](../anti-drift/adaptive-budget.md) - budget constraints that influence pattern selection
- [Provider Health](provider-health.md) - health-aware model assignment
- [A/B Test Manager](ab-test-manager.md) - compare single-model vs. mosaic patterns
- [Decision Log](decision-log.md) - audit trail for pattern selection decisions
