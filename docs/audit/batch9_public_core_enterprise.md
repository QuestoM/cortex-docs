# Batch 9 Audit: Public API + Core + Enterprise (16 pages)

**Auditor**: Claude Opus 4.6
**Date**: 2026-02-22
**Scope**: 4 Public API pages, 3 Core pages, 9 Enterprise pages

---

## PUBLIC API (4 pages)

---

### 1. reference/engine.md -> corteX/sdk.py (Engine class)
**Status**: [A] (gaps found)

#### WRONG_SIGNATURE
- **`create_agent` missing `mcp_servers` and `a2a_agents` parameters**: The doc shows `create_agent()` ending at `enable_session_recording`, but the actual code (sdk.py:145-157) also accepts `mcp_servers: Optional[List] = None` and `a2a_agents: Optional[List] = None`. These parameters are missing from both the signature block and the parameters table.

#### WRONG_EXAMPLE
- **Enterprise config example uses `EnterpriseConfig(compliance=["SOC2", "GDPR"])`**: In the doc (line 82-89), the example passes `compliance=["SOC2", "GDPR"]` as a list of strings. Looking at `sdk_config.py:64`, `compliance` is indeed `List[str]`, so this is technically correct. However, the enterprise config module (`enterprise/config.py`) uses `ComplianceFramework` enum values. **Not technically wrong, but could be confusing** since the enterprise-level TenantConfig expects enum values while the simple EnterpriseConfig accepts strings.

---

### 2. reference/agent.md -> corteX/sdk.py (Agent class)
**Status**: [A] (gaps found)

#### WRONG_SIGNATURE
- **Agent constructor missing `mcp_servers` and `a2a_agents` parameters**: The doc shows the constructor ending at `enable_session_recording` (line 29). The actual code (sdk.py:45-58) also accepts:
  - `mcp_servers: Optional[List] = None`
  - `a2a_agents: Optional[List] = None`
  These parameters are missing from both the constructor signature and the parameters table.

#### MISSING_API
- **Missing `mcp_servers` and `a2a_agents` attributes**: The attributes table (lines 54-65) does not list `mcp_servers` or `a2a_agents`, but these are set as instance attributes in `__init__` (sdk.py:70-71).

---

### 3. reference/session.md -> corteX/session/ (Session class)
**Status**: [A] (gaps found)

#### MISSING_API
- **`get_proactive_stats()`**: Exists in `session/stats.py:47-49` but not documented.
- **`get_cross_modal_stats()`**: Exists in `session/stats.py:51-53` but not documented.
- **`get_resource_map()`**: Exists in `session/stats.py:63-65` but not documented.
- **`get_attention_stats()`**: Exists in `session/stats.py:67-69` but not documented.
- **`get_territory_map()`**: Exists in `session/stats.py:79-81` but not documented.
- **`get_active_modulations()`**: Exists in `session/stats.py:83-85` but not documented.
- **`get_decision_log()`**: Exists in `session/stats.py:95-97` but not documented.
- **`get_goal_tree()`**: Exists in `session/stats.py:115-117` but not documented.
- **`get_nash_routing_stats()`**: Exists in `session/stats.py:119-121` but not documented.
- **`get_shapley_stats()`**: Exists in `session/stats.py:123-125` but not documented.
- **`get_summarization_stats()`**: Exists in `session/stats.py:127-129` but not documented.
- **`get_simulator()`**: Exists in `session/stats.py:131-133` but not documented.
- **`gdpr` property**: Exists in `session/core.py:205-218`, lazy-initializes `GDPRManager`, not documented.
- **Explainability mixin methods**: `SessionExplainabilityMixin` is included but none of its methods are documented.
- **Interop mixin methods**: `SessionInteropMixin` is included but none of its methods are documented.

#### WRONG_ATTR
- **Session `memory` type**: Doc says `MemoryFabric` -- code confirms `MemoryFabric` (core.py:89). **OK**.
- **Session `context_engine` type**: Doc says `CorticalContextEngine` -- code confirms (core.py:135). **OK**.
- **Missing attributes**: The doc does not list:
  - `proactive` (`ProactivePredictionEngine`)
  - `cross_modal` (`ContextEnricher`)
  - `resource_map` (`ResourceHomunculus`)
  - `reorganizer` (`CorticalMapReorganizer`)
  - `modulator` (`TargetedModulator`)
  - `simulator` (`ComponentSimulator`)
  - `adaptation` (`AdaptationFilter`)
  - `quality_estimator` (`PopulationQualityEstimator`)

  However, most of these are listed in the "Brain Components" table at the bottom. The issue is specifically in the "Attributes" table (lines 42-59), which is incomplete -- it lists 14 attributes but the actual Session has 20+ public brain component attributes.

#### WRONG_DESCRIPTION
- **`close()` return dict**: The doc (lines 391-400) lists return keys like `"memory_stats"`, `"context_stats"`, `"calibration_stats"`. The actual code (stats.py:174-238) returns these AND 30+ more stat keys (`"dual_process_stats"`, `"reputation_stats"`, `"proactive_stats"`, `"cross_modal_stats"`, `"column_stats"`, `"resource_map_stats"`, `"attention_stats"`, `"concept_graph_stats"`, `"reorganizer_stats"`, `"modulator_stats"`, `"simulator_stats"`, `"feedback_stats"`, `"goal_tracker_stats"`, `"plasticity_stats"`, `"prediction_stats"`, `"adaptation_stats"`, `"quality_estimator_stats"`, `"param_resolver_stats"`, `"nash_routing_stats"`, `"shapley_attribution_stats"`, `"summarization_stats"`, `"semantic_scorer_stats"`, `"context_compiler_stats"`, `"planner_stats"`, `"reflector_stats"`, `"recovery_stats"`, `"interaction_stats"`, `"policy_stats"`, `"sub_agent_stats"`, `"decision_log_stats"`, `"progress_estimator_stats"`, `"speculative_executor_stats"`, `"model_mosaic_stats"`, `"provider_health_stats"`, `"decision_tracer_stats"`, `"goal_tree_stats"`, `"active_forgetting_stats"`, `"tenant_dna_stats"`, `"quota_tracker_stats"`, `"audit_stream_stats"`, `"metrics_collector_stats"`). The doc's `...` notation is fair but the listed subset is misleadingly small.

---

### 4. reference/response.md -> corteX/sdk_config.py
**Status**: [OK] (matches)

All classes, attributes, types, and defaults match the source code exactly:

- **`Response`** (`@dataclass`): `content`, `metadata`, `artifacts`, `raw_response` -- all match sdk_config.py:82-88.
- **`ResponseMetadata`** (`@dataclass`): All 9 fields match sdk_config.py:68-78 exactly (types, defaults).
- **`WeightConfig`** (`pydantic.BaseModel`): All 12 fields match sdk_config.py:17-30 exactly.
- **`ContextManagementConfig`** (`pydantic.BaseModel`): All 8 fields match sdk_config.py:33-42 exactly.
- **`EnterpriseConfig`** (`pydantic.BaseModel`): All 8 fields match sdk_config.py:55-64 exactly.

No discrepancies found.

---

## CORE (3 pages)

---

### 5. reference/core/events.md -> corteX/core/events.py
**Status**: [OK] (matches)

All classes, enums, methods, and signatures match exactly:

- **`EventType`** enum: 6 values, all match events.py:22-28.
- **`Event`** model: 5 fields, all match events.py:31-36.
- **`EventHandler`** protocol: Matches events.py:39-40.
- **`EventBus`** class:
  - Constructor `(*, name: Optional[str] = None)` -- matches events.py:56.
  - Instance methods: `subscribe_instance`, `unsubscribe_instance`, `publish_instance`, `subscriber_count`, `clear_subscribers`, `name` property -- all match.
  - Class methods: `subscribe`, `publish`, `get_default` -- all match.
- **Module-level `bus`**: Documented as `EventBus(name="global-default")` -- matches events.py:134-137.

No discrepancies found.

---

### 6. reference/core/contracts.md -> corteX/core/contracts.py
**Status**: [OK] (matches)

All enums, models, and protocols match the source code:

- **`AgentRole`**: 4 values match contracts.py:10-14.
- **`ArtifactType`**: 8 values match contracts.py:16-24.
- **`AutonomyLevel`**: 3 values match contracts.py:26-29.
- **`Artifact`**: 5 fields match contracts.py:35-41.
- **`ContextSlice`**: 7 fields match contracts.py:43-52.
- **`IPlugin`**: 3 members match contracts.py:60-66.
- **`ILLMProvider`**: 2 methods match contracts.py:68-74.
- **`IMemoryDriver`**: 2 methods match contracts.py:76-82.
- **`ISubsystem`**: `execute(intent, context) -> Artifact` matches contracts.py:84-95.
- **`IAgent`**: `role` + `step(input_signal, context) -> Artifact` matches contracts.py:97-102.
- **`ILogInterceptor`**: `on_tool_output()` matches contracts.py:104-107.

No discrepancies found.

---

### 7. reference/core/registry.md -> corteX/core/registry.py
**Status**: [OK] (matches)

All constructors, methods, and behaviors match:

- **Constructor**: `(*, allowed_plugin_roots, tenant_id)` matches registry.py:29-33.
- **Instance methods**: `register_plugin`, `load_from_directory`, `get_agent`, `get_subsystem`, `get_all_subsystems`, `instantiate_subsystem`, `import_from`, `clear` -- all match.
- **Class methods**: `register`, `get_default` -- all match.
- **Security model**: Path allowlist, module isolation with `_cortex_plugin_{tenant_id}`, audit logging, blocked loads -- all match the code.
- **Module-level default**: `_default_registry = PluginRegistry(tenant_id="global-default")` -- matches registry.py:210.

No discrepancies found.

---

## ENTERPRISE (9 pages)

---

### 8. reference/enterprise/config.md -> corteX/enterprise/config.py
**Status**: [A] (gaps found)

#### MISSING_API
- **`TenantConfig.data_classification_enabled`**: Attribute exists in code (config.py:400) with default `True`, not documented in the doc's TenantConfig attributes table.
- **`TenantConfig.pii_protection_enabled`**: Attribute exists in code (config.py:401) with default `False`, not documented.
- **`LicenseConfig.allowed_features`**: Attribute exists in code (config.py:348-362), a Dict of per-plan feature gates, not documented.

#### WRONG_DESCRIPTION
- **`SafetyPolicy.check_input` order**: Doc says the order is (1) injection, (2) blocked patterns, (3) blocked topics. The actual code (config.py:147-185) order is (1) injection, (2) blocked patterns, (3) blocked topics. **Matches**. However, the doc at line 90-96 also mentions PII detection in `check_input` but the actual `check_input` in code does NOT perform PII detection -- only `check_output` does. The doc's description (lines 88-96) correctly describes `check_input` as checking injection/patterns/topics, but earlier in the SafetyPolicy method list it's fine. **No issue upon close inspection**.

---

### 9. reference/enterprise/licensing.md -> corteX/enterprise/licensing.py
**Status**: [OK] (matches)

All classes, methods, constants, and data structures match:

- **`LicenseStatus`**: 6 values match licensing.py:141-147.
- **`LicenseInfo`**: 9 attributes match licensing.py:150-161.
- **`UsageMeter`**: 7 attributes match licensing.py:164-173.
- **`LicenseManager`**: Constructor and 7 methods all match.
- **`generate_license_key`**: Signature and parameters match licensing.py:88-95.
- **Constants**: `OFFLINE_GRACE_DAYS=30`, `SECONDS_PER_DAY=86400` match.
- **Env vars**: `CORTEX_LICENSE_PUBLIC_KEY_B64`, `CORTEX_DEV_PRIVATE_KEY_B64` match.

No discrepancies found.

---

### 10. reference/enterprise/updates.md -> corteX/enterprise/updates.py
**Status**: [OK] (matches)

All classes, enums, methods, properties, and constants match:

- **Constants**: `SDK_VERSION="2.0.0"`, `SDK_VERSION_TUPLE=(2,0,0)` match.
- **`UpdateChannel`**: 3 values match.
- **`UpdateCheckResult`**: 5 values match.
- **`VersionInfo`**: 10 attributes match.
- **`UpdateConfig`**: 10 attributes match.
- **`UpdateState`**: 7 attributes match.
- **`UpdateManager`**: Constructor, 2 properties, and 8 methods all match.

No discrepancies found.

---

### 11. reference/enterprise/consent.md -> corteX/enterprise/consent.py + consent_types.py
**Status**: [OK] (matches)

All types, enums, and methods match:

- **`ConsentPurpose`**: 5 values match consent_types.py:23-29.
- **`ConsentMechanism`**: 5 values match consent_types.py:36-42.
- **`ConsentStatus`**: 3 values match consent_types.py:45-49.
- **`ConsentRecord`**: 11 attributes match consent_types.py:53-79. Methods `is_active()`, `to_dict()`, `from_dict()` all match.
- **`ConsentManager`**: Constructor and 9 methods all match consent.py exactly.

No discrepancies found.

---

### 12. reference/enterprise/gdpr.md -> corteX/enterprise/gdpr.py + gdpr_types.py
**Status**: [OK] (matches)

All enums, classes, methods, and constants match:

- **`DSARType`**: 7 values match gdpr_types.py:24-32.
- **`DSARStatus`**: 6 values match gdpr_types.py:35-42.
- **`ErasureScope`**: 9 values match gdpr_types.py:45-55.
- **`GDPRManager`**: Constructor with 5 parameters matches gdpr.py:49-55.
- **Methods**: `export_user_data`, `rectify_user_data`, `erase_user_data`, `restrict_processing`, `is_processing_restricted`, `export_portable_data`, `register_objection`, `has_objection`, `get_processing_info`, `get_request_log` -- all 10 methods match signatures and return types.
- **`DSAR_RESPONSE_DEADLINE_DAYS = 30`** matches gdpr.py:43.

No discrepancies found.

---

### 13. reference/enterprise/retention.md -> corteX/enterprise/retention.py + retention_types.py
**Status**: [OK] (matches)

All types, enums, classes, and methods match:

- **`DataCategory`**: 8 values match retention_types.py:23-32.
- **`RetentionPolicy`**: 4 attributes match retention_types.py:35-46.
- **`ExpiredItem`**: 7+1 attributes match retention_types.py:80-94 (doc lists 7, code has 7 + `metadata` which doc omits -- **minor gap**).
- **`PurgeResult`**: 7 attributes match retention_types.py:97-118.
- **`RetentionReport`**: 7 attributes match retention_types.py:49-66.
- **`CascadeDeleteResult`**: 4 attributes match retention_types.py:130-136.
- **`RetentionEnforcer`**: Constructor and 6 methods all match retention.py.
- **Default TTLs**: All 8 categories match retention.py:43-52.
- **Compliance presets**: All 4 frameworks (GDPR, SOC 2, HIPAA, PCI-DSS) match retention_types.py:141-186.

#### MINOR
- **`ExpiredItem.metadata`**: Code has `metadata: Dict[str, Any] = Field(default_factory=dict)` (retention_types.py:93), but the doc omits this field from the attributes table.

---

### 14. reference/enterprise/profiling.md -> corteX/enterprise/profiling.py
**Status**: [OK] (matches)

All enums, data classes, and methods match:

- **`ProfilingDecision`**: 5 values match profiling.py:24-30.
- **`ProfilingStatus`**: 8 attributes match profiling.py:33-43.
- **`ProfilingGuardResult`**: 5 attributes match profiling.py:55-62.
- **`ProfilingManager`**: Constructor and 11 methods all match profiling.py:72-235.

No discrepancies found.

---

### 15. reference/enterprise/explainability.md -> corteX/enterprise/explainability.py + explainability_types.py
**Status**: [A] (gaps found)

#### WRONG_ATTR
- **`DecisionExplanation` missing `latency_ms`, `tokens_consumed`, `metadata`**: The doc (lines 51-64) lists 12 attributes. The actual `DecisionExplanation` (explainability_types.py:116-132) has 14 attributes total. The doc omits:
  - `latency_ms: float = 0.0`
  - `tokens_consumed: int = 0`
  - `metadata: Dict[str, Any]`

- **`ToolSelectionExplanation` missing `alternatives`, `weight_influences`, `reasoning`, `human_readable`**: The doc (lines 67-82) lists 8 attributes. The actual class (explainability_types.py:85-98) has 12 attributes. The doc omits:
  - `alternatives: List[AlternativeConsidered]`
  - `weight_influences: List[WeightInfluence]`
  - `reasoning: str`
  - `human_readable: str`

- **`RoutingExplanation` missing `alternatives`, `weight_influences`, `reasoning`, `human_readable`**: The doc (lines 84-95) lists 5 attributes. The actual class (explainability_types.py:101-112) has 9 attributes. The doc omits:
  - `alternatives: List[AlternativeConsidered]`
  - `weight_influences: List[WeightInfluence]`
  - `reasoning: str`
  - `human_readable: str`

- **`SessionExplanationSummary` missing `human_readable`**: The doc (lines 97-114) lists 10 attributes. The actual class (explainability_types.py:183-195) has 11. The doc omits:
  - `human_readable: str`

#### WRONG_EXAMPLE
- **Example accesses `summary.human_readable`** (line 273) which actually does exist in code but is not listed in the doc's attributes table -- inconsistent.

#### MISSING_API
- **`AlternativeConsidered` dataclass**: Used by `DecisionExplanation.alternatives` but not documented anywhere on this page. Defined in explainability_types.py:38-43.
- **`WeightInfluence` dataclass**: Used by `DecisionExplanation.weight_influences` but not documented on this page. Defined in explainability_types.py:46-53.
- **`GoalAlignmentInfo` dataclass**: Used by `DecisionExplanation.goal_alignment` but not documented on this page. Defined in explainability_types.py:56-64.
- **`DecisionStep` dataclass**: Referenced by `get_decision_trace()` return type but not documented. Defined in explainability_types.py:68-81.

#### WRONG_DESCRIPTION
- **Import example**: Doc line 235 says `from corteX.enterprise.explainability_types import ExplanationLevel` but this is actually a valid import path. **OK**.

---

### 16. reference/enterprise/data-residency.md -> corteX/enterprise/data_residency.py
**Status**: [OK] (matches)

All enums, data classes, and methods match:

- **`Region`**: 11 values match data_residency.py:22-34.
- **`ResidencyCheck`**: 6 attributes match data_residency.py:37-45.
- **`TenantResidencyConfig`**: 6 attributes match data_residency.py:55-63.
- **`DataResidencyManager`**: Constructor and 10 methods all match data_residency.py:116-275.
- **Built-in provider mappings**: Doc's table matches the code's `_PROVIDER_REGIONS` list (data_residency.py:67-107).

No discrepancies found.

---

## SUMMARY

| # | Page | Source | Status | Gap Count |
|---|------|--------|--------|-----------|
| 1 | reference/engine.md | corteX/sdk.py | **[A]** | 1 WRONG_SIGNATURE |
| 2 | reference/agent.md | corteX/sdk.py | **[A]** | 1 WRONG_SIGNATURE, 1 MISSING_API |
| 3 | reference/session.md | corteX/session/ | **[A]** | 15+ MISSING_API, 1 WRONG_ATTR, 1 WRONG_DESCRIPTION |
| 4 | reference/response.md | corteX/sdk_config.py | **[OK]** | 0 |
| 5 | reference/core/events.md | corteX/core/events.py | **[OK]** | 0 |
| 6 | reference/core/contracts.md | corteX/core/contracts.py | **[OK]** | 0 |
| 7 | reference/core/registry.md | corteX/core/registry.py | **[OK]** | 0 |
| 8 | reference/enterprise/config.md | corteX/enterprise/config.py | **[A]** | 3 MISSING_API |
| 9 | reference/enterprise/licensing.md | corteX/enterprise/licensing.py | **[OK]** | 0 |
| 10 | reference/enterprise/updates.md | corteX/enterprise/updates.py | **[OK]** | 0 |
| 11 | reference/enterprise/consent.md | corteX/enterprise/consent.py | **[OK]** | 0 |
| 12 | reference/enterprise/gdpr.md | corteX/enterprise/gdpr.py | **[OK]** | 0 |
| 13 | reference/enterprise/retention.md | corteX/enterprise/retention.py | **[OK]** | 1 minor (missing `metadata` attr) |
| 14 | reference/enterprise/profiling.md | corteX/enterprise/profiling.py | **[OK]** | 0 |
| 15 | reference/enterprise/explainability.md | corteX/enterprise/explainability.py | **[A]** | 4 WRONG_ATTR, 4 MISSING_API, 1 WRONG_EXAMPLE |
| 16 | reference/enterprise/data-residency.md | corteX/enterprise/data_residency.py | **[OK]** | 0 |

**Total pages with gaps: 5 of 16**
**Total pages OK: 11 of 16**

### Priority Fixes

1. **HIGH - Engine/Agent docs**: Add `mcp_servers` and `a2a_agents` parameters (interop feature undocumented).
2. **HIGH - Session doc**: Add 15+ missing public methods and 8+ missing attributes to the attributes table.
3. **MEDIUM - Explainability doc**: Add missing attributes to `DecisionExplanation`, `ToolSelectionExplanation`, `RoutingExplanation`, `SessionExplanationSummary`. Document supporting types (`AlternativeConsidered`, `WeightInfluence`, `GoalAlignmentInfo`, `DecisionStep`).
4. **LOW - Enterprise config doc**: Add `data_classification_enabled`, `pii_protection_enabled`, `LicenseConfig.allowed_features` attributes.
5. **LOW - Retention doc**: Add `ExpiredItem.metadata` to attributes table.
