# Documentation Audit -- Batch 2: Engine Advanced Pages

**Auditor:** Claude Opus 4.6
**Date:** 2026-02-22
**Scope:** 10 documentation pages in `cortex-docs/docs/reference/engine/` vs source code in `corteX/`

Gap categories: `WRONG_API`, `WRONG_ATTR`, `MISSING_API`, `WRONG_SIGNATURE`, `WRONG_EXAMPLE`, `WRONG_DESCRIPTION`, `FEATURE_REQUEST`

---

## Summary

| Page | Status | Gaps |
|------|--------|------|
| game-theory.md | [A] | 5 |
| context.md | [A] | 8 |
| memory.md | [OK] | 0 |
| bayesian.md | [OK] | 0 |
| calibration.md | [A] | 3 |
| columns.md | [A] | 2 |
| concepts.md | [A] | 6 |
| cross-modal.md | [A] | 1 |
| attention.md | [A] | 5 |
| resource-map.md | [OK] | 0 |
| **TOTAL** | **7 [A] / 3 [OK]** | **30** |

---

## Page 1: game-theory.md [A]

**Doc:** `cortex-docs/docs/reference/engine/game-theory.md`
**Code:** `corteX/engine/game_theory.py` (744 lines)

### Gap 1 -- MISSING_API: DualProcessRouter.to_dict()

Code has `to_dict()` at line 154. Not documented.

### Gap 2 -- MISSING_API: ReputationSystem.get_stats()

Code has `get_stats()` at line 292 returning trust_scores, consistency_scores, quarantined, total_interactions. Not documented.

### Gap 3 -- MISSING_API: ReputationSystem.to_dict()

Code has `to_dict()` at line 300. Not documented.

### Gap 4 -- MISSING_API: TruthfulScoringMechanism.get_all_credibilities()

Code has `get_all_credibilities()` at line 728 returning `Dict[str, float]`. Not documented.

### Gap 5 -- MISSING_API: ShapleyAttributor.get_running_shapley()

Code has `get_running_shapley()` at line 648 returning `Dict[str, float]`. Not documented. Relates to the `update_running()` method which IS documented.

---

## Page 2: context.md [A]

**Doc:** `cortex-docs/docs/reference/engine/context.md`
**Code:** `corteX/engine/context.py` (1122 lines)

### Gap 1 -- MISSING_ATTR: ContextConfig.l2_max_tokens

Code has `l2_max_tokens: int = 2000` at line 86. Not listed in the doc table.

### Gap 2 -- MISSING_ATTR: ContextConfig.l3_max_tokens

Code has `l3_max_tokens: int = 500` at line 87. Not listed in the doc table.

### Gap 3 -- MISSING_ATTR: ContextConfig.max_total_tokens

Code has `max_total_tokens: Optional[int] = None` at line 95. Not listed in the doc table.

### Gap 4 -- MISSING_ATTR: TaskState.tool_usage_summary

Code has `tool_usage_summary: Dict[str, Dict[str, Any]] = field(default_factory=dict)` at line 230. Not listed in the doc table. This is a meaningful field that tracks per-tool usage.

### Gap 5 -- MISSING_API: CorticalContextEngine.should_compress()

Code has `should_compress()` at line 1016 returning `bool`. Not documented but is called internally by `get_context_window()`.

### Gap 6 -- MISSING_API: CorticalContextEngine.should_checkpoint()

Code has `should_checkpoint()` at line 1021 returning `bool`. Not documented.

### Gap 7 -- MISSING_API: CorticalContextEngine.create_checkpoint()

Code has `create_checkpoint()` at line 1026 returning `ContextCheckpoint`. Not documented as a public method (only the `ContextCheckpointer.create_checkpoint()` is documented).

### Gap 8 -- MISSING_API: CorticalContextEngine.record_tokens_spent()

Code has `record_tokens_spent(tokens: int) -> None` at line 948. Not documented. This is the way to track token expenditure.

---

## Page 3: memory.md [OK]

**Doc:** `cortex-docs/docs/reference/engine/memory.md`
**Code:** `corteX/memory/manager.py` (219 lines)

All classes, methods, and attributes match. The doc correctly references `corteX.memory.manager` as the module path. All 7 public methods of `ContextBroker` are documented with correct signatures. The deprecated module-level `broker` singleton is correctly documented. No gaps found.

---

## Page 4: bayesian.md [OK]

**Doc:** `cortex-docs/docs/reference/engine/bayesian.md`
**Code:** `corteX/engine/bayesian.py` (1092 lines)

All 11 classes are documented: BetaDistribution, GammaDistribution, NormalNormalUpdater, DirichletMultinomialUpdater, BayesianSurpriseCalculator, ProspectTheoreticUpdater, BayesianToolSelector, UCB1Selector, AnchorManager, AvailabilityFilter, FrameNormalizer. Constructor signatures, properties, and method signatures all match the code. The code example is accurate. No gaps found.

---

## Page 5: calibration.md [A]

**Doc:** `cortex-docs/docs/reference/engine/calibration.md`
**Code:** `corteX/engine/calibration.py` (589 lines)

### Gap 1 -- MISSING_ATTR: MetaCognitionAlert.timestamp

Code has `timestamp: float = field(default_factory=time.time)` at line 309. Not listed in the doc's MetaCognitionAlert table. The doc lists 5 fields; code has 6.

### Gap 2 -- MISSING_API: CalibrationReport class

Code has a full `CalibrationReport` class at line 428 with methods `generate() -> Dict[str, Any]` and `generate_text() -> str`. The doc does not mention this class at all. It is used internally by `ContinuousCalibrationEngine.report()` and `report_text()`, but developers may want to use it directly.

### Gap 3 -- MISSING_API: ContinuousCalibrationEngine.get_alarming_domains()

Code has `get_alarming_domains() -> List[CalibrationDomain]` at line 554. Not documented in the ContinuousCalibrationEngine methods table (only the CalibrationTracker method is documented).

---

## Page 6: columns.md [A]

**Doc:** `cortex-docs/docs/reference/engine/columns.md`
**Code:** `corteX/engine/columns.py` (1388 lines)

### Gap 1 -- MISSING_API: FunctionalColumn.get_coactivation_count()

Code has `get_coactivation_count(other_id: str) -> int` at line 154. Not documented. This is the read-side of `record_coactivation()` which IS documented.

### Gap 2 -- WRONG_API: ColumnCompetition.lateral_inhibit / get_activation_map

Doc lists methods `lateral_inhibit(column) -> None` and `get_activation_map() -> Dict[str, float]`. Code at line 494 (ColumnCompetition class) does NOT have a method named `lateral_inhibit`. Inhibition is applied INSIDE `compete()`. Code does store `self._last_competition_scores` but there is no public `get_activation_map()` method -- only an internal `_last_competition_scores` dict. The doc invented these two methods.

---

## Page 7: concepts.md [A]

**Doc:** `cortex-docs/docs/reference/engine/concepts.md`
**Code:** `corteX/engine/concepts.py` (1000+ lines)

### Gap 1 -- MISSING_API: ConceptGraph.spreading_activation()

Code has `spreading_activation(seed_nodes, depth=2, decay_per_hop=0.5, min_activation=0.01) -> Dict[str, float]` at line 807. The doc mentions ConceptGraph but does not document this major method.

### Gap 2 -- MISSING_API: ConceptGraph.hebbian_update_edges()

Code has `hebbian_update_edges() -> int` at line 897. Not documented. This is the core Hebbian learning method that updates all edges based on current activations.

### Gap 3 -- MISSING_API: ConceptGraph.inhibit_competing_concepts()

Code has `inhibit_competing_concepts(active_concept_id: str) -> List[str]` at line 926. Not documented. This is the lateral inhibition method.

### Gap 4 -- MISSING_API: ConceptFormationEngine class

Doc mentions ConceptFormationEngine in the overview ("Automatic concept discovery from co-occurrence") but does not document any of its methods. The class exists in code.

### Gap 5 -- MISSING_API: GraphQueryEngine class

Doc mentions GraphQueryEngine in the overview as part of the architecture but does not document it.

### Gap 6 -- MISSING_API: ConceptGraphManager class

Doc mentions ConceptGraphManager in the overview ("Main entry point wrapping all subsystems") but does not document it.

---

## Page 8: cross-modal.md [A]

**Doc:** `cortex-docs/docs/reference/engine/cross-modal.md`
**Code:** `corteX/engine/cross_modal.py` (600+ lines)

### Gap 1 -- MISSING_ATTR: AssociationLink.last_activated / created_at

Code has `last_activated: float` at line 82 and `created_at: float` at line 83 and property `age_seconds` at line 86. Doc table lists 7 fields but omits `last_activated`, `created_at`, and the `age_seconds` property. These are useful for understanding link freshness.

---

## Page 9: attention.md [A]

**Doc:** `cortex-docs/docs/reference/engine/attention.md`
**Code:** `corteX/engine/attention.py` (1700+ lines)

### Gap 1 -- MISSING_ATTR: ChangeEvent.timestamp

Code has `timestamp: float = field(default_factory=time.time)` at line 139. Not listed in the doc table. The doc lists 5 fields; code has 6.

### Gap 2 -- MISSING_ATTR: StateFingerprint.timestamp

Code has `timestamp: float = field(default_factory=time.time)` at line 196. Not listed in the doc table. The doc lists 8 fields; code has 9.

### Gap 3 -- WRONG_SIGNATURE: AttentionalFilter constructor

Doc shows constructor as:
```python
AttentionalFilter(
    foreground_threshold=0.35,
    subconscious_threshold=0.10,
    subconscious_streak=4,
    critical_keywords=None,
    goal_drift_threshold=0.5,
)
```
Code has an additional parameter `adaptation_filter: Optional[Any] = None` at line 806. Missing from doc.

### Gap 4 -- MISSING_API: ProcessingBudget dataclass

Code has a full `ProcessingBudget` dataclass at line 217 with 8 fields (`max_tokens`, `model_tier`, `use_tools`, `max_tool_calls`, `use_memory_retrieval`, `use_prediction_engine`, `allow_cache`, `context_depth`). The doc describes these fields as part of the AttentionalPriority enum table (showing model tier and max tokens per level) but does not document the ProcessingBudget class itself. The `get_processing_budget()` method returns a ProcessingBudget instance, but the doc describes it returning a Dict.

### Gap 5 -- MISSING_API: AttentionSystem constructor

Doc does not show the AttentionSystem constructor. Code at line 1622 has:
```python
AttentionSystem(
    foreground_threshold=0.35,
    subconscious_threshold=0.10,
    subconscious_streak=4,
    spotlight_capacity=5,
    adaptation_filter=None,
)
```
These constructor parameters should be documented.

---

## Page 10: resource-map.md [OK]

**Doc:** `cortex-docs/docs/reference/engine/resource-map.md`
**Code:** `corteX/engine/resource_map.py` (1000 lines)

All 4 classes documented (ResourceAllocation, UsageTracker, ResourceHomunculus, AdaptiveThrottler). Constructor signatures match. Method signatures match. Field names and types match. The code example is accurate. No gaps found.

---

## Gap Summary by Category

| Category | Count | Pages Affected |
|----------|-------|----------------|
| MISSING_API | 19 | game-theory, context, calibration, columns, concepts, attention |
| MISSING_ATTR | 7 | context, calibration, cross-modal, attention |
| WRONG_API | 1 | columns |
| WRONG_SIGNATURE | 1 | attention |
| WRONG_EXAMPLE | 0 | -- |
| WRONG_DESCRIPTION | 0 | -- |
| FEATURE_REQUEST | 0 | -- |
| **TOTAL** | **30** | **7 pages** |

## Priority Recommendations

1. **HIGH -- columns.md Gap 2 (WRONG_API):** ColumnCompetition documents `lateral_inhibit()` and `get_activation_map()` which do not exist in code. These are fabricated methods. Either implement them or remove from docs.

2. **HIGH -- concepts.md Gaps 4-6 (MISSING_API):** Three full classes (ConceptFormationEngine, GraphQueryEngine, ConceptGraphManager) are mentioned in the overview but entirely undocumented. These are the main entry points for concept graph functionality.

3. **MEDIUM -- context.md Gaps 1-3 (MISSING_ATTR):** Three ContextConfig fields (`l2_max_tokens`, `l3_max_tokens`, `max_total_tokens`) are undocumented. Developers need these to configure progressive summarization budgets.

4. **MEDIUM -- attention.md Gap 4 (MISSING_API):** ProcessingBudget dataclass is returned by `get_processing_budget()` but not documented. Developers receive this object but cannot reference its fields.

5. **LOW -- All `to_dict` / `get_stats` omissions:** Multiple classes have utility methods (`to_dict`, `get_stats`, `get_running_shapley`, etc.) that are not documented. These are useful for serialization and debugging but are not critical API surface.
