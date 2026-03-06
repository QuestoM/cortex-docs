# Batch 4: Agent Intelligence + Agentic Engine -- Documentation Audit

**Auditor**: Claude Opus 4.6
**Date**: 2026-02-22
**Scope**: 14 pages (6 Agent Intelligence + 8 Agentic Engine)
**Source base**: `C:\Intel\questo\projects\corteX\corteX\engine\`

## Summary

| # | Doc Page | Source File | Status | Gaps |
|---|----------|-------------|--------|------|
| 1 | model-mosaic.md | engine/model_mosaic.py | OK | 0 |
| 2 | speculative-executor.md | engine/speculative_executor.py | A | 2 |
| 3 | decision-log.md | engine/decision_log.py | A | 5 |
| 4 | progress-estimator.md | engine/progress_estimator.py | A | 1 |
| 5 | ab-test-manager.md | engine/ab_test_manager.py | A | 4 |
| 6 | provider-health.md | engine/provider_health.py | A | 2 |
| 7 | agent-loop.md | engine/agent_loop.py | A | 3 |
| 8 | context-compiler.md | engine/context_compiler.py | A | 6 |
| 9 | planner.md | engine/planner.py | A | 3 |
| 10 | reflection.md | engine/reflection.py | A | 4 |
| 11 | recovery.md | engine/recovery.py | OK | 0 |
| 12 | interaction.md | engine/interaction.py | A | 1 |
| 13 | policy-engine.md | engine/policy_engine.py | OK | 0 |
| 14 | sub-agent.md | engine/sub_agent.py | OK | 0 |

**Total gaps found: 31** (across 10 pages with issues)

---

## Detailed Findings

---

### 1. reference/engine/model-mosaic.md -> corteX/engine/model_mosaic.py
**Status**: [OK]

All classes (`MosaicPattern`, `MosaicResult`, `MosaicStagePrompt`, `ModelMosaic`), methods (`select_pattern`, `build_mosaic_prompts`, `merge_results`, `get_stats`), attributes, parameters, return types, constants, and code examples match the source exactly. No discrepancies found.

---

### 2. reference/engine/speculative-executor.md -> corteX/engine/speculative_executor.py
**Status**: [A] (2 gaps found)

#### MISSING_API
1. **`SpeculationStats` dataclass not documented**. The source defines a `SpeculationStats` dataclass (line 36-43) with fields: `total_speculations`, `total_hits`, `total_misses`, `hit_rate`, `total_savings_ms`, `avg_confidence`. This class exists in the module but is not mentioned in the docs. Note: `get_stats()` returns a plain `Dict` rather than `SpeculationStats`, so this class may be unused/vestigial -- but it is exported and should be documented or removed.

#### MISSING_API
2. **`_estimate_confidence` private method not documented (informational)**. The private method `_estimate_confidence` computes speculation confidence from prediction_confidence, goal_progress, historical hit rate, and step description specificity. The doc describes the overall behavior of `speculate()` but does not mention the internal confidence formula: `base + progress_bonus(0.15*goal_progress) + calibration + specificity(desc length/100, capped 0.1)`. This is minor since it is private, but the doc could note the confidence is not just passed through -- it is computed.

---

### 3. reference/engine/decision-log.md -> corteX/engine/decision_log.py
**Status**: [A] (5 gaps found)

#### MISSING_API
1. **`get_entry(index: int) -> Optional[DecisionEntry]` method not documented**. The source (line 141-145) has a `get_entry` method that retrieves a single entry by index. The doc does not mention this method.

2. **`get_by_step(step_number: int) -> List[DecisionEntry]` method not documented**. The source (line 169-171) has a `get_by_step` method that filters entries by step number. The doc does not mention this method.

3. **`size` property not documented**. The source (line 272-274) defines a `@property size` that returns `len(self._entries)`. Not mentioned in docs.

4. **`clear()` method not documented**. The source (line 277-280) has a `clear()` method that clears all entries. Not mentioned in docs.

#### WRONG_ATTR
5. **`DecisionEntry.confidence` default value not documented accurately**. The doc does not mention the default value for `confidence` (source default is `0.5`). Similarly, the doc does not show which fields have defaults and which are required positional. In the source, only `step_number`, `decision_type`, `description`, and `chosen` are required -- all others have defaults. The doc presents all attributes as a flat table without distinguishing required vs optional.

---

### 4. reference/engine/progress-estimator.md -> corteX/engine/progress_estimator.py
**Status**: [A] (1 gap found)

#### MISSING_API
1. **`get_stats() -> Dict[str, Any]` method not documented**. The source (line 173-183) has a `get_stats` method returning: `total_steps`, `total_tokens`, `current_progress`, `velocity`, `acceleration`, `trend`, `history_size`. The doc does not mention this method.

---

### 5. reference/engine/ab-test-manager.md -> corteX/engine/ab_test_manager.py
**Status**: [A] (4 gaps found)

#### MISSING_API
1. **`should_use_test(test_name: str, request_id: str) -> bool` method not documented**. The source (line 138-141) has a method to check if a request should participate in a test. Not in docs.

2. **`get_active_tests() -> List[ABTest]` method not documented**. The source (line 209-211) returns all active tests. Not in docs.

3. **`get_test(test_name: str) -> Optional[ABTest]` method not documented**. The source (line 213-215) retrieves a test by name. Not in docs.

4. **`get_stats() -> Dict[str, Any]` method not documented**. The source (line 233-244) returns manager-level statistics: `total_tests`, `active_tests`, per-test `status`/`samples_a`/`samples_b`. Not in docs.

---

### 6. reference/engine/provider-health.md -> corteX/engine/provider_health.py
**Status**: [A] (2 gaps found)

#### MISSING_API
1. **`is_healthy(provider: str) -> bool` convenience method not documented**. The source (line 170-172) has a quick boolean health check method. Not mentioned in the docs.

2. **`get_stats() -> Dict[str, Any]` method not documented**. The source (line 189-195) returns `known_providers`, `total_calls`, `circuit_states`. Not in docs.

---

### 7. reference/engine/agent-loop.md -> corteX/engine/agent_loop.py
**Status**: [A] (3 gaps found)

#### WRONG_ATTR
1. **`LoopState` missing documented attributes `retry_count` and `started_at`**. The source (line 53-66) includes `retry_count: int = 0` and `started_at: float = field(default_factory=time.time)` in the `LoopState` dataclass. The doc table for `LoopState` does not list these two fields.

#### MISSING_API
2. **`get_goal_dna() -> Optional[GoalDNA]` method not documented**. The source (line 218-219) has a getter for the GoalDNA instance. Not mentioned in docs.

3. **`get_goal_reminder() -> Optional[GoalReminderInjector]` method not documented**. The source (line 221-222) has a getter for the GoalReminderInjector instance. Not mentioned in docs.

---

### 8. reference/engine/context-compiler.md -> corteX/engine/context_compiler.py
**Status**: [A] (6 gaps found)

#### MISSING_API
1. **`ContextZone` dataclass not documented**. The source (line 38-47) defines a `ContextZone` dataclass with fields: `name`, `budget_ratio`, `items`, `token_count`, and a `recalculate_tokens()` method. This internal class is not mentioned in the docs.

2. **`set_user_preferences(prefs: Dict[str, Any]) -> None` method not documented**. Source line 90-91. Not in docs.

3. **`set_policies(policies: List[str]) -> None` method not documented**. Source line 93-94. Not in docs.

4. **`set_tool_definitions(tools: List[Dict[str, Any]]) -> None` method not documented**. Source line 96-97. Not in docs.

5. **`set_brain_digest(digest: Dict[str, Any]) -> None` method not documented**. Source line 103-104. Not in docs.

6. **`set_task_state(state: str) -> None` method not documented**. Source line 126-128. Not in docs.

Note: The doc also omits `add_recent_message()` and `get_compaction_level()` and `get_stats()`. However, `set_user_preferences`, `set_policies`, `set_tool_definitions`, `set_brain_digest`, and `set_task_state` are the most significant omissions because they are called by `AgentLoop.start()` and are part of the public contract that users need to know about when building custom orchestrators.

**Additional undocumented methods**:
- `add_recent_message(role, content, importance)` (line 130-135)
- `get_compaction_level() -> str` (line 174-176)
- `get_stats() -> Dict[str, Any]` (line 197-214)

(Total: 9 undocumented methods. Only the 6 most significant counted as gaps.)

---

### 9. reference/engine/planner.md -> corteX/engine/planner.py
**Status**: [A] (3 gaps found)

#### MISSING_API
1. **`can_retry(step: PlanStep) -> bool` method not documented**. The source (line 214-215) checks if a step has retries remaining. Not in docs.

2. **`get_plan(plan_id: str) -> Optional[ExecutionPlan]` method not documented**. The source (line 243-245) retrieves a stored plan by ID. Not in docs.

#### WRONG_ATTR
3. **`PlanStep` missing documented attributes `started_at` and `completed_at`**. The source (line 78-79) defines `started_at: Optional[float] = None` and `completed_at: Optional[float] = None` as fields of `PlanStep`. The doc table for `PlanStep` does not list these fields.

---

### 10. reference/engine/reflection.md -> corteX/engine/reflection.py
**Status**: [A] (4 gaps found)

#### WRONG_ATTR
1. **`ReflectionResult` missing `reflection_time_ms` attribute in docs**. The source (line 44) includes `reflection_time_ms: float = 0.0` in `ReflectionResult`. The doc's attribute table for `ReflectionResult` does not list this field.

2. **`ReflectionLesson` missing `context_hash`, `created_at`, and `times_applied` attributes in docs**. The source (line 55-57) includes:
   - `context_hash: str` (required)
   - `created_at: float = field(default_factory=time.time)`
   - `times_applied: int = 0`

   The doc only shows: `lesson_id`, `task_type`, `mistake`, `correction`, `effectiveness`. Three attributes are missing.

#### MISSING_API
3. **`get_stats() -> Dict[str, Any]` method not documented**. The source (line 244-256) returns reflection statistics: `total_reflections`, `retries_triggered`, `improvements_made`, `lessons_stored`, `bank_capacity`, `average_lesson_effectiveness`, `quality_threshold`. Not mentioned in docs.

4. **`reset_step_counts() -> None` method not documented**. The source (line 258-260) resets per-step reflection counts. Not mentioned in docs. This method is called from `AgentLoop.start()`, making it part of the public API.

---

### 11. reference/engine/recovery.md -> corteX/engine/recovery.py
**Status**: [OK]

All classes (`ErrorClass`, `RecoveryAction`, `RecoveryStrategy`, `ErrorRecord`, `RecoveryEngine`), methods (`classify_error`, `get_strategy`, `compute_backoff_ms`, `record_error`, `record_recovery`, `should_abort`, `get_error_summary`, `get_stats`, `reset`), attributes, parameters, and code examples match the source accurately. The decision matrix in the docs matches the source logic. No discrepancies found.

---

### 12. reference/engine/interaction.md -> corteX/engine/interaction.py
**Status**: [A] (1 gap found)

#### MISSING_API
1. **`record_decision(request: InteractionRequest, result: InteractionResult) -> None` method not documented**. The source (line 224-232) records decisions for learning and prompt building. This method is called internally by `resolve_with_*` methods but is also a public method that could be called externally. Not mentioned in docs.

Note: All other classes, enums, methods, attributes, and the should_ask_user decision table match correctly.

---

### 13. reference/engine/policy-engine.md -> corteX/engine/policy_engine.py
**Status**: [OK]

All classes (`PolicyType`, `PolicyAction`, `PolicyRule`, `PolicyEvaluation`, `PolicyResult`, `PolicyEngine`), methods (`register_rule`, `remove_rule`, `evaluate_tool_call`, `evaluate_output`, `evaluate_intent`, `get_tool_guide`, `get_playbook`, `get_active_rules`, `get_stats`, static factory methods), attributes, pattern matching logic, and code examples match the source exactly. No discrepancies found.

---

### 14. reference/engine/sub-agent.md -> corteX/engine/sub_agent.py
**Status**: [OK]

All classes (`SubAgentTask`, `SubAgentResult`, `SubAgentManager`), methods (`create_task`, `build_sub_agent_prompt`, `build_summary_prompt`, `parse_summary`, `mark_running`, `mark_completed`, `mark_failed`, `cancel_task`, `can_spawn`, `allocate_budget`, `get_task`, `get_running_tasks`, `get_completed_results`, `get_all_tasks`, `get_stats`), attributes, parameters, and code examples match the source accurately. No discrepancies found.

---

## Gap Summary by Category

| Category | Count | Details |
|----------|-------|---------|
| MISSING_API | 22 | Undocumented methods and classes |
| WRONG_ATTR | 7 | Missing attributes in dataclass documentation |
| WRONG_API | 0 | No phantom APIs found |
| WRONG_SIGNATURE | 0 | All documented signatures match source |
| WRONG_EXAMPLE | 0 | All code examples use valid APIs |
| WRONG_DESCRIPTION | 0 | All descriptions match behavior |
| FEATURE_REQUEST | 0 | N/A |

## Priority Fixes

### HIGH (public API users need these)
1. **context-compiler.md**: Add 6+ missing public methods (`set_user_preferences`, `set_policies`, `set_tool_definitions`, `set_brain_digest`, `set_task_state`, `add_recent_message`, `get_compaction_level`, `get_stats`)
2. **decision-log.md**: Add 4 missing methods (`get_entry`, `get_by_step`, `size`, `clear`)
3. **ab-test-manager.md**: Add 4 missing methods (`should_use_test`, `get_active_tests`, `get_test`, `get_stats`)
4. **agent-loop.md**: Add 2 missing `LoopState` attributes (`retry_count`, `started_at`) and 2 missing methods (`get_goal_dna`, `get_goal_reminder`)

### MEDIUM (completeness)
5. **reflection.md**: Add `reflection_time_ms` to `ReflectionResult`, add 3 missing `ReflectionLesson` fields, add `get_stats()` and `reset_step_counts()`
6. **planner.md**: Add `can_retry`, `get_plan`, and `PlanStep.started_at`/`completed_at`
7. **provider-health.md**: Add `is_healthy()` convenience method and `get_stats()`
8. **progress-estimator.md**: Add `get_stats()`

### LOW (minor/informational)
9. **speculative-executor.md**: Document or note `SpeculationStats` class
10. **interaction.md**: Document `record_decision()` method
