# Enterprise Foundation Architecture: Multi-Tenancy, Security, Observability

**Author:** Senior AI Architect
**Date:** 2026-02-15
**Status:** Architecture Design -- Ready for Implementation
**Scope:** Three-pillar enterprise foundation that transforms corteX from SDK to fortress

---

## 1. Executive Summary

corteX's brain-inspired engine is production-ready at the cognitive level, but the enterprise envelope -- the walls, locks, and meters around that brain -- has 14 known isolation gaps and zero runtime policy enforcement. This architecture closes every gap and introduces five inventions that no competing SDK offers: **Tenant DNA**, **Cascading Policy Cascade**, **Self-Healing Isolation**, **Capability Attenuation**, and **Cost Prediction**.

The design adds three new packages (`corteX/tenancy/`, `corteX/security/`, `corteX/observability/`) totaling ~15 modules, each under 300 lines, with zero new external dependencies.

---

## 2. Current State Analysis

### What Works Well
- `Session` creates isolated brain component instances (weights, memory, prediction, etc.)
- `TenantConfig` is richly modeled with safety, model, tool, audit, and license policies
- `PluginRegistry` was refactored to per-instance isolation
- `AuditLogger` has tenant_id/session_id fields and multi-backend output
- `PolicyEngine` evaluates tool/output/intent with 5 guardrail types

### Critical Gaps (5)
1. **EventBus** -- class-level `_subscribers` dict leaks events across all tenants
2. **ContextBroker** -- module-level singleton shares memory state process-wide
3. **Global tool registry** -- `_registered_tools` dict in `decorator.py` is module-level
4. **API key env fallback** -- `os.environ.get("GEMINI_API_KEY")` is process-global
5. **sys.path mutation** -- `load_from_directory()` modifies global import path

### Architectural Gap
`TenantConfig` (rich) and `EnterpriseConfig` (simple) are disconnected. The runtime path uses `EnterpriseConfig` but ignores `ModelPolicy.is_model_allowed()`, `ToolPolicy.is_tool_allowed()`, and `AuditConfig`. No contextvars propagation, no per-tenant rate limiting, no cost attribution.

---

## 3. Multi-Tenancy Architecture

### 3.1 TenantContext -- The Propagating Identity

Every operation in corteX carries an implicit tenant identity via Python's `contextvars`:

```python
# corteX/tenancy/context.py
@dataclass(frozen=True)
class TenantContext:
    tenant_id: str
    layer: TenantLayer          # QUESTO, DEVELOPER, END_USER
    config: TenantConfig
    key_vault: KeyVault         # encrypted, in-memory-only API keys
    quota: QuotaEnvelope        # current token/request budgets
    capabilities: CapabilitySet # what this tenant can do (see Security)
    dna: TenantDNA              # learned usage profile

    _current: ClassVar[ContextVar['TenantContext']] = ContextVar('_tenant_ctx')

    @classmethod
    def current(cls) -> 'TenantContext':
        return cls._current.get()

    @classmethod
    def bind(cls, ctx: 'TenantContext') -> Token:
        return cls._current.set(ctx)
```

Every async call inherits the context automatically. No explicit parameter passing needed -- `TenantContext.current()` is always available.

### 3.2 Three-Layer Isolation Model

```
Layer 1: Questo (SDK Provider)
    Controls: license validation, feature flags, global telemetry, SDK updates
    Owns: LicenseConfig, global rate ceilings, compliance enforcement

Layer 2: Developer (SaaS App Builder)
    Controls: model selection, tool registration, safety policies, budgets
    Owns: TenantConfig, API keys (BYOK), PluginRegistry, PolicyEngine

Layer 3: End-User (Customer of Developer's App)
    Controls: preferences, conversation style, autonomy level
    Owns: user_overridable subset of TenantConfig, MemoryFabric, session state
```

Each layer has a separate `TenantConfig` that cascades through the policy engine. Layer 3 can only relax settings within the bounds that Layer 2 explicitly permits via `user_overridable`.

### 3.3 Engine = Tenant Boundary (Formalized)

```python
# corteX/tenancy/manager.py
class TenantManager:
    """Lifecycle manager for all active tenants in a process."""

    async def create_tenant(self, tenant_id: str, config: TenantConfig,
                            keys: Dict[str, str]) -> Engine:
        """Provisions an isolated Engine with its own EventBus, Router, Registry."""

    async def get_tenant(self, tenant_id: str) -> Optional[Engine]:
        """Retrieve a live tenant's engine. Returns None if not found."""

    async def destroy_tenant(self, tenant_id: str) -> ErasureReceipt:
        """Full data erasure: memory, audit logs, weights, config. GDPR Art. 17."""

    async def update_config(self, tenant_id: str, config: TenantConfig) -> None:
        """Hot-reload tenant config. Applied at next turn boundary."""

    def list_tenants(self) -> List[TenantSummary]:
        """List active tenants with usage metrics."""
```

The `Engine` constructor wires the `TenantContext` into every child component:

```
Engine(tenant_id, config, keys)
  +-- own EventBus instance (not class-level)
  +-- own LLMRouter (with per-tenant circuit breaker + rate limiter)
  +-- own PluginRegistry (import_from built-in read-only defaults)
  +-- own KeyVault (encrypted in-memory, no env var fallback)
  +-- own QuotaEnvelope (token budget, RPM, concurrency)
  +-- Agents[]
       +-- Sessions[]
            +-- TenantContext bound via contextvars
```

### 3.4 Resource Quotas and Fair Scheduling

```python
# corteX/tenancy/quota.py
@dataclass
class QuotaEnvelope:
    tokens_per_day: int = 1_000_000
    tokens_per_session: int = 100_000
    requests_per_minute: int = 60
    requests_per_day: int = 10_000
    concurrent_sessions: int = 50
    max_tool_calls_per_turn: int = 10

    # Runtime counters (atomic via threading.Lock or asyncio.Lock)
    _tokens_used_today: int = 0
    _requests_this_minute: deque  # sliding window
    _active_sessions: int = 0

    async def acquire_tokens(self, estimated: int) -> QuotaDecision:
        """Returns ALLOW, SOFT_LIMIT (warning), or HARD_LIMIT (reject)."""

    async def acquire_request(self) -> QuotaDecision:
        """Sliding-window RPM check."""

    async def acquire_session(self) -> QuotaDecision:
        """Increment active sessions, reject if over limit."""
```

**Fair Scheduling**: When the process-level LLM capacity is saturated, requests from tenants with lower usage get priority. Implemented as a weighted fair queue keyed by `tenant_id`:

```python
class FairScheduler:
    """Weighted fair queue: tenants with lower usage_ratio get priority."""

    async def enqueue(self, tenant_id: str, request: LLMRequest) -> LLMResponse:
        """Queue request, schedule by inverse usage ratio."""
```

### 3.5 Tenant DNA -- Novel Invention

```python
# corteX/tenancy/dna.py
@dataclass
class TenantDNA:
    """Compact learned profile for a tenant. Persists across sessions."""
    avg_task_complexity: float = 0.5   # 0=simple, 1=complex
    preferred_model_tier: str = "worker"
    avg_tokens_per_task: int = 2000
    tool_usage_frequency: Dict[str, float] = field(default_factory=dict)
    peak_hours: List[int] = field(default_factory=list)  # UTC hours
    risk_profile: float = 0.3          # 0=conservative, 1=aggressive
    common_intents: List[str] = field(default_factory=list)
    last_updated: float = 0.0

    def update_from_session(self, session_metrics: SessionMetrics) -> None:
        """EMA update from completed session. Learns tenant behavior."""
```

New sessions start with DNA pre-loaded, enabling smarter model routing, pre-warming, and anomaly detection from the first turn.

---

## 4. Security Architecture

### 4.1 Zero Trust Agent Architecture

Every action passes through a 4-gate pipeline before execution:

```
User Input -> [Gate 1: Input Safety] -> [Gate 2: Policy Check]
           -> [Gate 3: Capability Verify] -> [Gate 4: Quota Check]
           -> Execute
           -> [Gate 5: Output Safety] -> [Gate 6: Audit Log]
           -> Response
```

**Principle**: The agent never trusts itself. Even after the LLM produces an action, it is validated against policies, capabilities, and quotas before execution.

### 4.2 Capability-Based Security -- Novel Invention

Replace role-based access (admin/user) with cryptographically signed capability tokens:

```python
# corteX/security/capabilities.py
@dataclass(frozen=True)
class Capability:
    resource: str         # "tool:search_db", "model:gpt-4", "memory:write"
    actions: FrozenSet[str]  # {"read", "execute", "write"}
    constraints: Dict[str, Any]  # {"max_calls": 10, "domains": ["*.acme.com"]}
    expires_at: Optional[float] = None

class CapabilitySet:
    """Immutable set of capabilities. Supports attenuation (subset only)."""

    def __init__(self, capabilities: List[Capability], signature: bytes):
        self._caps = frozenset(capabilities)
        self._signature = signature  # Ed25519 signed by Questo or Developer

    def has(self, resource: str, action: str) -> bool:
        """Check if this set grants action on resource."""

    def attenuate(self, subset: List[Capability]) -> 'CapabilitySet':
        """Create a reduced capability set. Can only REMOVE, never ADD."""

    def for_sub_agent(self) -> 'CapabilitySet':
        """Auto-attenuate for sub-agent delegation. Removes write, keeps read."""
```

**Sub-agent inheritance**: When Session spawns a sub-agent, capabilities are automatically attenuated. A sub-agent can never have MORE capabilities than its parent. This is enforced cryptographically.

### 4.3 Capability Attenuation by Risk -- Novel Invention

```python
# corteX/security/attenuation.py
class RiskAttenuator:
    """As risk increases, capabilities decrease. The agent becomes more careful."""

    def attenuate_by_risk(self, caps: CapabilitySet, risk: float) -> CapabilitySet:
        """
        risk 0.0-0.3: full capabilities
        risk 0.3-0.6: remove write, keep read+execute
        risk 0.6-0.8: remove execute, keep read only
        risk 0.8-1.0: read only + require human approval for any action
        """

    def compute_risk(self, action: str, context: TurnContext) -> float:
        """Risk score from: tool type, data classification, confidence, drift."""
```

### 4.4 Secret Management

```python
# corteX/security/vault.py
class KeyVault:
    """In-memory encrypted API key store. Keys never touch disk or logs."""

    def __init__(self, tenant_id: str):
        self._keys: Dict[str, bytes] = {}  # provider -> encrypted key
        self._fernet: Fernet  # per-tenant derived key from master secret

    def store(self, provider: str, api_key: str) -> None:
        """Encrypt and store. Original string is NOT retained."""

    def retrieve(self, provider: str) -> str:
        """Decrypt for one-time use. Caller must not store the result."""

    def rotate(self, provider: str, new_key: str) -> None:
        """Atomic key rotation. Old key is zeroed."""

    def detect_leak(self, text: str) -> Optional[str]:
        """Scan text for any stored key pattern. Returns provider if found."""
```

**Key resolution chain**: `KeyVault.retrieve(provider) -> ValueError if missing`. No `os.environ` fallback. This is enforced at the `LLMRouter` level.

### 4.5 Data Classification

```python
# corteX/security/classification.py
class DataLevel(str, Enum):
    PUBLIC = "public"           # Can be sent to any model
    INTERNAL = "internal"       # On-prem models only
    CONFIDENTIAL = "confidential"  # On-prem + audit required
    RESTRICTED = "restricted"   # On-prem + approval + full audit

class DataClassifier:
    """Assigns classification level to data. Flows through the pipeline."""

    def classify(self, text: str, context: TenantContext) -> DataLevel:
        """Classify based on PII detection, tenant policy, and content patterns."""

    def enforce(self, data_level: DataLevel, target_model: str,
                tenant: TenantContext) -> Tuple[bool, Optional[str]]:
        """Check if data at this level can be sent to this model."""
```

**Flow rule**: If input is CONFIDENTIAL, all derived outputs are at least CONFIDENTIAL. The classification level can only escalate, never downgrade, through the processing pipeline.

### 4.6 Audit Trail

```python
# corteX/security/audit.py (extends existing AuditLogger)
@dataclass
class EnterpriseAuditEntry:
    event_id: str
    tenant_id: str
    session_id: str
    user_id: str
    timestamp: float
    event_type: AuditEventType
    action: str                 # what was done
    justification: str          # why (from LLM reasoning)
    outcome: str                # success/failure
    cost_tokens: int            # tokens consumed
    data_level: DataLevel       # classification of data involved
    capabilities_used: List[str]  # which capabilities were exercised
    chain_hash: str             # SHA-256 of previous entry (tamper-evident)

class TenantAuditStream:
    """Per-tenant audit stream. Append-only, hash-chained, segregated."""

    async def append(self, entry: EnterpriseAuditEntry) -> None:
        """Append with chain hash. Thread-safe."""

    async def query(self, filters: AuditFilter) -> List[EnterpriseAuditEntry]:
        """Query entries. Enforces tenant_id isolation."""

    async def verify_chain(self) -> Tuple[bool, Optional[int]]:
        """Verify hash chain integrity. Returns (valid, break_index)."""

    async def export(self, format: str = "jsonl") -> AsyncIterator[str]:
        """Stream export for compliance. Supports jsonl, csv."""
```

### 4.7 Compliance as Code -- Novel Invention

```python
# corteX/security/compliance.py
class ComplianceEngine:
    """Machine-readable compliance policies. Enforced automatically."""

    def __init__(self, frameworks: List[ComplianceFramework]):
        self._rules = self._load_rules(frameworks)

    def enforce_gdpr(self, ctx: TenantContext) -> List[ComplianceAction]:
        """GDPR: data minimization, retention limits, erasure support."""

    def enforce_hipaa(self, ctx: TenantContext) -> List[ComplianceAction]:
        """HIPAA: PHI isolation, access logging, encryption requirement."""

    def enforce_soc2(self, ctx: TenantContext) -> List[ComplianceAction]:
        """SOC2: access controls, monitoring, incident response."""

    def pre_check(self, action: str, data_level: DataLevel,
                  ctx: TenantContext) -> ComplianceResult:
        """Check action against all active compliance frameworks."""
```

---

## 5. Observability Architecture

### 5.1 Decision Tracing -- Beyond Logging

```python
# corteX/observability/tracer.py
@dataclass
class DecisionTrace:
    trace_id: str
    parent_id: Optional[str]
    tenant_id: str
    step_type: str             # "model_selection", "tool_call", "plan_step"
    decision: str              # what was chosen
    alternatives: List[str]    # what was considered
    reasoning: str             # why this choice (from brain signals)
    confidence: float
    latency_ms: float
    tokens_consumed: int
    brain_state: Dict[str, float]  # attention, calibration, column weights

class DecisionTracer:
    """Records WHY the agent made each decision, not just what."""

    def trace_model_selection(self, routing: RoutingDecision,
                              brain_snapshot: BrainSnapshot) -> DecisionTrace:
        """Trace: which model was chosen, why, what alternatives existed."""

    def trace_tool_selection(self, tool: str, confidence: float,
                             alternatives: List[Tuple[str, float]]) -> DecisionTrace:
        """Trace: which tool was chosen, Bayesian scores, Thompson sampling state."""

    def trace_plan_step(self, step: str, risk: float,
                        reasoning: str) -> DecisionTrace:
        """Trace: plan step with risk assessment and reasoning."""
```

### 5.2 Cost Attribution

```python
# corteX/observability/cost.py
@dataclass
class CostRecord:
    tenant_id: str
    session_id: str
    task_id: str
    step_index: int
    provider: str
    model: str
    input_tokens: int
    output_tokens: int
    cost_usd: float            # computed from model pricing table
    timestamp: float

class CostTracker:
    """Per-tenant cost attribution with anomaly detection."""

    def record(self, record: CostRecord) -> None:
        """Record a cost event."""

    def get_tenant_cost(self, tenant_id: str, period: str = "today") -> CostSummary:
        """Aggregate cost by tenant for period (today, week, month)."""

    def detect_anomaly(self, record: CostRecord) -> Optional[CostAnomaly]:
        """Alert if this task costs >3x the historical average for this type."""

    async def predict_cost(self, plan_steps: List[str],
                           tenant: TenantContext) -> CostPrediction:
        """Before execution: estimate total cost. Ask approval if over budget."""
```

### 5.3 Cost Prediction -- Novel Invention

```python
# corteX/observability/prediction.py
@dataclass
class CostPrediction:
    estimated_tokens: int
    estimated_cost_usd: float
    confidence_interval: Tuple[float, float]  # (low, high)
    breakdown: List[StepEstimate]
    over_budget: bool
    budget_remaining: float

class CostPredictor:
    """Predict total plan cost BEFORE execution. Revolutionary for enterprise."""

    def predict(self, plan: List[PlanStep], tenant_dna: TenantDNA,
                model_pricing: Dict[str, float]) -> CostPrediction:
        """
        Uses tenant DNA (avg tokens per task type) + model pricing
        to estimate total cost. Returns prediction with confidence interval.
        If over_budget, the agent asks for approval before proceeding.
        """
```

### 5.4 Metrics and Health Dashboard

```python
# corteX/observability/metrics.py
class MetricsCollector:
    """Collects and exposes metrics for monitoring systems."""

    def record_latency(self, tenant_id: str, operation: str, ms: float) -> None:
    def record_tokens(self, tenant_id: str, model: str, count: int) -> None:
    def record_success(self, tenant_id: str, operation: str, success: bool) -> None:
    def record_drift(self, tenant_id: str, score: float) -> None:

    def get_dashboard(self, tenant_id: Optional[str] = None) -> DashboardData:
        """Aggregate metrics. If tenant_id is None, returns process-wide."""

    def export_prometheus(self) -> str:
        """Export in Prometheus text format for scraping."""

    def export_opentelemetry(self) -> Dict[str, Any]:
        """Export as OpenTelemetry-compatible spans."""
```

---

## 6. Integration Points

### 6.1 How Components Wire Together

```
Developer Code:
    manager = TenantManager()
    engine = await manager.create_tenant("acme", config, keys)
    agent = engine.create_agent(name="support", system_prompt="...")
    session = agent.start_session(user_id="u123")
    response = await session.run("Help me")

Internal Flow:
    session.run(input)
        -> TenantContext.bind(engine.tenant_context)     # contextvars
        -> Gate 1: SafetyPolicy.check_input(input)       # safety
        -> Gate 2: PolicyEngine.evaluate_intent(input)    # policy
        -> Gate 3: CapabilitySet.has("model:*", "execute") # capability
        -> Gate 4: QuotaEnvelope.acquire_tokens(est)     # quota
        -> CostPredictor.predict(plan)                   # cost prediction
        -> LLMRouter.generate(messages, role)            # model call
            -> KeyVault.retrieve(provider)               # key resolution
            -> DataClassifier.enforce(level, model)      # data classification
            -> FairScheduler.enqueue(tenant_id, req)     # fair scheduling
        -> Gate 5: SafetyPolicy.check_output(response)   # output safety
        -> Gate 6: TenantAuditStream.append(entry)       # audit
        -> DecisionTracer.trace_model_selection(...)      # observability
        -> CostTracker.record(cost_record)               # cost tracking
        -> TenantDNA.update_from_session(metrics)        # DNA learning
```

### 6.2 Cascading Policy Cascade -- Novel Invention

Policies from all three layers merge using "strictest wins":

```python
# corteX/tenancy/cascade.py
class PolicyCascade:
    """Merge policies from Questo -> Developer -> User. Strictest wins."""

    def merge(self, questo: TenantConfig, developer: TenantConfig,
              user_overrides: Dict[str, Any]) -> ResolvedPolicy:
        """
        For each policy dimension:
        - Safety level: max(questo, developer) -- user cannot lower
        - Token budget: min(questo, developer, user) -- tightest wins
        - Blocked topics: union(questo, developer) -- cumulative
        - Allowed models: intersection(questo, developer) -- both must allow
        - Tool permissions: intersection(questo, developer) then user filter
        """
```

### 6.3 Self-Healing Isolation -- Novel Invention

```python
# corteX/tenancy/isolation_monitor.py
class IsolationMonitor:
    """Runtime detection and repair of isolation violations."""

    def check_memory_isolation(self, sessions: List[Session]) -> List[Violation]:
        """Detect if any two sessions share a memory backend reference."""

    def check_event_isolation(self, engines: List[Engine]) -> List[Violation]:
        """Detect if any EventBus subscriber leaks across engines."""

    def repair(self, violation: Violation) -> RepairResult:
        """Auto-repair: clone the shared resource into separate instances."""

    async def run_continuous(self, interval_seconds: float = 60.0) -> None:
        """Background task: periodic isolation health check."""
```

---

## 7. Innovation Highlights

| Innovation | What It Does | Why It Matters |
|------------|-------------|----------------|
| **Tenant DNA** | Learns usage patterns per tenant, pre-loads into new sessions | Faster cold start, smarter routing, anomaly detection from turn 1 |
| **Cascading Policy Cascade** | Merges 3-layer policies automatically (strictest wins) | Zero config for end-users, guaranteed compliance at every layer |
| **Self-Healing Isolation** | Detects and repairs global state leaks at runtime | Defense against regression -- even if a developer introduces a singleton, it gets caught |
| **Capability Attenuation** | Reduces agent capabilities as risk increases | Agent automatically becomes more careful with sensitive operations |
| **Cost Prediction** | Estimates total plan cost before execution | Enterprise budget approval workflow, prevents surprise bills |
| **Compliance as Code** | GDPR/HIPAA/SOC2 as machine-readable policy rules | Automated compliance enforcement, not just documentation |
| **Hash-Chained Audit** | Tamper-evident audit trail with SHA-256 chain | Proves audit integrity for SOC2 CC7.2 and legal proceedings |
| **Data Classification Flow** | Classification level can only escalate, never downgrade | Confidential input produces confidential output -- always |

---

## 8. Implementation Priority

### Phase 1: Foundation (Week 1) -- Critical Path
1. Build `corteX/tenancy/context.py` -- TenantContext with contextvars
2. Fix EventBus: convert to instance-level dict, remove @classmethod
3. Fix ContextBroker: remove module-level singleton
4. Fix tool registry: scope `_registered_tools` to Agent
5. Fix API key fallback: ValueError instead of os.environ
6. Wire TenantContext into Engine -> Agent -> Session chain

### Phase 2: Security (Week 2) -- Enterprise Requirement
7. Build `corteX/security/vault.py` -- KeyVault with Fernet encryption
8. Build `corteX/security/capabilities.py` -- CapabilitySet with attenuation
9. Build `corteX/security/classification.py` -- DataClassifier
10. Wire TenantConfig (replacing EnterpriseConfig) into Session runtime
11. Enforce ModelPolicy/ToolPolicy at LLMRouter and ToolExecutor

### Phase 3: Observability (Week 3) -- Production Visibility
12. Build `corteX/observability/tracer.py` -- DecisionTracer
13. Build `corteX/observability/cost.py` -- CostTracker + CostPredictor
14. Build `corteX/observability/metrics.py` -- MetricsCollector with Prometheus
15. Build `corteX/tenancy/quota.py` -- QuotaEnvelope with fair scheduling
16. Build `corteX/tenancy/dna.py` -- TenantDNA learning

### Phase 4: Advanced (Week 4) -- Competitive Moat
17. Build `corteX/tenancy/cascade.py` -- PolicyCascade (3-layer merge)
18. Build `corteX/tenancy/manager.py` -- TenantManager with GDPR erasure
19. Build `corteX/security/compliance.py` -- ComplianceEngine
20. Build `corteX/tenancy/isolation_monitor.py` -- Self-Healing Isolation
21. Build `corteX/security/attenuation.py` -- RiskAttenuator

**Total: ~21 modules, ~4 weeks for 1 engineer, zero external dependencies.**
