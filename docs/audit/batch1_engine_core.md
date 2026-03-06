# Documentation Audit: Engine Brain Core (Batch 1)

**Auditor**: Claude Opus 4.6 documentation audit agent
**Date**: 2026-02-22
**Scope**: 6 doc pages vs 6 source files in `corteX/engine/`

---

## Summary

| Page | Status | Gaps |
|------|--------|------|
| `reference/engine/weights.md` | [A] | 5 gaps |
| `reference/engine/goal-tracker.md` | [A] | 3 gaps |
| `reference/engine/feedback.md` | [A] | 3 gaps |
| `reference/engine/prediction.md` | [OK] | 0 gaps |
| `reference/engine/plasticity.md` | [OK] | 0 gaps |
| `reference/engine/adaptation.md` | [OK] | 0 gaps |

**Total gaps found: 11**

---

### reference/engine/weights.md -> corteX/engine/weights.py
**Status**: [A] (5 gaps found)

#### WRONG_API
_(none)_

#### WRONG_ATTR
_(none)_

#### MISSING_API
1. **`ModelSelectionWeights.to_dict()`** -- Code has `to_dict() -> Dict[str, Dict[str, float]]` at line 320 but docs do not mention it.
2. **`UserInsightWeights.to_dict()`** -- Code has `to_dict() -> Dict[str, Any]` at line 356 but docs do not mention it.
3. **`EnterpriseWeights.to_dict()`** -- Code has `to_dict() -> Dict[str, Any]` at line 401 but docs do not mention it.
4. **`GlobalWeights.to_dict()`** -- Code has `to_dict() -> Dict[str, Any]` at line 429 but docs do not mention it.

#### WRONG_SIGNATURE
_(none)_

#### WRONG_EXAMPLE
_(none)_

#### WRONG_DESCRIPTION
1. **UserInsightWeights tier label** -- Doc says "Tier 2 feedback weights" but code docstring says "Tier 2 Feedback" (consistent). However, the doc header says "Tier 2 feedback weights" while the code actually tracks it as a separate category (WeightCategory.USER_INSIGHTS). The doc description is accurate enough to not be a real issue, but the tier numbering is slightly inconsistent: the intro says "seven weight categories" and calls this "Tier 2" while the WeightCategory enum has it at position 5 of 7. The tier numbering in the doc section headers (Tier 1, Tier 2, Tier 2, Tier 3, Tier 3, Tier 4) is confusing but actually mirrors the code's own grouping. This is a minor labeling inconsistency rather than a factual error.

#### FEATURE_REQUEST
_(none)_

---

### reference/engine/goal-tracker.md -> corteX/engine/goal_tracker.py
**Status**: [A] (3 gaps found)

#### WRONG_API
_(none)_

#### WRONG_ATTR
_(none)_

#### MISSING_API
_(none)_

#### WRONG_SIGNATURE
_(none)_

#### WRONG_EXAMPLE
_(none)_

#### WRONG_DESCRIPTION
1. **Recommended Actions table: "abort" condition is misleading** -- Doc says `"abort"` fires when `alignment < 0.3` (no loop/drift override). Code at line 299 shows `alignment < 0.3` returns `"abort"` BUT only if it was not already caught by the earlier checks for loop/drift/stall (which all return `"replan"`). The doc's parenthetical "(no loop/drift override)" is a reasonable clarification, but importantly the doc also says `alignment < 0.5` triggers `"adjust"` -- in code (line 302) the `adjust` for `alignment < 0.5` is checked AFTER the `abort` for `alignment < 0.3`. The doc lists `"abort"` before `"adjust"` which is correct code order, but the doc table lists `"abort"` LAST after `"adjust"`, which reverses the actual evaluation order. This could mislead a reader about precedence.

   - **Doc table order**: replan -> adjust -> abort -> continue
   - **Code check order**: replan (loop) -> replan (drift >= 0.6) -> replan (stall >= 5) -> adjust (drift >= 0.3) -> abort (alignment < 0.3) -> adjust (alignment < 0.5) -> continue
   - The doc omits that `alignment < 0.5` returns `"adjust"` only if `alignment >= 0.3` (otherwise `"abort"` fires first).

2. **`verify_step` doc says alignment < 0.7 triggers LLM verification** -- Code at line 154 confirms `alignment < 0.7`, this is correct. However, the doc says the blending is "weighted 70% LLM / 30% heuristic" which matches code `0.3 * alignment + 0.7 * llm_alignment` at line 160. This is accurate.

3. **Missing `_step_times` attribute and timing behavior** -- The doc does not mention that `verify_step` records step timestamps in `_step_times` (code line 134), nor that `get_summary()` uses `_start_time` to compute `elapsed_seconds`. This is internal detail but the `elapsed_seconds` field IS documented in `get_summary()`. The `_step_times` list is tracked but never exposed publicly -- HOWEVER, it grows unboundedly since there is no pruning, which could be a memory concern for 1000s-step tasks. This is more of a code issue than a doc issue.

#### FEATURE_REQUEST
_(none)_

**Note on accuracy**: After careful review, the Recommended Actions table discrepancy (gap #1) is the only meaningful gap. The doc's table format groups conditions by action rather than by evaluation order, which could confuse developers trying to understand precedence. The code's actual precedence chain has 7 checks (3 replan conditions, 1 adjust-by-drift, 1 abort, 1 adjust-by-alignment, 1 continue), while the doc shows only 4 rows.

---

### reference/engine/feedback.md -> corteX/engine/feedback.py
**Status**: [A] (3 gaps found)

#### WRONG_API
_(none)_

#### WRONG_ATTR
_(none)_

#### MISSING_API
1. **`Tier1DirectFeedback.SATISFACTION_PATTERNS`** -- Code at line 115 has a backward-compatible flat list: `SATISFACTION_PATTERNS = [p for p, _ in SATISFACTION_PATTERNS_WEIGHTED]`. Doc does not mention this attribute. It is derived from the weighted list and kept for backward compatibility.
2. **`Tier1DirectFeedback._TOPIC_INDICATORS`** -- Code has a compiled regex pattern at line 138 used for context-aware satisfaction scoring. Doc does not mention this. It is a private attribute used in confidence computation but contributes to the public behavior described in the doc.
3. **`Tier1DirectFeedback._compute_satisfaction_confidence()`** -- Code has this method at line 145 that computes context-aware confidence. Doc describes the behavior (context-aware confidence scoring) but does not document this method specifically. Since it starts with `_`, this is a private method and omission is acceptable.

#### WRONG_SIGNATURE
_(none)_

#### WRONG_EXAMPLE
_(none)_

#### WRONG_DESCRIPTION
1. **Satisfaction signal strength value in example comment** -- Doc example comment says `# -> [FeedbackSignal(SATISFACTION, strength=0.35)]`. This is plausible but depends on the exact message. For "Thanks, that's perfect!" the code would detect both `\bthank(?:s| you)\b` (base_weight 0.9) and `\bperfect\b` (base_weight 0.7). Both would likely pass the confidence threshold for a 3-word message (short message bonus +0.3). With 2 patterns matching, strength = `min(1.0, 2 * 0.35) = 0.7`, not 0.35. The example comment is misleading about the expected strength value.

#### FEATURE_REQUEST
_(none)_

---

### reference/engine/prediction.md -> corteX/engine/prediction.py
**Status**: [OK] (0 gaps found)

#### WRONG_API
_(none)_

#### WRONG_ATTR
_(none)_

#### MISSING_API
_(none)_ -- All public methods documented. The private methods (`_outcome_error`, `_latency_error`, `_explain_surprise`, `_update_stats`) are internal and appropriately omitted from docs.

#### WRONG_SIGNATURE
_(none)_ -- All method signatures match exactly.

#### WRONG_EXAMPLE
_(none)_ -- Example uses real API correctly.

#### WRONG_DESCRIPTION
_(none)_ -- All descriptions verified against code:
- OutcomeType enum values and ranking: match code lines 254-260
- Prediction dataclass fields: match code lines 34-45
- Outcome dataclass fields: match code lines 49-56
- SurpriseSignal fields: match code lines 60-74
- Surprise magnitude formula (`0.5 * |outcome| + 0.2 * |latency| + 0.3 * |quality|`): matches code lines 208-213
- Learning signal formula (`tanh(magnitude * confidence * 2)`): matches code lines 223-224
- `get_average_surprise()` uses last 10: matches code line 338
- `get_stats()` return keys: match code lines 350-355

#### FEATURE_REQUEST
_(none)_

---

### reference/engine/plasticity.md -> corteX/engine/plasticity.py
**Status**: [OK] (0 gaps found)

#### WRONG_API
_(none)_

#### WRONG_ATTR
_(none)_

#### MISSING_API
_(none)_ -- All public methods documented. Internal attributes (`_step_count`, `_homeostasis_interval`, `_event_log`) are appropriately omitted.

#### WRONG_SIGNATURE
_(none)_ -- All signatures verified:
- `HebbianRule.apply(weight_engine, tool_name, task_type, model_name, outcome_quality)`: matches code line 53-60
- `LTPRule.apply(weight_engine, pattern_key, success)`: matches code line 113-118
- `LTDRule.apply(weight_engine, pattern_key, success)`: matches code line 159-163
- `HomeostaticRegulation.__init__(target_mean=0.5, regulation_strength=0.02)`: matches code line 201
- `CriticalPeriodModulator.__init__(critical_period_turns=10)`: matches code line 263
- `PlasticityManager.on_step_complete(tool, task_type, model, success, quality, surprise)`: matches code lines 325-332
- `PlasticityManager.get_stats()` return keys: match code lines 398-404

#### WRONG_EXAMPLE
_(none)_ -- Example uses real API correctly.

#### WRONG_DESCRIPTION
_(none)_ -- All descriptions verified:
- LTP threshold = 3 consecutive successes: matches code line 111
- LTD threshold = 2 consecutive failures: matches code line 157
- LTP bonus formula `min(0.2, 0.05 * log1p(streak - threshold + 1))`: matches code line 126
- LTD penalty formula `min(0.3, 0.1 * log1p(streak - threshold + 1))`: matches code line 172
- CriticalPeriodModulator returns 2.0 decaying to 1.0 during critical period: matches code lines 277-278
- Post-critical-period floor of 0.5: matches code line 282
- Homeostatic regulation pulls extreme behavioral weights > 0.8: matches code line 215
- Model monopoly prevention at > 0.95: matches code line 234
- Homeostasis runs every 10 steps: matches code line 373
- `new_session()` resets critical period: matches code line 393-394

#### FEATURE_REQUEST
_(none)_

---

### reference/engine/adaptation.md -> corteX/engine/adaptation.py
**Status**: [OK] (0 gaps found)

#### WRONG_API
_(none)_

#### WRONG_ATTR
_(none)_

#### MISSING_API
_(none)_ -- All public methods documented. The `_change_history` internal attribute is appropriately omitted.

#### WRONG_SIGNATURE
_(none)_ -- All signatures verified:
- `RapidAdaptation.__init__(decay_rate=0.5, novelty_bonus=2.0)`: matches code line 69
- `RapidAdaptation.filter(signal_key, value)`: matches code line 74
- `RapidAdaptation.reset(signal_key=None)`: matches code line 139
- `SustainedAdaptation.__init__(habituation_threshold=8, recovery_time=300.0)`: matches code line 162
- `SustainedAdaptation.filter(signal_key, value)`: matches code line 167
- `SustainedAdaptation.is_habituated(signal_key)`: matches code line 266
- `SustainedAdaptation.get_baseline(signal_key)`: matches code line 271
- `SustainedAdaptation.reset(signal_key=None)`: matches code line 276
- `AdaptationFilter.__init__(rapid_decay=0.5, habituation_threshold=8, recovery_time=300.0)`: matches code lines 308-313
- `AdaptationFilter.process(signal_key, value)`: matches code line 321
- `AdaptationFilter.detect_behavioral_shift(signal_key, current_value, window=5)`: matches code line 362
- `AdaptationFilter.get_habituated_signals()`: matches code line 393
- `AdaptationFilter.get_active_signals()`: matches code line 400
- `AdaptationFilter.get_stats()`: matches code line 407
- `AdaptationFilter.new_session()`: matches code line 419

#### WRONG_EXAMPLE
_(none)_ -- Example uses real API correctly. The described sequence behavior (novel -> decaying -> habituated -> change bonus) matches code logic.

#### WRONG_DESCRIPTION
_(none)_ -- All descriptions verified:
- Rapid adaptation: novel signal gets `novelty_bonus` (2.0): matches code line 98
- Repeated same value (delta < 0.05): decay formula `decay_rate ^ (repetitions - 1)`: matches code line 112
- Value change resets adaptation, returns `novelty_bonus`: matches code lines 125-136
- Sustained: novel signal gets weight 1.0: matches code line 192
- Linear decay to 0.2 before habituation: `1.0 - progress * 0.8`: matches code line 233
- After threshold: None returned (habituated): matches code lines 222-229
- Change after steady state: weight 1.5: matches code line 262
- Dishabituation after recovery_time: matches code lines 197-203
- Behavioral shift threshold of 0.15: matches code line 379
- Shift weight = `1.5 + |deviation|`: matches code line 389
- `new_session()` preserves baselines: matches code lines 419-425
- `get_stats()` return keys: match code lines 409-416

#### FEATURE_REQUEST
_(none)_

---

## Full Gap Inventory

| # | Page | Category | Description | Severity |
|---|------|----------|-------------|----------|
| 1 | weights.md | MISSING_API | `ModelSelectionWeights.to_dict()` not documented | LOW |
| 2 | weights.md | MISSING_API | `UserInsightWeights.to_dict()` not documented | LOW |
| 3 | weights.md | MISSING_API | `EnterpriseWeights.to_dict()` not documented | LOW |
| 4 | weights.md | MISSING_API | `GlobalWeights.to_dict()` not documented | LOW |
| 5 | weights.md | WRONG_DESCRIPTION | UserInsightWeights tier numbering inconsistent (Tier 2 vs position 5/7) | LOW |
| 6 | goal-tracker.md | WRONG_DESCRIPTION | Recommended Actions table omits precedence detail (7 checks shown as 4 rows) | MEDIUM |
| 7 | goal-tracker.md | WRONG_DESCRIPTION | Missing note that `_step_times` grows unboundedly (code concern, not doc gap) | LOW |
| 8 | goal-tracker.md | WRONG_DESCRIPTION | `verify_step` timing details (`_step_times`, `_start_time`) not mentioned | LOW |
| 9 | feedback.md | MISSING_API | `SATISFACTION_PATTERNS` flat list (backward-compat) not documented | LOW |
| 10 | feedback.md | MISSING_API | `_TOPIC_INDICATORS` regex (private, acceptable omission) | LOW |
| 11 | feedback.md | WRONG_DESCRIPTION | Example comment says strength=0.35 but likely 0.7 for that input | MEDIUM |

**HIGH severity gaps: 0**
**MEDIUM severity gaps: 2**
**LOW severity gaps: 9**
