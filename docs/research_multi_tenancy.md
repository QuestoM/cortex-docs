# Multi-Tenancy Research Report: corteX Enterprise AI Agent SDK

**Author:** Senior AI Engineer (Research)
**Date:** 2026-02-15
**Status:** Research Complete -- Ready for Architecture Review
**Scope:** Complete tenant isolation analysis, gap identification, and architecture recommendations

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Multi-Tenancy Model for corteX](#multi-tenancy-model-for-cortex)
3. [Tenant Isolation Requirements](#1-tenant-isolation-requirements)
4. [Session-Level Isolation Analysis](#2-session-level-isolation-analysis)
5. [Per-Tenant Configuration](#3-per-tenant-configuration)
6. [API Key Management (BYOK)](#4-api-key-management-byok)
7. [Resource Management](#5-resource-management)
8. [Enterprise Security Patterns](#6-enterprise-security-patterns)
9. [Gap Analysis: Current Codebase](#7-gap-analysis-current-codebase)
10. [Architecture Recommendations](#architecture-recommendations)
11. [Implementation Priority List](#implementation-priority-list)
12. [Security Checklist](#security-checklist)

---

## Executive Summary

corteX operates in a three-layer multi-tenancy model: Questo (SDK provider) -> Developer (SDK consumer who builds a SaaS product) -> End-User (customer of the developer's product). Each layer introduces isolation requirements that compound. This report analyzes the current codebase, identifies 14 isolation gaps (5 critical, 4 high, 5 medium), and provides a prioritized remediation plan.

**Key Findings:**

- The SDK's `Session` class provides good per-session component isolation (each session gets its own WeightEngine, MemoryFabric, GoalTracker, etc.)
- The `PluginRegistry` was recently refactored to support per-instance isolation -- good progress
- **5 Critical Global State Violations** exist that would cause cross-tenant data leakage in production
- The enterprise config system (`TenantConfig`) is well-designed but not wired into the runtime path
- No per-tenant rate limiting, cost tracking, or resource quotas exist at the SDK level

---

## Multi-Tenancy Model for corteX

### The Three-Layer Model

```
Layer 1: Questo (SDK Provider)
    |
    +-- Layer 2: Developer A (SaaS App using corteX)
    |       |
    |       +-- Layer 3: End-User A1 (customer of Developer A)
    |       +-- Layer 3: End-User A2
    |       +-- Layer 3: End-User A3
    |
    +-- Layer 2: Developer B (Different SaaS App)
            |
            +-- Layer 3: End-User B1
            +-- Layer 3: End-User B2
```

### Isolation Boundaries

| Boundary | Isolation Type | Current Status |
|----------|---------------|----------------|
| Questo <-> Developer | SDK license + config | Partial (LicenseConfig exists) |
| Developer <-> Developer | Process-level (separate installs) | Good (separate pip installs) |
| End-User <-> End-User | Session-level within same process | **Gaps exist (see below)** |
| Agent <-> Agent | Engine-level within same developer | Partial |

### Primary Deployment Scenarios

1. **On-Prem Single-Tenant**: Developer deploys corteX for one customer. Lowest isolation risk.
2. **On-Prem Multi-Tenant**: Developer's SaaS serves multiple customers from one deployment. **Highest risk -- this is our primary concern.**
3. **Cloud/SaaS**: Developer uses corteX in a cloud SaaS. Isolation provided by infrastructure + SDK.

---

## 1. Tenant Isolation Requirements

### What MUST Be Isolated (Per End-User / Per Tenant)

| Resource | Why | Current State |
|----------|-----|---------------|
| **Memory** (Working, Episodic, Semantic) | Contains user conversations, goals, learned patterns | Isolated per Session (good) |
| **Weights** (all 7 categories) | Contains behavioral preferences, tool preferences, user insights | Isolated per Session (good) |
| **Conversation History** | Private user messages | Isolated per Session (good) |
| **API Keys** | Tenant's LLM credentials | **NOT isolated -- env vars are process-global** |
| **Tool State** | Tool execution results, registered tools | Partially isolated (ToolExecutor per session, but global `_registered_tools`) |
| **Policy/Safety Config** | Per-tenant safety rules, blocked topics | Exists in TenantConfig but not wired to Session |
| **Audit Logs** | Per-tenant compliance trail | AuditLogger has tenant_id field but no enforcement |
| **GoalTracker State** | User's current goals, progress | Isolated per Session (good) |
| **Brain State** (all engine modules) | Attention, calibration, columns, prediction, etc. | Isolated per Session (good) |
| **EventBus Subscriptions** | System events | **NOT isolated -- class-level global dict** |
| **ContextBroker** | Memory management backend | **NOT isolated -- module-level global singleton** |

### What CAN Be Shared Safely

| Resource | Why Sharing Is OK |
|----------|-------------------|
| SDK code (Python modules) | Read-only, stateless |
| LLM Provider classes (code) | Stateless factories |
| Tool framework code (decorators, executor class) | Stateless |
| Static configurations (default learning rates, thresholds) | Immutable defaults |
| Model weight files (e.g., neurollama) | Read-only inference |
| Plugin class definitions (not instances) | Type definitions only |

### Data Leakage Vectors

1. **Global singletons**: `EventBus._subscribers`, `ContextBroker(broker)`, `_registered_tools`, `_default_registry`
2. **Environment variables**: `os.getenv("GEMINI_API_KEY")` is process-wide
3. **Logging**: Logger output may intermix tenant data without tenant_id tagging
4. **File paths**: FileBackend for memory uses shared filesystem without tenant namespacing
5. **sys.path manipulation**: `PluginRegistry.load_from_directory()` modifies global `sys.path`

---

## 2. Session-Level Isolation Analysis

### Session Architecture Review

The `Session` class (sdk.py line 212) is the primary isolation boundary. Each Session instantiates its own:

```python
# From Session.__init__ (sdk.py lines 245-398)
self.weights = WeightEngine()           # Own weight state
self.feedback = FeedbackEngine(...)     # Own feedback history
self.prediction = PredictionEngine()    # Own prediction state
self.plasticity = PlasticityManager(...)# Own plasticity
self.memory = MemoryFabric()            # Own 3-tier memory
self.dual_process = DualProcessRouter() # Own routing state
self.reputation = ReputationSystem()    # Own tool trust
self.context_engine = CorticalContextEngine(...)  # Own context
self.proactive = ProactivePredictionEngine()      # Own predictions
self.calibration = ContinuousCalibrationEngine()  # Own calibration
self.columns = ColumnManager()          # Own columns
self.resource_map = ResourceHomunculus() # Own resource map
self.attention = AttentionSystem(...)   # Own attention
self.concepts = ConceptGraphManager()   # Own concept graph
self.reorganizer = CorticalMapReorganizer()  # Own reorganizer
self.modulator = TargetedModulator()    # Own modulator
self.simulator = ComponentSimulator()   # Own simulator
self.audit = AuditLogger(...)           # Own audit logger
self._tool_executor = ToolExecutor(tools=agent.tools)  # Own tool executor
self._goal_tracker = None               # Own goal tracker (lazy)
self._messages = []                     # Own conversation history
```

**Verdict: Session-level component isolation is GOOD.** Each session creates fresh instances of all brain components. No session can access another session's WeightEngine, MemoryFabric, or GoalTracker.

### Remaining Session-Level Gaps

1. **Shared Agent Reference**: Multiple sessions from the same Agent share `self.agent`, which holds `engine`, `system_prompt`, and `tools`. The `tools` list is shared by reference -- if a tool has mutable internal state, it leaks across sessions.

2. **Shared Engine/Router**: `self.agent.engine.router` is shared across all sessions created by the same Engine. The LLMRouter holds `_providers`, `_latency_history`, `_failure_counts`, and `_circuit_breaker` state. One tenant's API failures could trip the circuit breaker for all tenants sharing the same Engine.

3. **No Session-to-Tenant Binding**: Sessions have `user_id` but no `tenant_id`. The AuditLogger's `tenant_id` field defaults to empty string.

### Can One Session's Memory Leak Into Another?

**Within the same Engine/Agent scope**: No, because each Session creates its own `MemoryFabric()` with fresh `InMemoryBackend()` instances.

**Exception**: If a developer passes a shared `FileBackend` or external backend, memories could overlap. The SDK does not enforce tenant-namespaced paths for file-based persistence.

**Exception**: The global `ContextBroker` singleton (`corteX/memory/manager.py` line 185) has `_short_term_memory`, `_trajectories`, and `_graph_nodes` that are module-level shared state. If any code path uses this singleton instead of the per-session MemoryFabric, data leaks.

---

## 3. Per-Tenant Configuration

### Current State: TenantConfig

The `TenantConfig` class (enterprise/config.py line 371) is well-designed:

```python
@dataclass
class TenantConfig:
    tenant_id: str = "default"
    tenant_name: str = ""
    safety: SafetyPolicy        # Per-tenant safety rules
    models: ModelPolicy          # Per-tenant model restrictions
    tools: ToolPolicy            # Per-tenant tool restrictions
    audit: AuditConfig           # Per-tenant audit settings
    license: LicenseConfig       # Per-tenant license
    data_retention: DataRetention
    compliance: List[ComplianceFramework]
    custom_settings: Dict[str, Any]
    user_overridable: Set[str]   # Which settings users can override
```

This is a comprehensive tenant configuration object that covers safety, models, tools, audit, licensing, and compliance.

### Gap: TenantConfig Not Wired Into Session

The `Session` class accepts an `EnterpriseConfig` (a simpler Pydantic model at sdk.py line 177) but does NOT accept a full `TenantConfig`. The rich policy infrastructure in `TenantConfig` (ModelPolicy, ToolPolicy, AuditConfig, LicenseConfig) is disconnected from the runtime.

```python
# What exists (sdk.py line 177):
class EnterpriseConfig(BaseModel):
    safety_level: str = "moderate"
    data_retention: str = "none"
    allowed_models: List[str] = []
    blocked_topics: List[str] = []
    audit_log: bool = False
    ...

# What should be used instead:
# TenantConfig from enterprise/config.py (much richer)
```

### Config Inheritance Model (Recommended)

```
Questo Defaults (SDK defaults)
    |
    v
Developer-Level Config (Engine constructor)
    |
    v
Tenant-Level Config (TenantConfig per customer)
    |
    v
User-Level Overrides (only allowed fields from user_overridable set)
```

### Hot-Reload Requirements

For production multi-tenant deployments, configuration must be reloadable without restart:

1. **File-based**: Watch `TenantConfig` JSON files for changes, reload on modify
2. **API-based**: Expose admin endpoint to push config updates
3. **Runtime**: `Session` should check config freshness at turn boundaries (not every call)

Currently no hot-reload mechanism exists. `TenantConfig.load()` reads from file but is only called at initialization.

---

## 4. API Key Management (BYOK)

### Current State: Critical Gap

API keys are resolved through environment variables:

```python
# corteX/config.py
def get_gemini_secret_name() -> str:
    return os.getenv("GEMINI_SECRET_NAME", "gemini_api_key")

# corteX/core/llm/gemini_adapter.py
self.api_key = api_key or os.environ.get("GEMINI_API_KEY", "")

# corteX/core/llm/gemini_client.py
self.api_key = api_key or os.environ.get("GEMINI_API_KEY") or self._fetch_secret_key()
```

Environment variables are **process-global**. In a multi-tenant deployment, ALL tenants would use the SAME API key. This violates BYOK and makes cost attribution impossible.

### Current Key Flow (Good Part)

The `Engine` constructor does accept per-provider API keys:

```python
engine = cortex.Engine(providers={
    "openai": {"api_key": "sk-tenant-specific-key"},
    "gemini": {"api_key": "AIza-tenant-specific-key"},
})
```

This means if a developer creates a separate `Engine` per tenant, keys are isolated. However:
- If `api_key` is empty/missing, fallback to `os.environ` leaks the process-level key
- No mechanism to rotate keys without recreating the Engine
- No key validation or health-checking per tenant

### BYOK Architecture Requirements

1. **Key Resolution Chain**: `explicit_key > tenant_config_key > NEVER_fall_back_to_env`
2. **Key Storage**: Keys should be held in memory only, never logged, never serialized to disk
3. **Key Rotation**: Support rotating keys without session interruption
4. **Key Validation**: Validate key on registration (test API call)
5. **Key Isolation**: Each LLMRouter instance holds its own provider instances with their own keys
6. **Error Masking**: API errors must NEVER include the key in logs or error messages

### Key Leakage Risks

| Risk | Location | Severity |
|------|----------|----------|
| Key in error messages | LLM provider error handlers | HIGH |
| Key in logs | Provider registration log lines | MEDIUM |
| Key in serialized state | Weight/config save/load | LOW (currently keys not saved) |
| Key in memory dumps | Process crash dumps | MEDIUM |
| Key fallback to env | gemini_adapter.py, gemini_client.py | CRITICAL |

---

## 5. Resource Management

### Current State: No Per-Tenant Resource Controls

The SDK has some resource management primitives, but none are tenant-scoped:

1. **Rate Limiter** (resilience.py): Exists but is per-provider, not per-tenant. One tenant's heavy usage blocks all tenants sharing the same provider.

2. **Token Tracking**: `Session._total_tokens` tracks tokens per session, and `EnterpriseConfig.max_tokens_per_session` sets a limit, but there is no enforcement -- the limit is never checked during execution.

3. **Tool Timeout**: `ToolExecutor.timeout` is configurable but not per-tenant.

4. **No Cost Tracking**: No mechanism to attribute LLM costs to specific tenants.

5. **No Concurrency Limits**: No limit on concurrent sessions per tenant.

### Required Resource Controls

| Control | Scope | Priority |
|---------|-------|----------|
| RPM rate limit | Per tenant | CRITICAL |
| Token budget enforcement | Per session | HIGH |
| Concurrent session limit | Per tenant | HIGH |
| Cost attribution | Per tenant | MEDIUM |
| Tool execution quota | Per session | MEDIUM |
| Memory capacity limits | Per tenant | MEDIUM |
| Model access control | Per tenant | HIGH (ModelPolicy exists but not enforced) |

### Rate Limiting Architecture

```
Per-Tenant Rate Limiter
    |
    +-- RPM limit (requests per minute)
    +-- RPD limit (requests per day)
    +-- TPM limit (tokens per minute)
    +-- Concurrent request limit
    +-- Burst allowance (token bucket)
    |
    Enforcement Point: LLMRouter.generate() -- BEFORE API call
```

---

## 6. Enterprise Security Patterns

### Industry Best Practices (2025-2026)

Based on research of current enterprise AI multi-tenancy patterns:

**1. Tenant Context Propagation (Microsoft Azure / AWS pattern)**

Every request, background job, and data access path must carry a tenant identifier. Python's `contextvars` module enables this without threading issues:

```python
import contextvars
current_tenant = contextvars.ContextVar('current_tenant', default='unknown')
```

This is NOT currently implemented in corteX.

**2. Zero-Trust Tenant Isolation (AWS Lambda pattern)**

Isolation should begin at the SDK entry point. Tenant resolution happens BEFORE any business logic:

```python
engine = cortex.Engine(tenant_id="acme_corp", ...)
# All subsequent operations are scoped to this tenant
```

corteX partially does this with the Engine constructor's `tenant_id` parameter, but it's not enforced downstream.

**3. Defense in Depth (NIST / SOC2 CC6.3)**

Multiple isolation layers, not just one:
- Layer 1: Tenant ID on every object (audit trail)
- Layer 2: Logical isolation (separate data stores per tenant)
- Layer 3: Policy enforcement (TenantConfig governs behavior)
- Layer 4: Encryption (per-tenant keys for data at rest)
- Layer 5: Runtime isolation (resource quotas, rate limits)

### Compliance Implications

| Framework | Multi-Tenant Requirement | corteX Status |
|-----------|-------------------------|---------------|
| **SOC2 CC6.3** | Logical access controls between tenants | Partial (SafetyPolicy exists) |
| **SOC2 CC7.2** | Monitor for anomalous tenant access | Missing (no cross-tenant access detection) |
| **GDPR Art. 5** | Data minimization per tenant | Missing (no tenant-scoped data retention) |
| **GDPR Art. 17** | Right to erasure per tenant | Partial (MemoryFabric.clear() exists) |
| **GDPR Art. 32** | Appropriate security measures | Missing (no per-tenant encryption) |
| **HIPAA** | PHI isolation between covered entities | Missing (no healthcare-grade isolation) |
| **ISO 27001 A.12.4** | Audit logging per tenant | Partial (AuditLogger has tenant_id) |

### Audit Logging Requirements

The current `AuditLogger` (engine/audit_logger.py) is well-structured with:
- Event types: tool execution, LLM calls, policy decisions, goal events, sessions
- Multiple backends: file, structured logger, callback
- Tenant ID field on every entry

However, it needs:
- **Mandatory tenant_id**: Currently defaults to empty string
- **Log segregation**: Per-tenant log files or streams
- **PII scrubbing**: Before writing audit entries
- **Tamper protection**: Signed or chained audit entries
- **Retention enforcement**: Per-tenant retention policies

---

## 7. Gap Analysis: Current Codebase

### CRITICAL Gaps (Data Leakage / Security)

#### GAP-MT-1: EventBus Uses Class-Level Mutable Dict (CRITICAL)

**File:** `corteX/core/events.py` lines 31, 33-51

```python
class EventBus:
    _subscribers: Dict[EventType, List[EventHandler]] = {}  # CLASS variable!

    @classmethod
    def subscribe(cls, event_type, handler):
        cls._subscribers[event_type].append(handler)  # Shared across ALL instances
```

**Impact:** All EventBus instances in the same process share the same subscriber dict. A handler registered by Tenant A's session receives events from Tenant B's session. This is a class-level mutable default -- a well-known Python anti-pattern.

**Fix:** Convert to instance-level dict. Remove `@classmethod`. Create per-session EventBus instances.

---

#### GAP-MT-2: ContextBroker Is a Global Singleton (CRITICAL)

**File:** `corteX/memory/manager.py` line 185

```python
# Global Singleton
broker = ContextBroker(use_gemini_backend=True)
```

**Impact:** The module-level `broker` instance holds `_short_term_memory`, `_trajectories`, and `_graph_nodes` that are shared across all imports. Any code path using `from corteX.memory.manager import broker` accesses shared state.

**Fix:** Remove module-level singleton. Create ContextBroker instances per Engine or per Session.

---

#### GAP-MT-3: Global Tool Registry (CRITICAL)

**File:** `corteX/tools/decorator.py` line 233

```python
_registered_tools: Dict[str, ToolWrapper] = {}
```

**Impact:** The `@tool` decorator registers tools into a module-level dict. If Developer A registers a tool named `search_db` and Developer B registers a different `search_db`, they overwrite each other. In a multi-tenant deployment within a single process, tool definitions leak across tenants.

**Fix:** Tools should be registered into per-Engine or per-Agent scope, not module-level. The `@tool` decorator should be deprecated in favor of explicit `engine.register_tool()` or passing tools directly to `Agent`.

Note: The `ToolExecutor` inside `Session` is already properly scoped (it receives `agent.tools` explicitly), so this gap primarily affects the `@tool` decorator API path.

---

#### GAP-MT-4: API Key Fallback to Environment Variables (CRITICAL)

**Files:**
- `corteX/core/llm/gemini_adapter.py` line 161
- `corteX/core/llm/gemini_client.py` line 36
- `corteX/memory/file_search_driver.py` lines 36, 183

```python
self.api_key = api_key or os.environ.get("GEMINI_API_KEY", "")
```

**Impact:** If a tenant does not provide an API key, the system silently falls back to the process-level environment variable. This means one tenant could inadvertently use another tenant's key (if set in the environment), making cost attribution wrong and violating BYOK isolation.

**Fix:** If `api_key` is not explicitly provided, raise a `ValueError` instead of falling back to env. The developer must always provide keys explicitly per-tenant.

---

#### GAP-MT-5: sys.path Modification in PluginRegistry (CRITICAL)

**File:** `corteX/core/registry.py` lines 94-95

```python
if str(root_path) not in sys.path:
    sys.path.append(str(root_path))
```

**Impact:** `sys.path` is process-global. When one tenant loads plugins from a directory, that directory becomes importable by ALL tenants in the process. A malicious tenant could craft a plugin directory that shadows standard library modules.

**Fix:** Use `importlib` with explicit module specs instead of modifying `sys.path`. Or isolate plugin loading into subprocess/container boundaries.

---

### HIGH Gaps (Functional Isolation)

#### GAP-MT-6: LLMRouter State Shared Across Sessions (HIGH)

**File:** `corteX/core/llm/router.py` lines 117-127

The `LLMRouter` holds `_latency_history`, `_failure_counts`, `_circuit_breaker`, and `_rate_limiter` that are shared across all sessions created by the same Engine. One tenant's provider failures affect all tenants' circuit breaker state.

**Fix:** Either create per-tenant LLMRouter instances, or make circuit breaker and rate limiter tenant-aware (keyed by `tenant_id + provider`).

---

#### GAP-MT-7: No Tenant ID Propagation (HIGH)

The `Session` class has `user_id` but no `tenant_id`. The `Agent` class has no `tenant_id`. The `Engine` class accepts `tenant_id` but only passes it to `PluginRegistry` -- it does not propagate to Sessions, AuditLogger, or LLMRouter.

**Fix:** Add `tenant_id` to `Engine` -> `Agent` -> `Session` -> all components. Use `contextvars.ContextVar` for implicit propagation through async call chains.

---

#### GAP-MT-8: EnterpriseConfig Not Connected to TenantConfig (HIGH)

The `EnterpriseConfig` (sdk.py line 177) is a simplified duplicate of what `TenantConfig` (enterprise/config.py) already provides. ModelPolicy, ToolPolicy, and AuditConfig from TenantConfig are never used in the runtime path.

**Fix:** Replace `EnterpriseConfig` with `TenantConfig` in the SDK surface. Wire `ModelPolicy.is_model_allowed()` into `LLMRouter.generate()`. Wire `ToolPolicy.is_tool_allowed()` into `ToolExecutor.execute()`.

---

#### GAP-MT-9: No Token Budget Enforcement (HIGH)

`EnterpriseConfig.max_tokens_per_session` is stored but never checked. A tenant can consume unlimited tokens.

**Fix:** Check `self._total_tokens >= max_tokens` in `Session.run()` before calling `LLMRouter.generate()`. Return an error response when exceeded.

---

### MEDIUM Gaps (Best Practice Violations)

#### GAP-MT-10: FileBackend Memory Not Tenant-Namespaced (MEDIUM)

`FileBackend(base_path)` stores memory items at the given path without tenant namespacing. If two tenants use the same `base_path`, their memories collide.

**Fix:** Auto-prepend `tenant_id` to file paths: `base_path / tenant_id / ...`

---

#### GAP-MT-11: Server Orchestrator Is a Global Singleton (MEDIUM)

**File:** `corteX/server/main.py` line 14

```python
orchestrator = Orchestrator()  # Singleton Runtime
```

**Fix:** For multi-tenant server deployments, the orchestrator should be per-tenant or carry tenant context.

---

#### GAP-MT-12: No Per-Tenant Log Segregation (MEDIUM)

Python logging is process-global. All tenants' logs go to the same handlers. Tenant A's admin could see Tenant B's errors.

**Fix:** Use structured logging with mandatory `tenant_id` field. Optionally, per-tenant log handlers.

---

#### GAP-MT-13: Default Plugin Registry Imports (MEDIUM)

**File:** `corteX/core/registry.py` line 190

```python
self.plugin_registry.import_from(PluginRegistry.get_default())
```

Every new Engine starts with the global default registry's plugins. If a previous Engine registered plugins into the default registry, they would leak into new Engines.

**Fix:** The default registry should only contain built-in corteX plugins (read-only). Custom plugin registration should never target the default registry.

---

#### GAP-MT-14: Environment-Based Configuration (MEDIUM)

`corteX/config.py` reads all configuration from environment variables (GCP_PROJECT, GCS_ARTIFACT_BUCKET, etc.). These are process-global.

**Fix:** Configuration should flow through the Engine/TenantConfig, not environment variables. Env vars should only be used for process-level settings (log level, debug mode), never for tenant-specific settings.

---

## Architecture Recommendations

### Recommendation 1: Tenant Context Object

Create a `TenantContext` that propagates through the entire call chain:

```python
@dataclass
class TenantContext:
    tenant_id: str
    config: TenantConfig
    api_keys: Dict[str, str]  # provider -> key (in-memory only)
    rate_limiter: TenantRateLimiter
    audit_logger: AuditLogger

    # contextvars for async propagation
    _current: ClassVar[contextvars.ContextVar] = contextvars.ContextVar('tenant_ctx')

    @classmethod
    def current(cls) -> 'TenantContext':
        return cls._current.get()
```

### Recommendation 2: Isolate All Global State

Replace every global mutable with per-instance state:

| Global | Replace With |
|--------|-------------|
| `EventBus._subscribers` (class dict) | Per-Engine EventBus instance |
| `broker = ContextBroker(...)` | Per-Engine ContextBroker |
| `_registered_tools: Dict` | Per-Agent tool list (already partially done) |
| `_default_registry` | Read-only built-in registry |
| `os.environ` for keys | Explicit key injection via Engine/TenantConfig |

### Recommendation 3: Per-Tenant Rate Limiting

Implement a `TenantRateLimiter` that wraps the existing `RateLimiter`:

```python
class TenantRateLimiter:
    """Per-tenant rate limiting with configurable limits."""

    def __init__(self, tenant_id: str, config: ModelPolicy):
        self.tenant_id = tenant_id
        self.rpm_limit = config.max_requests_per_minute
        self.tpm_limit = config.max_tokens_per_session
        self._request_times: deque = deque()
        self._token_count: int = 0

    async def acquire(self, estimated_tokens: int = 0) -> bool:
        """Returns True if request is allowed, False if rate-limited."""
        ...
```

### Recommendation 4: Engine = Tenant Boundary

Formalize the `Engine` as the tenant boundary:

```python
engine = cortex.Engine(
    tenant_id="acme_corp",
    tenant_config=TenantConfig.load("config/acme.json"),
    providers={
        "gemini": {"api_key": "tenant-specific-key"},
    },
)
# Everything created from this engine is tenant-isolated
agent = engine.create_agent(...)
session = agent.start_session(user_id="user_123")
```

The Engine should:
- Own its own `PluginRegistry` (already done)
- Own its own `LLMRouter` (already done)
- Own its own `EventBus` (not done)
- Enforce `TenantConfig` policies on all child objects
- Propagate `tenant_id` to all child objects

### Recommendation 5: Defense in Depth Layers

```
Layer 1: Input Validation
    - SafetyPolicy.check_input() on every user message
    - Tenant ID required on every API call

Layer 2: Policy Enforcement
    - ModelPolicy.is_model_allowed() before LLM call
    - ToolPolicy.is_tool_allowed() before tool execution
    - TokenBudget check before LLM call

Layer 3: Resource Isolation
    - Per-tenant rate limiter
    - Per-tenant memory capacity
    - Per-tenant token budget

Layer 4: Audit Trail
    - Every action logged with tenant_id
    - Per-tenant log segregation
    - Tamper-evident audit chain

Layer 5: Data Isolation
    - Per-tenant memory backends
    - Per-tenant file paths
    - Per-tenant encryption keys (future)
```

### Recommendation 6: Tenant Lifecycle Management

```python
class TenantManager:
    """Manages tenant lifecycle: creation, config, teardown."""

    def create_tenant(self, tenant_id: str, config: TenantConfig) -> Engine:
        """Create a new tenant-scoped engine."""
        ...

    def get_tenant(self, tenant_id: str) -> Engine:
        """Retrieve an existing tenant's engine."""
        ...

    def destroy_tenant(self, tenant_id: str) -> None:
        """Complete data erasure for a tenant (GDPR Art. 17)."""
        ...

    def update_config(self, tenant_id: str, config: TenantConfig) -> None:
        """Hot-reload tenant configuration."""
        ...
```

---

## Implementation Priority List

### Phase 1: Critical Fixes (Must Have Before Production)

| # | Gap | Effort | Impact |
|---|-----|--------|--------|
| 1 | GAP-MT-1: Fix EventBus class-level dict | 2h | Eliminates cross-tenant event leakage |
| 2 | GAP-MT-2: Remove ContextBroker singleton | 2h | Eliminates cross-tenant memory leakage |
| 3 | GAP-MT-3: Scope global tool registry | 4h | Eliminates cross-tenant tool leakage |
| 4 | GAP-MT-4: Remove API key env fallback | 2h | Enforces BYOK isolation |
| 5 | GAP-MT-5: Fix sys.path modification | 4h | Eliminates code injection risk |
| 6 | GAP-MT-7: Add tenant_id propagation | 8h | Foundation for all other isolation |

**Phase 1 Total: ~22 hours**

### Phase 2: Policy Enforcement (Required for Enterprise)

| # | Gap | Effort | Impact |
|---|-----|--------|--------|
| 7 | GAP-MT-8: Wire TenantConfig into Session | 8h | Enables per-tenant policies |
| 8 | GAP-MT-9: Enforce token budget | 4h | Prevents resource abuse |
| 9 | GAP-MT-6: Per-tenant LLMRouter state | 8h | Prevents cross-tenant circuit breaker |
| 10 | Implement TenantRateLimiter | 8h | Per-tenant rate limiting |

**Phase 2 Total: ~28 hours**

### Phase 3: Best Practices (Required for Compliance)

| # | Gap | Effort | Impact |
|---|-----|--------|--------|
| 11 | GAP-MT-10: Tenant-namespaced file paths | 4h | Prevents memory file collision |
| 12 | GAP-MT-12: Per-tenant log segregation | 8h | Compliance requirement |
| 13 | GAP-MT-14: Remove env-based config | 8h | Per-tenant config purity |
| 14 | GAP-MT-11: Server orchestrator scoping | 4h | Multi-tenant server support |
| 15 | GAP-MT-13: Read-only default registry | 2h | Prevent plugin leakage |
| 16 | TenantManager lifecycle class | 16h | Full tenant CRUD + GDPR erasure |
| 17 | Per-tenant cost tracking | 8h | Usage metering and billing |

**Phase 3 Total: ~50 hours**

### Grand Total: ~100 hours (approximately 2.5 weeks for 1 engineer)

---

## Security Checklist

### Pre-Production Multi-Tenancy Checklist

- [ ] **No global mutable state**: All module-level singletons eliminated or made read-only
- [ ] **Tenant ID on every object**: Engine, Agent, Session, AuditEntry all carry tenant_id
- [ ] **API keys never in env vars**: Keys only accepted via explicit constructor parameters
- [ ] **API keys never in logs**: All logging statements scrubbed for key patterns
- [ ] **API keys never in errors**: Exception messages sanitized before surfacing
- [ ] **Rate limiting per tenant**: RPM/TPM/RPD enforced before LLM calls
- [ ] **Token budget enforced**: max_tokens_per_session checked in Session.run()
- [ ] **Model access controlled**: ModelPolicy.is_model_allowed() checked in LLMRouter
- [ ] **Tool access controlled**: ToolPolicy.is_tool_allowed() checked in ToolExecutor
- [ ] **Memory isolated**: Working/Episodic/Semantic memory per-session with no shared backends
- [ ] **File paths namespaced**: FileBackend paths include tenant_id
- [ ] **Audit logging mandatory**: Every session has AuditLogger with tenant_id
- [ ] **Audit logs segregated**: Per-tenant log streams or files
- [ ] **PII detection active**: SafetyPolicy.pii_detection enforced on outputs
- [ ] **Plugin isolation**: Plugins loaded per-tenant, no sys.path modification
- [ ] **EventBus isolated**: Per-engine or per-session event subscribers
- [ ] **Circuit breaker scoped**: Per-tenant provider health tracking
- [ ] **Config hot-reloadable**: TenantConfig changes apply without restart
- [ ] **Data erasure supported**: Tenant deletion removes all data (GDPR Art. 17)
- [ ] **No cross-tenant queries**: Memory backends enforce tenant_id filtering
- [ ] **Compliance validated**: TenantConfig.compliance list determines active controls

### Testing Requirements

1. **Isolation Tests**: Create two Engines (tenants), verify:
   - Session A cannot access Session B's memory
   - Tool registered in Engine A is not visible in Engine B
   - API key for Tenant A is not used by Tenant B
   - Rate limit for Tenant A does not affect Tenant B
   - Event published by Session A does not reach Session B's handler
   - Audit log for Tenant A does not contain Tenant B's entries

2. **Stress Tests**: Run concurrent sessions across tenants, verify:
   - No memory corruption under concurrent access
   - Rate limits hold under load
   - Token budgets enforced even with rapid requests
   - Circuit breaker state is tenant-isolated

3. **Compliance Tests**:
   - GDPR erasure: Delete tenant, verify zero residual data
   - Audit completeness: Every action has an audit entry with correct tenant_id
   - PII detection: Verify PII never appears in audit logs or error messages

---

## References

- [Microsoft Azure: AI/ML Multi-Tenant Architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/approaches/ai-machine-learning)
- [AWS: Multi-Tenant Generative AI Environment](https://aws.amazon.com/blogs/machine-learning/build-a-multi-tenant-generative-ai-environment-for-your-enterprise-on-aws/)
- [AWS: Multi-Tenant Architectures for Agentic AI](https://docs.aws.amazon.com/pdfs/prescriptive-guidance/latest/agentic-ai-multitenant/agentic-ai-multitenant.pdf)
- [Azure AI in Production Guide: Chapter 13 Multi-Tenant Architecture](https://azure.github.io/AI-in-Production-Guide/chapters/chapter_13_building_for_everyone_multitenant_architecture)
- [Multi-Tenancy in AI Agentic Systems (Isuru Siriwardana)](https://isurusiri.medium.com/multi-tenancy-in-ai-agentic-systems-9c259c8694ac)
- [Building a Multi-Tenant Production-Grade AI Agent (Ingenimax)](https://ingenimax.ai/blog/building-multi-tenant-ai-agent)
- [Scalable Multi-Tenant Architectures for AI-Enabled SaaS (Brim Labs)](https://brimlabs.ai/blog/how-to-build-scalable-multi-tenant-architectures-for-ai-enabled-saas/)
- [Designing Secure Tenant Isolation in Python (InfoQ)](https://www.infoq.com/articles/serverless-tenant-isolation/)
- [Multi-Tenant Performance Crisis: Advanced Isolation 2026 (AddWeb)](https://www.addwebsolution.com/blog/multi-tenant-performance-crisis-advanced-isolation-2026)
- [Tenant Isolation in Multi-Tenant Systems (Security Boulevard)](https://securityboulevard.com/2025/12/tenant-isolation-in-multi-tenant-systems-architecture-identity-and-security/)
- [SaaS Privacy Compliance Requirements 2025 Guide](https://secureprivacy.ai/blog/saas-privacy-compliance-requirements-2025-guide)
- [Compliance Frameworks for AI Infrastructure (Introl)](https://introl.com/blog/compliance-frameworks-ai-infrastructure-soc2-iso27001-gdpr)
- [Why Most SaaS Architectures Fall Short for Enterprise AI](https://blog.x1discovery.com/2025/12/03/why-most-saas-architectures-fall-short-for-enterprise-grade-ai/)
- [WorkOS: Developer's Guide to SaaS Multi-Tenant Architecture](https://workos.com/blog/developers-guide-saas-multi-tenant-architecture)
- [BYOK: Bring Your Own Key Guide (Daon)](https://www.daon.com/resource/how-byok-empowers-organizations-with-true-data-ownership/)
