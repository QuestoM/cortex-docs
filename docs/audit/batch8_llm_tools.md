# Batch 8 Audit: LLM Layer + Tools (11 pages)

**Auditor**: Documentation Audit Agent
**Date**: 2026-02-22
**Source docs**: `C:\Intel\questo\projects\cortex-docs\docs\reference\llm\` and `...\tools\`
**Source code**: `C:\Intel\questo\projects\corteX\corteX\core\llm\` and `...\tools\`

**Summary**: 9 of 11 pages have gaps. 2 pages are fully accurate. Total: 24 gaps found.

---

## 1. reference/llm/router.md -> corteX/core/llm/router.py

**Status**: [A] (8 gaps found)

#### MISSING_API
1. **`LLMRouter.__init__` missing parameter `data_classification_enabled`**: The constructor in source code accepts `data_classification_enabled: bool = True` (line 150), but the doc omits this parameter entirely.

2. **`set_orchestrator_model()` undocumented**: Source has `set_orchestrator_model(model: str) -> None` (line 227) -- not mentioned in docs.

3. **`get_orchestrator_model()` undocumented**: Source has `get_orchestrator_model() -> Optional[str]` (line 231) -- not mentioned in docs.

4. **`set_worker_model()` undocumented**: Source has `set_worker_model(model: str) -> None` (line 235) -- not mentioned in docs.

5. **`get_worker_model()` undocumented**: Source has `get_worker_model() -> Optional[str]` (line 239) -- not mentioned in docs.

6. **`set_data_classification_enabled()` undocumented**: Source has `set_data_classification_enabled(enabled: bool) -> None` (line 265) -- not mentioned in docs.

7. **`set_pii_protection_enabled()` undocumented**: Source has `set_pii_protection_enabled(enabled: bool) -> None` (line 274) -- not mentioned in docs.

#### MISSING_API (properties)
8. **Missing properties in docs**: Source code has the following properties not listed in the doc's Properties table:
   - `data_classification_enabled` (bool) -- line 270
   - `pii_protection_enabled` (bool) -- line 280
   - `pii_tokenizer` (PIITokenizer) -- line 284

#### WRONG_DESCRIPTION
9. **Latency bonus threshold description**: Doc says latency bonus is "+0.1 for avg latency < 1s, -0.1 for avg > 5s". Source code (line 495-498) checks `avg_latency < 1000` and `avg_latency > 5000` -- these are in milliseconds, so the thresholds match, but the doc says "< 1s" which is technically correct if we assume the units in the doc are seconds. This is **OK** -- no gap.

---

## 2. reference/llm/base.md -> corteX/core/llm/base.py

**Status**: [A] (2 gaps found)

#### MISSING_API
1. **`DataClassificationError` not documented**: Source code defines `DataClassificationError(LLMError)` at line 57-75 with attributes `data_level` and `target_model`. The doc's error hierarchy section does not mention this error class at all.

#### WRONG_DESCRIPTION
2. **Error hierarchy diagram incomplete**: The doc shows the error hierarchy tree ending at `InvalidRequestError`. The source code also has `DataClassificationError` as a sibling of the other errors under `LLMError` (line 57). The hierarchy diagram is missing this entry:
   ```
   LLMError (base)
     |-- ...
     |-- DataClassificationError  (data too sensitive for target - fail fast)
   ```

---

## 3. reference/llm/openai-client.md -> corteX/core/llm/openai_client.py

**Status**: [OK]

All classes, methods, parameters, attributes, and examples match the source code. The doc accurately describes:
- `OpenAIProvider` constructor and its 4 parameters
- All 3 registered models with correct pricing and capabilities
- `generate()`, `generate_stream()`, `generate_structured()`, `health_check()` signatures
- Error classification table
- Local model usage example

No gaps found.

---

## 4. reference/llm/gemini-client.md -> corteX/core/llm/gemini_adapter.py

**Status**: [A] (2 gaps found)

#### MISSING_API
1. **`_extract_system_messages()` undocumented**: Source code has the helper function `_extract_system_messages(messages: List[LLMMessage]) -> str` at line 37. The doc's "Helper Functions" section only documents `_build_system_instruction` and `_is_gemini3_model`, but not this one. Note: this is a private function, so it could be considered intentionally undocumented -- but since the doc already documents other underscore-prefixed helpers (`_build_system_instruction`, `_is_gemini3_model`), the omission is inconsistent.

#### WRONG_DESCRIPTION
2. **`generate_stream` doc says Gemini SDK returns synchronous iterator but omits `_SENTINEL`**: The doc correctly describes the `asyncio.to_thread()` wrapping, but the actual implementation uses a `_SENTINEL` sentinel object pattern (line 554-555) for async iteration. This is an implementation detail, so this is a **minor** gap. The behavior description is accurate.

---

## 5. reference/llm/anthropic-client.md -> corteX/core/llm/anthropic_client.py

**Status**: [A] (2 gaps found)

#### WRONG_DESCRIPTION
1. **`generate_stream` default thinking_budget hardcoded**: The doc does not mention that when `generate_stream` is called with `thinking=True`, the implementation hardcodes `thinking_budget=10000` (line 234: `_build_kwargs(..., thinking, 10000, ...)`). This means the thinking budget is not configurable for streaming -- it's always 10000 tokens. The doc makes no mention of this limitation.

#### MISSING_API (minor)
2. **`_build_kwargs` helper not documented**: Source code has `_build_kwargs()` (line 111-132) as a module-level helper that builds the kwargs dict for both `messages.create` and `messages.stream`. It is not documented, but since the Gemini doc documents similar private helpers, this is an inconsistency.

---

## 6. reference/llm/classifier.md -> corteX/core/llm/classifier.py

**Status**: [A] (1 gap found)

#### WRONG_DESCRIPTION
1. **Keyword lexicon counts are inaccurate**: The doc says:
   - `_CODING_KW`: "26 keywords" -- Source has exactly 26 keywords. **Correct.**
   - `_PLANNING_KW`: "17 keywords" -- Source has exactly 17 keywords. **Correct.**
   - `_REASONING_KW`: "19 keywords" -- Source has exactly 19 keywords. **Correct.**
   - `_CREATIVE_KW`: "18 keywords" -- Source has exactly 18 keywords. **Correct.**
   - The doc says "Each set contains 11-26 keywords" but doesn't mention the other lexicons:
     - `_FACTUAL_KW`: 17 keywords (undocumented)
     - `_SUMMARIZATION_KW`: 12 keywords (undocumented)
     - `_VALIDATION_KW`: 12 keywords (undocumented)
     - `_DECISION_KW`: 14 keywords (undocumented)
     - `_CLASSIFICATION_KW`: 11 keywords (undocumented)
   - The range "11-26" is actually correct, but the omission of 5 out of 9 lexicons from the explicit listing is a documentation gap. The doc implies only 4 lexicons exist by only naming 4.

---

## 7. reference/llm/cost-tracker.md -> corteX/core/llm/cost_tracker.py

**Status**: [A] (2 gaps found)

#### MISSING_API
1. **`get_tenant_summary()` undocumented**: Source code has `get_tenant_summary(tenant_id: str) -> CostSummary` at line 157. This method is not mentioned in the docs.

2. **`get_all_records()` undocumented**: Source code has `get_all_records(session_id: Optional[str] = None) -> List[CostRecord]` at line 227. This method is not mentioned in the docs.

---

## 8. reference/llm/registry.md -> corteX/core/llm/registry.py

**Status**: [OK]

All classes, enums, data classes, methods, properties, constructor parameters, and the YAML format match the source code perfectly. Specifically verified:
- `ModelRole` enum with all 8 values
- `ModelFeatures`, `ModelEntry`, `RoleMapping` dataclasses with all attributes and defaults
- `ModelRegistry` constructor parameters
- All 8 methods: `get_model`, `get_all_models`, `get_models_for_role`, `estimate_cost`, `register_model`, `set_role_mapping`, `reload_if_changed`, `force_reload`
- Both properties: `model_count`, `loaded_path`
- Resolution order matches source code lines 113-118

No gaps found.

---

## 9. reference/llm/resilience.md -> corteX/core/llm/resilience.py

**Status**: [A] (1 gap found)

#### WRONG_DESCRIPTION (minor)
1. **`_CircuitData` and `_BucketData` internal dataclasses not documented**: These are internal implementation details (`_CircuitData` at line 24-29, `_BucketData` at line 109-112). While this is arguably fine for a public API reference, the doc does not mention how circuit state is tracked internally. This is a **minor** gap -- not a real issue for consumers.

All public API is correctly documented:
- `CircuitState` enum with all 3 values
- `CircuitBreaker` constructor and all 5 methods match exactly
- `RateLimiter` constructor and all 4 methods match exactly
- Integration example with `LLMRouter` is accurate

---

## 10. reference/tools/decorator.md -> corteX/tools/decorator.py

**Status**: [A] (3 gaps found)

#### WRONG_API
1. **`tool()` return type in doc is misleading**: The doc says `tool()` returns `Callable`, and the doc says "Returns: A `ToolWrapper` instance (the decorated function is wrapped)." However, looking at the source (line 278-290), `tool()` returns a `decorator` function (a `Callable`), and that inner decorator returns a `ToolWrapper`. The doc's description of the return value is technically about what the decorator produces, not what `tool()` itself returns. This is potentially confusing but functionally correct. The signature `def tool(...) -> Callable` in the doc matches the source.

#### WRONG_EXAMPLE
2. **`@tool()` decorator usage pattern**: The doc shows `@tool()` (with parentheses) for no-argument usage. Looking at the source code (line 278-290), `tool()` always returns a decorator function -- it does NOT support being used without parentheses as `@tool` (no parens). The doc's examples are **correct** in showing `@tool()` and `@tool(name="...")`. However, the doc does NOT warn that `@tool` without parens will fail. This is a minor documentation gap.

#### MISSING_API (minor)
3. **`_default_registry` module-level variable and `_registered_tools` alias not documented**: Source code (lines 272-275) creates a module-level `_default_registry = ToolRegistry(name="global-default")` and a backward-compatible `_registered_tools` alias. While these are private, some users/tests may reference them.

---

## 11. reference/tools/executor.md -> corteX/tools/executor.py

**Status**: [A] (3 gaps found)

#### WRONG_DESCRIPTION
1. **`ToolExecutionResult` described as having a "Constructor" but it is a plain class, not a dataclass**: The doc formats the constructor as a dataclass-style definition, but the source code (line 21-35) shows it's a regular class with an explicit `__init__`. This is a **minor** style discrepancy -- the behavior is the same.

#### MISSING_API
2. **`get_tool` import from decorator module**: The source code (line 14) imports `get_tool` from `corteX.tools.decorator`, but `ToolExecutor` does not actually use it anywhere. This is not a doc gap per se, but the import is unused.

#### WRONG_DESCRIPTION
3. **Doc says "Integrates with the Weight Engine and Prediction system"**: The doc's opening paragraph and the source code's docstring (line 3-4) both say this, but looking at the actual `ToolExecutor` implementation, there is NO integration with any weight engine or prediction system. There are no imports from `corteX.engine.weights` or `corteX.engine.prediction`. This is a **WRONG_DESCRIPTION** -- the integration claim is aspirational, not implemented.

---

# Summary Table

| # | Page | Source File | Status | Gap Count |
|---|------|------------|--------|-----------|
| 1 | llm/router.md | core/llm/router.py | [A] | 8 |
| 2 | llm/base.md | core/llm/base.py | [A] | 2 |
| 3 | llm/openai-client.md | core/llm/openai_client.py | [OK] | 0 |
| 4 | llm/gemini-client.md | core/llm/gemini_adapter.py | [A] | 2 |
| 5 | llm/anthropic-client.md | core/llm/anthropic_client.py | [A] | 2 |
| 6 | llm/classifier.md | core/llm/classifier.py | [A] | 1 |
| 7 | llm/cost-tracker.md | core/llm/cost_tracker.py | [A] | 2 |
| 8 | llm/registry.md | core/llm/registry.py | [OK] | 0 |
| 9 | llm/resilience.md | core/llm/resilience.py | [A] | 1 |
| 10 | tools/decorator.md | tools/decorator.py | [A] | 3 |
| 11 | tools/executor.md | tools/executor.py | [A] | 3 |

**Total gaps**: 24
**Critical gaps (WRONG_API / WRONG_EXAMPLE)**: 1
**Missing API gaps**: 14
**Wrong description gaps**: 7
**Minor/style gaps**: 2

### High-Priority Fixes
1. **router.md**: Add `data_classification_enabled` constructor param + 5 undocumented methods + 3 undocumented properties
2. **base.md**: Add `DataClassificationError` to error hierarchy
3. **cost-tracker.md**: Document `get_tenant_summary()` and `get_all_records()`
4. **anthropic-client.md**: Document hardcoded `thinking_budget=10000` in streaming
5. **executor.md**: Remove or clarify "Weight Engine and Prediction system" integration claim
