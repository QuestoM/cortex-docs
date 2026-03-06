# Batch 7 Audit: Tenancy + Observability (7 pages)

**Auditor**: Documentation Audit Agent (Batch 7)
**Date**: 2026-02-22
**Scope**: 3 Tenancy doc pages + 4 Observability doc pages vs. source code

---

## Summary

| # | Page | Source | Status | Gaps |
|---|------|--------|--------|------|
| 1 | reference/tenancy/manager.md | corteX/tenancy/manager.py | **[OK]** | 0 |
| 2 | reference/tenancy/dna.md | corteX/tenancy/dna.py | **[OK]** | 0 |
| 3 | reference/tenancy/quota.md | corteX/tenancy/quota.py | **[OK]** | 0 |
| 4 | reference/observability/tracer.md | corteX/observability/tracer.py | **[OK]** | 0 |
| 5 | reference/observability/cost-predictor.md | corteX/observability/cost_predictor.py | **[A]** | 1 |
| 6 | reference/observability/metrics.md | corteX/observability/metrics.py | **[OK]** | 0 |
| 7 | reference/observability/audit-stream.md | corteX/observability/audit_stream.py | **[OK]** | 0 |

**Total gaps found: 1**

---

## Detailed Findings

### 1. reference/tenancy/manager.md -> corteX/tenancy/manager.py
**Status**: [OK]

All classes, methods, attributes, signatures, and return types match the source code exactly.

**Verified items**:
- `TenantSummary` dataclass: All 4 attributes (`tenant_id`, `active_sessions`, `tokens_used_today`, `created_at`) match source (lines 20-33).
- `ErasureReceipt` dataclass: All 4 attributes (`tenant_id`, `erased_at`, `components_erased`, `success`) match source (lines 36-48).
- `TenantManager` class:
  - Constructor: No parameters -- matches source (line 65).
  - `tenant_count` property: Returns `int` -- matches source (lines 245-248).
  - `create_tenant(tenant_id, config, keys)` -> `Dict[str, Any]`: Matches source (lines 73-121). Raises `ValueError` per doc.
  - `get_tenant(tenant_id)` -> `Optional[Dict[str, Any]]`: Returns dict with documented keys -- matches source (lines 123-138).
  - `destroy_tenant(tenant_id)` -> `Dict[str, Any]`: Returns receipt dict with documented keys, erases 3 components -- matches source (lines 140-184).
  - `update_config(tenant_id, config)` -> `None`: Matches source (lines 186-204).
  - `list_tenants()` -> `List[TenantSummary]`: Matches source (lines 206-216).
  - `increment_sessions(tenant_id)` -> `None`: Matches source (lines 222-225).
  - `decrement_sessions(tenant_id)` -> `None`: Matches source (lines 227-231).
  - `add_tokens(tenant_id, count)` -> `None`: Matches source (lines 233-236).
  - `get_vault(tenant_id)` -> `Optional[KeyVault]`: Matches source (lines 238-243).
- Code example: Uses correct import path, method names, and return value keys. Valid.

---

### 2. reference/tenancy/dna.md -> corteX/tenancy/dna.py
**Status**: [OK]

All classes, methods, attributes, signatures, and behavior match the source code exactly.

**Verified items**:
- `TenantDNA` dataclass: All 8 attributes match source (lines 30-50):
  - `avg_task_complexity` (float, default 0.5)
  - `preferred_model_tier` (str, default "worker")
  - `avg_tokens_per_task` (int, default 2000)
  - `tool_usage_frequency` (Dict[str, float], default {})
  - `peak_hours` (List[int], default [])
  - `risk_profile` (float, default 0.3)
  - `common_intents` (List[str], default [])
  - `last_updated` (float, default time.time())
- `update_from_session(session_metrics, alpha=0.2)` -> `None`: Matches source (lines 52-142). Alpha clamped to [0.01, 1.0] per doc. All 7 update rules in the doc table match source behavior (EMA for complexity/tokens/risk, majority wins for model_tier, append+dedup for peak_hours/intents, EMA per tool with prune at 50).
- `to_dict()` -> `Dict[str, Any]`: Matches source (lines 144-157).
- `from_dict(data)` -> `TenantDNA` (classmethod): Matches source (lines 159-171).
- `TenantDNAStore` class:
  - Constructor: No parameters -- matches source (lines 190-192).
  - `count` property: Returns `int` -- matches source (lines 223-226).
  - `get(tenant_id)` -> `TenantDNA`: Returns default if missing -- matches source (lines 194-204).
  - `save(tenant_id, dna)` -> `None`: Matches source (lines 206-209).
  - `delete(tenant_id)` -> `None`: Matches source (lines 211-217).
  - `list_tenants()` -> `List[str]`: Matches source (lines 219-221).
- EMA formula in doc matches `_ema()` static method (line 174-176).
- Code example: Uses correct import path, method names, and data structures. Valid.

---

### 3. reference/tenancy/quota.md -> corteX/tenancy/quota.py
**Status**: [OK]

All classes, methods, attributes, signatures, and behavior match the source code exactly.

**Verified items**:
- `QuotaDecision` enum (str, Enum): All 3 values (`ALLOW`, `SOFT_LIMIT`, `HARD_LIMIT`) match source (lines 23-27).
- `QuotaEnvelope` dataclass: All 6 attributes match source (lines 31-38):
  - `tokens_per_day` (int, default 1_000_000)
  - `tokens_per_session` (int, default 100_000)
  - `requests_per_minute` (int, default 60)
  - `requests_per_day` (int, default 10_000)
  - `concurrent_sessions` (int, default 50)
  - `max_tool_calls_per_turn` (int, default 10)
- `QuotaTracker` class:
  - Constructor `QuotaTracker(envelope)`: Matches source (lines 44-58).
  - `acquire_tokens(estimated, session_id="")` -> `QuotaDecision`: Matches source (lines 60-79). Decision logic matches doc table (daily HARD_LIMIT, session HARD_LIMIT, 80% SOFT_LIMIT, ALLOW).
  - `acquire_request()` -> `QuotaDecision`: Matches source (lines 81-101). Uses 60s sliding window. Decision logic matches doc table.
  - `acquire_session()` -> `QuotaDecision`: Matches source (lines 103-113). Decision logic matches doc table.
  - `release_session(session_id="")` -> `None`: Matches source (lines 115-120). Clears per-session tracking as documented.
  - `get_usage()` -> `Dict[str, Any]`: All 9 keys in doc table match source (lines 122-138).
  - `reset_daily()` -> `None`: Matches source (lines 140-147). Resets tokens, requests, per-session tracking.
- Auto-reset behavior: `_maybe_reset_day()` checks UTC day change on acquire_tokens/acquire_request -- matches doc.
- 80% soft limit threshold: `_SOFT_LIMIT_FRACTION = 0.8` at line 20 -- matches doc.
- Thread safety: Uses `threading.Lock` per doc -- confirmed at line 46.
- Code example: Uses correct import path, method names, and enum comparison. Valid.

---

### 4. reference/observability/tracer.md -> corteX/observability/tracer.py
**Status**: [OK]

All classes, methods, attributes, signatures, and behavior match the source code exactly.

**Verified items**:
- `DecisionTrace` dataclass: All 13 attributes match source (lines 21-35):
  - `trace_id` (str, auto-generated 16-char hex) -- uses `uuid.uuid4().hex[:16]`
  - `parent_id` (Optional[str])
  - `tenant_id` (str)
  - `step_type` (str)
  - `decision` (str)
  - `alternatives` (List[str])
  - `reasoning` (str)
  - `confidence` (float)
  - `latency_ms` (float)
  - `tokens_consumed` (int)
  - `brain_state` (Dict[str, float])
  - `timestamp` (float)
  - `metadata` (Dict[str, Any])
- `to_dict()` -> `Dict[str, Any]`: Matches source (lines 37-39).
- `to_json()` -> `str`: Matches source (lines 41-43).
- `DecisionTracer` class:
  - Constructor `DecisionTracer(tenant_id="", max_traces=1000)`: Matches source (lines 54-66). Min 50 traces per doc -- confirmed at line 60.
  - `tenant_id` property: Matches source (lines 68-70).
  - `trace_model_selection(routing_decision, brain_snapshot, latency_ms=0.0, tokens_consumed=0)` -> `DecisionTrace`: Matches source (lines 97-134). routing_decision keys match.
  - `trace_tool_selection(tool, confidence, alternatives, latency_ms=0.0, reasoning="")` -> `DecisionTrace`: Matches source (lines 136-169). alternatives type is `List[Tuple[str, float]]`.
  - `trace_plan_step(step, risk, reasoning, latency_ms=0.0, tokens_consumed=0, brain_state=None)` -> `DecisionTrace`: Matches source (lines 171-205). Confidence computed as `1.0 - risk` per doc -- confirmed at line 199.
  - `push_parent(trace_id)` -> `None`: Matches source (lines 83-85).
  - `pop_parent()` -> `Optional[str]`: Matches source (lines 87-91).
  - `get_traces(last_n=20)` -> `List[DecisionTrace]`: Matches source (lines 207-216).
  - `get_by_type(step_type)` -> `List[DecisionTrace]`: Matches source (lines 218-227).
  - `get_by_parent(parent_id)` -> `List[DecisionTrace]`: Matches source (lines 229-238).
  - `export_json()` -> `str`: Matches source (lines 240-250).
  - `get_stats()` -> `Dict[str, Any]`: All 6 keys match source (lines 252-274).
  - `clear()` -> `int`: Returns count -- matches source (lines 276-281).
- Code example: Uses correct import path, method names, nested tracing, and query patterns. Valid.

---

### 5. reference/observability/cost-predictor.md -> corteX/observability/cost_predictor.py
**Status**: [A] (1 gap found)

#### WRONG_DESCRIPTION
**Location**: Doc lines 98-102 (Token estimation heuristics - StepEstimate.confidence default)

**Issue**: The `StepEstimate` dataclass has a `confidence` attribute documented in the attributes table with no explicit default mentioned (just described as "Confidence in this estimate (0.0-1.0)"). However, in the source code (line 43), the default value is `0.8`:

```python
confidence: float = 0.8
```

This is not explicitly wrong in the doc (it doesn't state a different default), but it is a minor omission. The doc's main text section correctly describes how confidence is computed dynamically by `_step_confidence()` (line 233-255 in source), which returns 0.95 for explicit tokens, 0.75 for 10+ word descriptions, 0.65 for 5+ word descriptions, and 0.5 otherwise -- the 0.8 default in the dataclass is overridden by the CostPredictor logic.

**Severity**: Low (informational)

**Everything else verified and correct**:
- `StepEstimate` dataclass: All 6 attributes match source (lines 37-44).
- `CostPrediction` dataclass: All 7 attributes match source (lines 48-56). `summary()` method matches (lines 58-67).
- `CostPredictor` class:
  - Constructor `CostPredictor()`: Matches source (lines 78-81).
  - `predict(plan_steps, tenant_dna=None, model_pricing=None, budget=inf)` -> `CostPrediction`: Matches source (lines 83-164). All parameters documented correctly.
  - Token estimation heuristics: Simple keywords (`_SIMPLE_KEYWORDS` line 25: lookup, get, read, check, list, count, echo) match doc. Complex keywords (`_COMPLEX_KEYWORDS` lines 26-29: analyze, generate, write, create, plan, summarize, compare, refactor, debug, transform, synthesize) match doc. Base tokens: 500/1500/4000 match source (_DEFAULT_TOKENS_SIMPLE/MEDIUM/COMPLEX at lines 19-21).
  - 60% heuristic + 40% tenant DNA blending: Matches source (line 195).
  - Confidence interval multipliers: No DNA = 0.6x/1.8x (source lines 32-33); With DNA = 0.7x/1.5x (source lines 143-145) -- matches doc.
  - Price lookup: Exact match -> prefix match -> $0.002 default. Matches source (lines 202-231).
  - `is_within_budget(prediction, budget)` -> `bool`: Matches source (lines 263-277).
  - `get_calibration_stats()` -> `Dict[str, Any]`: Returns `history_size` and `avg_estimated_tokens` -- matches source (lines 279-291).
- Code example: Uses correct import path, method names, and data structures. Valid.

---

### 6. reference/observability/metrics.md -> corteX/observability/metrics.py
**Status**: [OK]

All classes, methods, attributes, signatures, and behavior match the source code exactly.

**Verified items**:
- `MetricEntry` dataclass: All 6 attributes match source (lines 24-31):
  - `timestamp` (float, default time.time())
  - `tenant_id` (str, default "")
  - `operation` (str, default "")
  - `value` (float, default 0.0)
  - `unit` (str, default "")
  - `metadata` (Dict[str, Any], default {})
- `DashboardData` dataclass: All 9 attributes match source (lines 35-45):
  - `total_requests` (int), `avg_latency_ms` (float), `error_rate` (float), `tokens_total` (int), `cost_total_usd` (float), `top_operations` (List[Tuple[str, int]]), `top_models` (List[Tuple[str, int]]), `drift_avg` (float), `period_seconds` (float)
  - `to_dict()` and `to_json()` methods: Match source (lines 47-51).
- `MetricsCollector` class:
  - Constructor `MetricsCollector(window_size=10_000)`: Matches source (lines 61-68). Min 100 per doc -- confirmed at line 62.
  - `record_latency(tenant_id, operation, ms)` -> `None`: Matches source (lines 70-74).
  - `record_tokens(tenant_id, model, count)` -> `None`: Matches source (lines 76-79).
  - `record_success(tenant_id, operation, success)` -> `None`: Matches source (lines 81-85).
  - `record_drift(tenant_id, score)` -> `None`: Matches source (lines 87-90).
  - `record_cost(tenant_id, model, cost_usd)` -> `None`: Matches source (lines 92-95).
  - `get_dashboard(tenant_id=None)` -> `DashboardData`: Matches source (lines 105-134). None = process-wide per doc.
  - `export_prometheus()` -> `str`: Matches source (lines 136-176). All 4 Prometheus metrics (`cortex_latency_ms`, `cortex_tokens_total`, `cortex_error_rate`, `cortex_cost_usd`) with correct types (gauge/counter) match source. Prometheus output format matches the example in the doc.
  - `export_opentelemetry()` -> `Dict[str, Any]`: Matches source (lines 178-197). `service.name`="cortex-agent", `service.version`="1.0.0" -- confirmed. Span structure with `operationName`, `startTime`, `duration`, `tags`, `attributes` matches source.
  - `get_stats()` -> `Dict[str, Any]`: All 6 keys match source (lines 199-208).
  - `clear()` -> `None`: Matches source (lines 210-216).
- Memory management: Bounded `deque` with `maxlen=window_size`, 5 types = 50K max at default -- matches doc.
- Code example: Uses correct import path, method names, and data access patterns. Valid.

---

### 7. reference/observability/audit-stream.md -> corteX/observability/audit_stream.py
**Status**: [OK]

All classes, methods, attributes, signatures, and behavior match the source code exactly.

**Verified items**:
- `AuditEventType` enum (str, Enum): All 8 values match source (lines 24-33): `LLM_CALL`, `TOOL_EXECUTION`, `POLICY_DECISION`, `GOAL_EVENT`, `SESSION_START`, `SESSION_END`, `DATA_ACCESS`, `CONFIG_CHANGE`.
- `EnterpriseAuditEntry` dataclass: All 13 attributes match source (lines 37-52):
  - `event_id` (str, auto-generated 16-char hex via uuid.uuid4().hex[:16])
  - `tenant_id` (str), `session_id` (str), `user_id` (str), `timestamp` (float), `event_type` (AuditEventType), `action` (str), `justification` (str), `outcome` (str), `cost_tokens` (int), `data_level` (str), `capabilities_used` (List[str]), `chain_hash` (str), `metadata` (Dict[str, Any])
  - `to_dict()`: Serializes with `event_type` as string -- matches source (lines 54-57).
  - `to_json()`: Matches source (lines 59-60).
  - `hashable_content()`: Excludes `chain_hash` -- matches source (lines 62-70).
- `TenantAuditStream` class:
  - Constructor `TenantAuditStream(tenant_id, max_entries=50_000)`: Matches source (lines 80-85). Min 100 per doc -- confirmed at line 82.
  - `tenant_id` property: Matches source (lines 87-89).
  - `length` property: Matches source (lines 91-93).
  - `append(entry)` -> `EnterpriseAuditEntry`: Sets `tenant_id`, computes `chain_hash` -- matches source (lines 95-106). Hash computed as `SHA-256(previous_hash + ":" + hashable_content)` -- matches `_compute_hash()` at lines 174-178.
  - `query(filters)` -> `List[EnterpriseAuditEntry]`: All 8 filter keys documented (`event_type`, `session_id`, `user_id`, `start_time`, `end_time`, `action`, `data_level`, `limit`) match source (lines 108-136). Default limit=100.
  - `verify_chain()` -> `Tuple[bool, Optional[int]]`: Returns `(True, None)` or `(False, break_index)` -- matches source (lines 138-152).
  - `export(format="jsonl")` -> `List[str]`: Supports "jsonl" and "csv" -- matches source (lines 154-158).
  - `get_stats()` -> `Dict[str, Any]`: All 5 keys (`tenant_id`, `total_entries`, `max_entries`, `entries_by_type`, `last_hash`) match source (lines 180-192). `last_hash` is truncated per doc -- confirmed at line 191.
  - `clear()` -> `int`: Resets chain to genesis hash, returns count -- matches source (lines 194-199).
- Hash chain design: Genesis hash is `"0" * 64` (source line 21). Chain formula in doc matches `_compute_hash()` exactly.
- Code example: Uses correct import path, class names, enum values, and API patterns. Valid.

---

## Cross-Cutting Observations

### Missing Documentation for `corteX/tenancy/context.py`

The `corteX/tenancy/context.py` module defines three significant public classes (`TenantContext`, `TenantLayer`, `NoTenantBoundError`) that are exported from `corteX/tenancy/__init__.py`. There is no corresponding documentation page in `reference/tenancy/`. This is NOT a gap in any of the 7 audited pages (none of them claim to document context.py), but it is a **documentation coverage gap** for the tenancy package overall.

**Classes without doc pages**:
- `TenantContext` (frozen dataclass with `bind()`, `current()`, `reset()`, `for_end_user()` methods)
- `TenantLayer` (str, Enum with QUESTO, DEVELOPER, END_USER values)
- `NoTenantBoundError` (RuntimeError subclass)

---

## Final Statistics

- **Pages audited**: 7
- **Pages with zero gaps**: 6 (manager.md, dna.md, quota.md, tracer.md, metrics.md, audit-stream.md)
- **Pages with gaps**: 1 (cost-predictor.md -- 1 minor WRONG_DESCRIPTION)
- **Total gaps found**: 1
- **Gap severity**: 1 Low

This is an exceptionally well-documented batch. The documentation accurately reflects the source code across all 7 modules with only one minor informational gap.
