# corteX Documentation vs Code Audit Report
## Generated: 2026-02-22
## Audited by: 10 parallel Claude Opus 4.6 agents

This report documents all discrepancies between the SDK documentation (cortex-docs) and the actual source code (corteX/).

## Executive Summary

| Metric | Value |
|--------|-------|
| Pages audited | 113 |
| Pages fully accurate (OK) | 65 (58%) |
| Pages with gaps (A) | 48 (42%) |
| Total gaps found | ~164 |
| CRITICAL gaps (wrong/fabricated API) | 4 pages |
| HIGH gaps (wrong signatures/examples) | 8 pages |
| MEDIUM gaps (missing APIs users need) | 18 pages |
| LOW gaps (minor omissions) | 18 pages |

## Gap Categories
- **WRONG_API**: Documentation describes methods/classes that don't exist in code
- **WRONG_ATTR**: Documentation lists attributes that don't exist or are named differently
- **MISSING_API**: Code has methods/classes not documented
- **WRONG_SIGNATURE**: Method exists but parameters/return type differ
- **WRONG_EXAMPLE**: Code example uses non-existent API
- **WRONG_DESCRIPTION**: Behavior described doesn't match implementation

## Action Types
- **DOC_FIX**: Fix the documentation to match existing code (vast majority)
- **CODE_IMPL**: Implement feature in code that docs describe (rare - only if genuinely useful)

---

## CRITICAL Priority (Complete doc rewrite needed)

### 1. reference/engine/modulator.md [DOC_FIX]
**Source**: `corteX/engine/modulator.py` (1,855 lines)
**Problem**: Entire API section is fabricated. 4 phantom methods, wrong dataclass attributes.
- `add_modulation()` does not exist -> Real: `activate()`, `silence()`, `amplify()`, `dampen()`, `clamp()`
- `remove_modulation()` does not exist -> Real: `remove()`
- `apply(target, value)` does not exist -> Real: `apply_modulations(weights_dict)`
- `clear_scope()` does not exist -> Real: `clear_all()`
- `factor` and `clamped_value` attributes don't exist -> All use `strength`
- Usage example completely wrong
- Missing: `scope_param`, `reason`, safety boundaries, enterprise policies, conditional modulation, conflict detection, audit trail

### 2. reference/engine/reorganization.md [DOC_FIX]
**Source**: `corteX/engine/reorganization.py`
**Problem**: Constructor params wrong, 6+ method names wrong, example broken.
- Constructor: ALL parameter names differ (`learning_rate` vs `decay_factor`, etc.)
- `add_entity()` -> Real: `register_entity()`
- `record_usage(entity, reward)` -> Real: `record_usage(entities_list, success, quality)`
- `record_co_activation()` doesn't exist (automatic in `record_usage`)
- `check_fusion()` doesn't exist (internal to `reorganize()`)
- `apply_decay()` doesn't exist (internal to `maintenance()`)
- `trigger_reorganization()` -> Real: `reorganize()`
- `get_all_territories()` -> Real: `get_territory_map()`
- Missing: EntityType enum, TerritoryAllocation, supporting classes

### 3. reference/security/vault.md [DOC_FIX]
**Source**: `corteX/security/vault.py`
**Problem**: Describes XOR obfuscation but code uses Fernet (AES-256-CBC + HMAC-SHA256).
- Doc says "XOR with tenant-derived SHA-256 pad" -> Code does Fernet with PBKDF2 (600k iterations)
- "Not Production Cryptography" warning is now wrong (IS production crypto)
- Entire Security Design table is stale

### 4. reference/engine/columns.md [DOC_FIX]
**Source**: `corteX/engine/columns.py`
**Problem**: 2 fabricated methods on ColumnCompetition.
- `lateral_inhibit()` does not exist (inhibition is inside `compete()`)
- `get_activation_map()` does not exist (only internal `_last_competition_scores`)

---

## HIGH Priority (Wrong signatures, missing critical APIs)

### 5. reference/session.md [DOC_FIX]
**Source**: `corteX/session/` (25 files)
**Problem**: 15+ missing public methods, 8+ missing attributes, incomplete close() return.
- Missing 12+ stats methods: `get_proactive_stats()`, `get_cross_modal_stats()`, `get_resource_map()`, etc.
- Missing `gdpr` property, explainability mixin methods, interop mixin methods
- Missing attributes: `proactive`, `cross_modal`, `resource_map`, `reorganizer`, `modulator`, `simulator`, `adaptation`, `quality_estimator`
- `close()` return dict shows ~10 keys but actual returns 40+ stat keys

### 6. reference/engine.md + reference/agent.md [DOC_FIX]
**Source**: `corteX/sdk.py`
**Problem**: Missing `mcp_servers` and `a2a_agents` parameters from interop integration.
- `create_agent()` missing `mcp_servers: Optional[List]` and `a2a_agents: Optional[List]`
- Agent constructor missing same parameters
- Agent attributes table missing these fields

### 7. reference/llm/router.md [DOC_FIX]
**Source**: `corteX/core/llm/router.py`
**Problem**: 8 gaps - missing constructor param, 5 undocumented methods, 3 missing properties.
- Missing: `data_classification_enabled` constructor param
- Missing methods: `set_orchestrator_model()`, `get_orchestrator_model()`, `set_worker_model()`, `get_worker_model()`, `set_data_classification_enabled()`, `set_pii_protection_enabled()`
- Missing properties: `data_classification_enabled`, `pii_protection_enabled`, `pii_tokenizer`

### 8. reference/engine/context-compiler.md [DOC_FIX]
**Source**: `corteX/engine/context_compiler.py`
**Problem**: 6+ missing public methods that users need for custom orchestrators.
- `set_user_preferences()`, `set_policies()`, `set_tool_definitions()`, `set_brain_digest()`, `set_task_state()`, `add_recent_message()`, `get_compaction_level()`, `get_stats()`
- Missing `ContextZone` dataclass

### 9. reference/engine/concepts.md [DOC_FIX]
**Source**: `corteX/engine/concepts.py`
**Problem**: 3 major classes mentioned but not documented, 3 important methods missing.
- `ConceptFormationEngine` - undocumented
- `GraphQueryEngine` - undocumented
- `ConceptGraphManager` - undocumented
- Missing: `spreading_activation()`, `hebbian_update_edges()`, `inhibit_competing_concepts()`

### 10. reference/enterprise/explainability.md [DOC_FIX]
**Source**: `corteX/enterprise/explainability.py`
**Problem**: Multiple dataclasses missing attributes, 4 supporting types undocumented.
- `DecisionExplanation` missing `latency_ms`, `tokens_consumed`, `metadata`
- `ToolSelectionExplanation` missing 4 attributes
- `RoutingExplanation` missing 4 attributes
- Undocumented types: `AlternativeConsidered`, `WeightInfluence`, `GoalAlignmentInfo`, `DecisionStep`

### 11. reference/security/compliance.md [DOC_FIX]
**Source**: `corteX/security/compliance.py`
**Problem**: Missing `consent_manager` constructor param and consent auto-resolution.
- Constructor missing `consent_manager: Optional[Any]` param
- `pre_check()` consent auto-resolution behavior undocumented
- ISO 27001 enforcement details missing

### 12. reference/security/tenant-encryption.md [DOC_FIX]
**Source**: `corteX/security/tenant_encryption.py`
**Problem**: 7 gaps including wrong vault reference, missing error conditions.
- "XOR obfuscation" reference in See Also (vault now uses Fernet)
- Missing: `_validate_tenant_id` raises for all methods, RuntimeError on cache full
- Thread safety not documented, rotate without prior key behavior undocumented

---

## MEDIUM Priority (Missing useful APIs, incomplete docs)

### 13. reference/engine/context.md [DOC_FIX]
- Missing 3 ContextConfig attrs: `l2_max_tokens`, `l3_max_tokens`, `max_total_tokens`
- Missing `TaskState.tool_usage_summary`
- Missing 4 CorticalContextEngine methods: `should_compress()`, `should_checkpoint()`, `create_checkpoint()`, `record_tokens_spent()`

### 14. reference/engine/attention.md [DOC_FIX]
- Missing `ProcessingBudget` dataclass (returned by `get_processing_budget()`)
- Missing `adaptation_filter` constructor param
- Missing `AttentionSystem` constructor documentation
- Missing `ChangeEvent.timestamp` and `StateFingerprint.timestamp`

### 15. reference/engine/calibration.md [DOC_FIX]
- Missing `MetaCognitionAlert.timestamp` attribute
- Missing `CalibrationReport` class
- Missing `ContinuousCalibrationEngine.get_alarming_domains()`

### 16. reference/engine/game-theory.md [DOC_FIX]
- Missing: `DualProcessRouter.to_dict()`, `ReputationSystem.get_stats()`, `ReputationSystem.to_dict()`, `TruthfulScoringMechanism.get_all_credibilities()`, `ShapleyAttributor.get_running_shapley()`

### 17. reference/engine/decision-log.md [DOC_FIX]
- Missing: `get_entry()`, `get_by_step()`, `size` property, `clear()`, `DecisionEntry` field defaults

### 18. reference/engine/ab-test-manager.md [DOC_FIX]
- Missing: `should_use_test()`, `get_active_tests()`, `get_test()`, `get_stats()`

### 19. reference/engine/reflection.md [DOC_FIX]
- Missing `ReflectionResult.reflection_time_ms`
- Missing `ReflectionLesson.context_hash`, `created_at`, `times_applied`
- Missing `get_stats()` and `reset_step_counts()`

### 20. reference/engine/planner.md [DOC_FIX]
- Missing `can_retry()`, `get_plan()`
- Missing `PlanStep.started_at`, `completed_at`

### 21. reference/engine/agent-loop.md [DOC_FIX]
- Missing `LoopState.retry_count`, `started_at`
- Missing `get_goal_dna()`, `get_goal_reminder()`

### 22. reference/enterprise/config.md [DOC_FIX]
- Missing `TenantConfig.data_classification_enabled`, `pii_protection_enabled`
- Missing `LicenseConfig.allowed_features`

### 23. reference/llm/base.md [DOC_FIX]
- Missing `DataClassificationError` in error hierarchy

### 24. reference/llm/cost-tracker.md [DOC_FIX]
- Missing `get_tenant_summary()`, `get_all_records()`

### 25. reference/engine/drift-engine.md [DOC_FIX]
- Local `GoalDNA` in drift_engine.py is different from standalone `goal_dna.GoalDNA` - not clarified
- Local `GoalDNA` dataclass not documented

### 26. reference/interop/index.md [DOC_FIX]
- Architecture diagram names `MCPToolBridge` class (it's a module of functions)
- See Also links wrong: `mcp-setup.md` -> `mcp-servers.md`, `a2a-delegation.md` -> `a2a-agents.md`

---

## LOW Priority (Minor omissions, mostly to_dict/get_stats)

### 27. reference/engine/weights.md - 4 missing `to_dict()` methods (LOW)
### 28. reference/engine/goal-tracker.md - Recommended Actions table omits precedence (LOW)
### 29. reference/engine/feedback.md - Example comment strength=0.35 should be ~0.7 (LOW)
### 30. reference/engine/cross-modal.md - Missing `AssociationLink.last_activated`, `created_at` (LOW)
### 31. reference/engine/provider-health.md - Missing `is_healthy()`, `get_stats()` (LOW)
### 32. reference/engine/progress-estimator.md - Missing `get_stats()` (LOW)
### 33. reference/engine/speculative-executor.md - `SpeculationStats` class undocumented (LOW)
### 34. reference/engine/interaction.md - Missing `record_decision()` (LOW)
### 35. reference/engine/adaptive-budget.md - Missing `reset()` method (LOW)
### 36. reference/engine/cognitive-pipeline.md - `DEFAULT_ZONE_BUDGETS` constant (LOW)
### 37. reference/engine/goal-reminder.md - `ReminderMode` is plain class not Enum (LOW)
### 38. reference/observability/cost-predictor.md - `StepEstimate.confidence` default (LOW)
### 39. reference/llm/gemini-client.md - `_extract_system_messages()` helper, `_SENTINEL` (LOW)
### 40. reference/llm/anthropic-client.md - Hardcoded `thinking_budget=10000` in streaming (LOW)
### 41. reference/llm/classifier.md - Only 4/9 lexicons explicitly listed (LOW)
### 42. reference/llm/resilience.md - `_CircuitData`/`_BucketData` internal classes (LOW)
### 43. reference/tools/decorator.md - `@tool` without parens will fail (LOW)
### 44. reference/tools/executor.md - "Weight Engine integration" claim is aspirational (LOW)
### 45. reference/security/attenuation.md - Drift risk non-linear mapping (LOW)
### 46. reference/enterprise/retention.md - `ExpiredItem.metadata` missing (LOW)
### 47. reference/neurollama/neurollama-model.md - 70B preset exit_layers wrong, GELU claim (LOW)

---

## Missing Documentation Pages (no doc exists for these modules)

| Module | Code Location | Priority |
|--------|--------------|----------|
| `TenantContext` | `corteX/tenancy/context.py` | MEDIUM |
| `TenantLayer` enum | `corteX/tenancy/context.py` | MEDIUM |
| `NoTenantBoundError` | `corteX/tenancy/context.py` | LOW |

---

## Triage: What to Implement vs What to Fix

### All gaps are DOC_FIX (fix docs to match code)
After reviewing all 164 gaps, **ZERO** require new code implementation. Every gap falls into:
1. **Documentation describes non-existent API** -> Rewrite doc to show real API
2. **Documentation misses existing API** -> Add the method/class to docs
3. **Documentation describes wrong behavior** -> Fix description

The code is the source of truth. No "feature requests" emerged from this audit.

### Fix Strategy (3 waves)
1. **Wave A (CRITICAL)**: Rewrite 4 pages from scratch (modulator, reorganization, vault, columns)
2. **Wave B (HIGH+MEDIUM)**: Update 22 pages with missing methods/attrs
3. **Wave C (LOW)**: Minor fixes to 21 pages

---

## Batch Reports (detailed findings)
- [Batch 1: Engine Core](batch1_engine_core.md) - 6 pages, 11 gaps
- [Batch 2: Engine Advanced](batch2_engine_advanced.md) - 10 pages, 30 gaps
- [Batch 3: Engine Ext + Bridge](batch3_engine_ext_bridge.md) - 12 pages, 18 gaps
- [Batch 4: Agentic + Intelligence](batch4_agentic_intelligence.md) - 14 pages, 31 gaps
- [Batch 5: Cognitive + Goal + Drift](batch5_cognitive_goal_drift.md) - 19 pages, 5 gaps
- [Batch 6: Security](batch6_security.md) - 8 pages, 14 gaps
- [Batch 7: Tenancy + Observability](batch7_tenancy_observability.md) - 7 pages, 1 gap
- [Batch 8: LLM + Tools](batch8_llm_tools.md) - 11 pages, 24 gaps
- [Batch 9: Public API + Core + Enterprise](batch9_public_core_enterprise.md) - 16 pages, ~25 gaps
- [Batch 10: Interop + NeuroLlama](batch10_interop_neurollama.md) - 10 pages, 5 gaps
