# corteX SDK -- System Description

> **SOC 2 Type 1 -- Trust Service Criteria: Security, Confidentiality, Processing Integrity**
>
> **Document ID**: SD-001
> **Version**: 1.0
> **Classification**: CONFIDENTIAL
> **Owner**: Questo Ltd. Engineering
> **Last Updated**: 2026-02-16
> **Review Cycle**: Annual or upon material change

---

## 1. Overview of Services

### 1.1 Nature of the System

corteX is an enterprise-grade Python SDK (`cortex-ai` on PyPI) that enables SaaS
developers to build reliable, goal-driven AI agent capabilities within their own
applications. The SDK is developed and maintained by Questo Ltd. ("Questo", "the
Company").

corteX replaces general-purpose AI agent frameworks (LangChain, CrewAI, AutoGen)
with a brain-inspired AI agent engine that provides:

- **Goal tracking and verification** at every processing step
- **Loop prevention** through state hashing and semantic drift detection
- **Adaptive synaptic weights** with Bayesian posterior updates
- **4-tier feedback integration** (Direct, User Insights, Enterprise, Global)
- **Multi-tenant isolation** with per-tenant configuration, policies, and data boundaries
- **100% on-premises capability** with zero required external cloud dependencies

### 1.2 Principal Users

| User Type | Description | Interaction Mode |
|-----------|-------------|-----------------|
| **SaaS Developers** | Software engineers who integrate `cortex-ai` into their applications via `pip install cortex-ai`. They configure agents, providers, safety policies, and tools. | Python SDK API |
| **Enterprise Administrators** | IT/security staff at developer organizations who configure tenant policies, compliance frameworks, model restrictions, and audit settings. | `TenantConfig` JSON files, `EnterpriseConfig` API |
| **End Users** | Customers of the SaaS applications built with corteX. They interact with AI agents through the developer's application interface. End users never interact directly with the SDK. | Indirect (via developer application) |

### 1.3 How the System Is Used

Developers integrate corteX into their applications using a four-step pattern:

```python
import cortex

engine = cortex.Engine(providers={"openai": {"api_key": "sk-..."}})
agent = engine.create_agent(name="support", system_prompt="You help users.")
session = agent.start_session(user_id="user_123")
response = await session.run("Help me with my order")
```

The SDK runs within the developer's own infrastructure (cloud VMs, on-premises
servers, or containerized environments). Questo distributes the SDK package but
does not host, operate, or have access to customer deployments.

---

## 2. Principal Service Commitments and System Requirements

### 2.1 Security Commitments

| Commitment | Implementation |
|-----------|---------------|
| API keys are never stored in plaintext | `KeyVault` encrypts all keys at rest using Fernet (AES-128-CBC + HMAC-SHA256) with a PBKDF2-derived 256-bit master key (600,000 iterations) |
| No hardcoded secrets in source code | License signing keys loaded exclusively from environment variables; no API keys, tokens, or credentials in the codebase |
| Prompt injection protection | `SafetyPolicy` includes 12 regex-based injection detection patterns applied to all inputs |
| Capability-based access control | `CapabilitySet` provides immutable, attenuatable permission tokens that can only shrink, never grow |
| Risk-based attenuation | `RiskAttenuator` automatically reduces agent capabilities as risk scores increase |
| Tamper-evident audit logging | `AuditLogger` produces SHA-256 hash-chained, append-only JSONL audit trails |
| Ed25519 license validation | Cryptographic license verification with no network dependency |

### 2.2 Confidentiality Commitments

| Commitment | Implementation |
|-----------|---------------|
| 4-level data classification | `DataClassifier` automatically classifies data as PUBLIC, INTERNAL, CONFIDENTIAL, or RESTRICTED |
| PII auto-detection | Regex-based detection of emails, phone numbers, SSNs, credit card numbers in both inputs and outputs |
| Secret detection | Automated detection of API keys, bearer tokens, passwords, AWS keys, GitHub tokens, OpenAI keys, private key headers |
| Data routing enforcement | INTERNAL+ data is blocked from cloud LLM providers; only local/on-prem models are permitted |
| Classification escalation only | Data classification can only escalate (never downgrade) through the processing pipeline |
| Per-tenant key isolation | Each tenant's API keys are encrypted with a tenant-specific derived key |

### 2.3 Processing Integrity Commitments

| Commitment | Implementation |
|-----------|---------------|
| Goal-driven execution | `GoalTracker` + `GoalDNA` verify every processing step against the original user goal |
| Loop prevention | `MultiResolutionLoopDetector` runs 4 parallel detection strategies (state hash, semantic, action pattern, temporal) |
| Drift detection | `DriftEngine` scores goal drift using 5 independent signals with graduated response |
| Input validation | `SafetyPolicy.check_input()` validates all user inputs before processing |
| Output validation | `SafetyPolicy.check_output()` validates all generated outputs before delivery |
| Compliance pre-checks | `ComplianceEngine` evaluates every action against active compliance frameworks (GDPR, HIPAA, SOC 2, ISO 27001) before execution |

---

## 3. Components of the System

### 3.1 Infrastructure

corteX is a **customer-deployed Python SDK**. There is no Questo-operated
production infrastructure that processes customer data.

| Component | Description |
|-----------|-------------|
| **Source Code Repository** | Private GitHub repository (`github.com/QuestoM/cortex-sdk`) |
| **CI/CD Pipeline** | GitHub Actions for automated testing (3,355+ unit tests), linting (ruff), type checking (mypy), and package building |
| **Package Distribution** | PyPI (`cortex-ai`) -- the only Questo-operated distribution channel |
| **Documentation** | MkDocs Material static site at `docs.cortex-ai.com` (97 pages) |
| **Development Machines** | Developer workstations with access controlled via GitHub SSO |

### 3.2 Software Components

The SDK consists of approximately 103+ modules organized into the following packages:

#### 3.2.1 SDK Core (`corteX/sdk.py`)

- `Engine` -- Provider management, agent factory, global configuration
- `Agent` -- Stateless agent template (system prompt, tools, config)
- `Session` -- Stateful runtime with full brain integration

#### 3.2.2 Session Architecture (`corteX/session/`)

25 mixin-based modules providing:

- Chat execution (`chat.py`)
- Agentic multi-step execution (`agentic.py`, `agentic_loop.py`, `agentic_planner.py`)
- Streaming (`stream.py`)
- Tool preparation and execution (`tools.py`, `prepare_tools.py`)
- Brain integration (`brain.py`)
- Learning and weight updates (`learning.py`, `learning_signals.py`)
- Sub-agent delegation (`subagent.py`)

#### 3.2.3 Brain Engine (`corteX/engine/`)

Neuroscience-inspired processing engine with 60+ modules:

- **Core Systems**: WeightEngine (7-category Bayesian weights), GoalTracker, GoalDNA, LoopDetector, DriftEngine, FeedbackEngine, PredictionEngine, PlasticityManager
- **Advanced Components**: ProactivePrediction, CrossModalAssociator, ContinuousCalibration, ColumnManager, ResourceHomunculus, AttentionSystem, ConceptGraph, CorticalMapReorganizer
- **Brain-LLM Bridge**: BrainStateInjector, BrainParameterResolver, InferenceHooks, StructuredOutput
- **Game Theory**: DualProcessRouter, ReputationSystem, NashRoutingOptimizer, ShapleyAttributor
- **Agentic Engine**: AgentLoop, ContextCompiler, PlanningEngine, ReflectionEngine, RecoveryEngine, PolicyEngine, SubAgentManager

#### 3.2.4 LLM Providers (`corteX/core/llm/`)

- `LLMRouter` -- Multi-provider routing with circuit breaking, rate limiting, and fallback
- `OpenAIProvider` -- OpenAI and Azure OpenAI compatible
- `GeminiAdapter` -- Google Gemini (via google-genai SDK)
- `AnthropicProvider` -- Anthropic Claude
- Local provider support (vLLM, Ollama, any OpenAI-compatible endpoint)
- `ModelRegistry` -- YAML-based model catalog with 8 roles
- `CognitiveClassifier` -- Zero-LLM task classification
- `CostTracker` -- Per-tenant cost tracking and budget enforcement

#### 3.2.5 Security (`corteX/security/`)

- `KeyVault` -- Fernet-encrypted (AES-128-CBC + HMAC-SHA256, PBKDF2-derived 256-bit key) in-memory API key store
- `CapabilitySet` -- Immutable, attenuatable capability tokens
- `RiskAttenuator` -- Risk-based capability reduction
- `DataClassifier` -- Automatic 4-level data classification with PII and secret detection
- `ComplianceEngine` -- GDPR, HIPAA, SOC 2, ISO 27001 enforcement
- `AuditLogger` -- Tamper-evident, hash-chained audit logging

#### 3.2.6 Enterprise (`corteX/enterprise/`)

- `TenantConfig` -- Per-tenant isolation (safety, models, tools, audit, license, compliance)
- `SafetyPolicy` -- Content filtering, PII detection, prompt injection protection
- `ModelPolicy` -- Model allowlisting/blocklisting, token limits, default-deny mode
- `ToolPolicy` -- Tool allowlisting/blocklisting, approval requirements, default-deny mode
- `LicenseManager` -- Ed25519 cryptographic license validation with offline grace period
- `AuditConfig` -- Audit logging configuration with retention policies

#### 3.2.7 Multi-Tenancy (`corteX/tenancy/`)

- `TenantContext` -- `contextvars`-based tenant propagation (3-layer model: Questo / Developer / End User)
- `TenantManager` -- Tenant lifecycle management
- `TenantDNA` -- Tenant identity hashing
- `QuotaManager` -- Per-tenant resource quotas

#### 3.2.8 Memory (`corteX/memory/`)

- Working Memory (current session state)
- Episodic Memory (past experiences)
- Semantic Memory (factual knowledge)
- `MemoryFabric` -- Unified retrieval interface
- File-based and Vertex AI Search drivers

#### 3.2.9 Context Management (`corteX/engine/cognitive/`)

- `CognitiveContextPipeline` -- 8-phase context assembly
- `ContextQualityEngine` -- 6-dimensional context scoring
- `StateFileManager` -- 3-layer compaction-proof persistent state

#### 3.2.10 Plugins (`corteX/plugins/`)

- Code Interpreter (sandboxed execution)
- WebEngine V2 (browser automation)
- Computer Use (desktop automation)

### 3.3 People

| Role | Responsibilities |
|------|-----------------|
| **Engineering Lead** | Architecture decisions, code review approval, security design |
| **SDK Engineers** | Feature development, bug fixes, unit/integration testing |
| **Security Engineer** | Security module development, vulnerability assessment, dependency audit |
| **QA/Release Engineer** | CI/CD pipeline maintenance, release management, PyPI publishing |
| **Documentation Engineer** | API reference, tutorials, compliance documentation |

All personnel with code commit access undergo security awareness training and
operate under the principle of least privilege for repository access.

### 3.4 Procedures

#### 3.4.1 Change Management

- All code changes require pull request review and approval before merge to main
- CI pipeline enforces: 3,355+ unit tests passing, ruff linting, mypy type checking
- Maximum 300 lines per file enforced by convention
- Semantic versioning (currently v1.0.0)
- Protected main branch; no direct pushes

#### 3.4.2 Incident Response

- Security vulnerabilities triaged within 24 hours of discovery
- Critical vulnerabilities patched and released within 72 hours
- Dependency vulnerability scanning via GitHub Dependabot
- Security advisories published via GitHub Security Advisories

#### 3.4.3 Access Control

- GitHub repository access restricted to authorized personnel via SSO
- PyPI publishing credentials stored in GitHub Secrets (encrypted at rest)
- No shared accounts; individual developer identities required
- Branch protection rules enforce review requirements

#### 3.4.4 Dependency Management

- Core dependencies minimized: `pydantic`, `numpy`, `python-dotenv`, `cryptography`
- LLM provider SDKs are optional extras (`pip install cortex-ai[openai]`)
- No Google Cloud hard dependencies -- everything works fully on-premises
- Dependency versions pinned with minimum version constraints

### 3.5 Data

#### 3.5.1 Data Flowing Through the System

| Data Category | Description | Classification | Handling |
|--------------|-------------|---------------|----------|
| **User Messages** | Natural language inputs from end users | Varies (auto-classified) | Validated by SafetyPolicy, classified by DataClassifier, routed based on classification level |
| **Agent Responses** | LLM-generated outputs | Varies (auto-classified) | Validated by SafetyPolicy.check_output(), PII-scanned before delivery |
| **API Keys** | LLM provider credentials | RESTRICTED | Fernet-encrypted (AES-128-CBC) in KeyVault with PBKDF2-derived 256-bit key, never logged, never persisted to disk |
| **Conversation History** | Session message history | CONFIDENTIAL | Stored in working memory, retention controlled by DataRetention policy |
| **Synaptic Weights** | Brain engine learning state | INTERNAL | Persisted locally per tenant, no cross-tenant access |
| **Audit Logs** | Security and operational events | INTERNAL | Hash-chained JSONL, configurable retention (default 90 days online, 365 archive) |
| **License Keys** | Ed25519 signed tokens | CONFIDENTIAL | Validated locally, no network transmission required |
| **Tool Execution Results** | Output from tools (code interpreter, browser, etc.) | Varies | Subject to same classification and policy enforcement as all other data |

#### 3.5.2 Data Classification Levels

| Level | Description | Allowed Destinations | Requirements |
|-------|-------------|---------------------|-------------|
| **PUBLIC** | Non-sensitive data | Any model (cloud or local) | None |
| **INTERNAL** | Business-internal data | Local/on-prem models only | Tenant configuration |
| **CONFIDENTIAL** | Sensitive data (PII detected) | Local/on-prem models only | Audit trail required |
| **RESTRICTED** | Highly sensitive (secrets, credentials) | Local/on-prem models only | Human approval + full audit trail |

#### 3.5.3 Data at Rest

The SDK itself does not operate persistent databases. Data at rest consists of:

- **Weight persistence files**: JSON files containing learned Bayesian weights
- **Audit log files**: Append-only JSONL with hash chains
- **Tenant configuration files**: JSON files with policy settings
- **Memory persistence**: Optional file-based storage for episodic/semantic memory
- **License state files**: JSON files with activation status and usage meters

All data at rest is stored within the customer's own infrastructure. Questo has
no access to customer data at rest.

#### 3.5.4 Data in Transit

- **SDK to LLM Providers**: HTTPS/TLS 1.2+ (enforced by provider SDKs)
- **SDK to Local Models**: Configurable; local network or loopback
- **Developer to PyPI**: HTTPS/TLS 1.3 for package installation
- **Developer to GitHub**: SSH or HTTPS for source code access

---

## 4. System Boundaries

### 4.1 In Scope

The following are within the SOC 2 examination boundary:

| Component | Justification |
|-----------|--------------|
| **corteX SDK source code** | Core product under examination |
| **Security modules** (`corteX/security/`) | KeyVault, CapabilitySet, RiskAttenuator, DataClassifier, ComplianceEngine, AuditLogger |
| **Enterprise configuration** (`corteX/enterprise/`) | TenantConfig, SafetyPolicy, ModelPolicy, ToolPolicy, LicenseManager |
| **Multi-tenancy isolation** (`corteX/tenancy/`) | TenantContext, contextvars-based propagation |
| **GitHub repository** | Source code management, change control, access management |
| **CI/CD pipeline** (GitHub Actions) | Automated testing, build, and release process |
| **PyPI package distribution** | The sole distribution channel for the SDK |
| **Development documentation** | Architecture, API reference, compliance documentation |
| **Development processes** | Change management, code review, incident response |

### 4.2 Out of Scope

The following are explicitly excluded from the examination boundary:

| Component | Justification |
|-----------|--------------|
| **Customer deployments** | Customers deploy and operate the SDK within their own infrastructure; Questo has no access to or responsibility for these environments |
| **LLM provider infrastructure** | OpenAI, Google, Anthropic operate their own platforms; covered by carved-out subservice organization treatment (Section 6) |
| **Customer applications** | The SaaS applications developers build using corteX are outside Questo's control |
| **End-user data** | Data processed through customer deployments is under customer control |
| **Optional FastAPI server** | An optional component for local REST API exposure; not a Questo-operated service |
| **Browser/Computer Use plugins** | Optional automation plugins that execute within the customer environment |
| **Customer network infrastructure** | Firewalls, VPNs, and network controls in customer environments |

---

## 5. Trust Service Criteria Coverage

### 5.1 Security (Common Criteria)

| CC Reference | Control | corteX Implementation |
|-------------|---------|----------------------|
| **CC1.1** | Control environment | Documented security policies, code review requirements, access controls |
| **CC2.1** | Information and communication | Security documentation, CLAUDE.md project guidelines, MkDocs documentation site |
| **CC3.1** | Risk assessment | Risk register (`docs/compliance/risk_register.md`), threat modeling in design |
| **CC4.1** | Monitoring activities | AuditLogger with hash-chained events, CI/CD test monitoring |
| **CC5.1** | Control activities | SafetyPolicy enforcement, DataClassifier checks, ComplianceEngine pre-checks |
| **CC6.1** | Logical access controls | KeyVault encryption, CapabilitySet permissions, GitHub SSO, branch protection |
| **CC6.2** | Access authentication | Ed25519 license validation, per-tenant KeyVault derivation |
| **CC6.3** | Access authorization | Capability-based access, ToolPolicy/ModelPolicy allowlisting, default-deny option |
| **CC6.6** | System boundary protection | Data classification enforcement, cloud routing blocks for sensitive data |
| **CC7.1** | System monitoring | AuditLogger event capture, CI/CD health checks |
| **CC7.2** | Security event monitoring | AuditLogger with 15+ event types, severity classification, tamper detection |
| **CC7.3** | Security incident evaluation | Incident response procedures, GitHub Security Advisories |
| **CC8.1** | Change management | Pull request reviews, CI gates, protected branches, semantic versioning |
| **CC9.1** | Risk mitigation | RiskAttenuator, DriftEngine, LoopDetector, multiple redundant safety layers |

### 5.2 Confidentiality

| CC Reference | Control | corteX Implementation |
|-------------|---------|----------------------|
| **C1.1** | Confidential information identification | DataClassifier with 4 classification levels, PII detection, secret detection |
| **C1.2** | Confidential information disposal | KeyVault.destroy() for GDPR erasure, configurable data retention, audit log rotation |

### 5.3 Processing Integrity

| CC Reference | Control | corteX Implementation |
|-------------|---------|----------------------|
| **PI1.1** | Processing completeness and accuracy | GoalTracker verification, SafetyPolicy input/output validation, ComplianceEngine pre-checks |
| **PI1.2** | Processing output accuracy | PopulationQualityEstimator, PredictionEngine surprise detection, output validation |
| **PI1.3** | Processing integrity monitoring | AuditLogger integrity verification (hash chain), DriftEngine monitoring |

---

## 6. Subservice Organizations

corteX integrates with third-party LLM providers via their respective APIs. These
are treated using the **carved-out method**: their controls are excluded from the
scope of this examination.

| Subservice Organization | Service Used | Integration Point | Data Transmitted |
|------------------------|-------------|-------------------|-----------------|
| **OpenAI, Inc.** | GPT-4o, o1/o3 series API | `OpenAIProvider` via `openai` SDK | Conversation messages, tool definitions (only PUBLIC-classified data unless customer explicitly configures otherwise) |
| **Google LLC / Vertex AI** | Gemini 3 Pro/Flash, Gemini 2.5 API | `GeminiAdapter` via `google-genai` SDK | Conversation messages, tool definitions (subject to same classification rules) |
| **Anthropic PBC** | Claude Opus, Sonnet, Haiku API | `AnthropicProvider` via `anthropic` SDK | Conversation messages, tool definitions (subject to same classification rules) |
| **GitHub, Inc.** | Source code hosting, CI/CD, package registry | Git, GitHub Actions | Source code, CI artifacts, package metadata |
| **Python Software Foundation** | PyPI package hosting | `pip install cortex-ai` | Built package distributions |

### 6.1 Controls Over Subservice Organizations

- **Data classification enforcement**: `DataClassifier` blocks INTERNAL, CONFIDENTIAL, and RESTRICTED data from being sent to cloud LLM providers
- **Provider selection controls**: `ModelPolicy` allows enterprise administrators to restrict which providers and models are permitted
- **Circuit breaker and rate limiting**: `LLMRouter` implements circuit breaking (automatic fallback after 3 failures) and rate limiting per provider
- **Cost tracking**: `CostTracker` enforces per-session and per-tenant budget limits
- **No sensitive data in transit to providers by default**: API keys are never sent as message content; `KeyVault.detect_leak()` scans outputs for accidentally exposed credentials

---

## 7. Complementary User Entity Controls (CUECs)

The effectiveness of corteX's controls depends on customers implementing the
following controls within their own environments. These CUECs are communicated
to customers via documentation and onboarding materials.

### CUEC-1: Secure API Key Management

**Requirement**: Customers must securely manage and rotate LLM provider API keys.
Keys must not be hardcoded in application source code or stored in version control.

**Rationale**: corteX's `KeyVault` encrypts keys in memory, but customers are
responsible for the initial secure provisioning of keys to the SDK.

### CUEC-2: Network Security

**Requirement**: Customers must implement appropriate network security controls
(firewalls, TLS enforcement, network segmentation) for their deployment environment.

**Rationale**: corteX runs within the customer's infrastructure. Network-level
security is the customer's responsibility.

### CUEC-3: Access Control to Deployment Environment

**Requirement**: Customers must restrict access to servers and environments where
corteX is deployed, using role-based access control and authentication.

**Rationale**: The SDK's security controls protect against application-layer threats.
Infrastructure access control is the customer's responsibility.

### CUEC-4: Tenant Configuration Review

**Requirement**: Customers must review and appropriately configure `TenantConfig`
settings including safety policies, model restrictions, tool restrictions, and
compliance framework selections.

**Rationale**: corteX provides configurable security policies, but customers must
enable and tune them to match their risk profile and compliance requirements.

### CUEC-5: Audit Log Monitoring and Retention

**Requirement**: Customers must configure audit log destinations, monitor audit
events for security anomalies, and maintain audit log retention in accordance
with their compliance requirements.

**Rationale**: corteX generates comprehensive audit events, but customers must
configure storage, monitoring, and alerting.

### CUEC-6: Data Classification Policy

**Requirement**: Customers must establish data classification policies and
configure `DataClassifier` tenant policies to match their organization's data
sensitivity definitions.

**Rationale**: The SDK provides automatic PII and secret detection, but
customers may need to add organization-specific classification rules.

### CUEC-7: LLM Provider Agreements

**Requirement**: Customers using cloud LLM providers must establish appropriate
data processing agreements (DPAs), business associate agreements (BAAs for HIPAA),
or equivalent contractual protections with each provider.

**Rationale**: corteX routes requests to LLM providers but does not establish
contractual relationships on behalf of customers.

### CUEC-8: Incident Response Plan

**Requirement**: Customers must maintain an incident response plan that includes
procedures for AI-related security incidents (prompt injection attacks, data
leakage through AI outputs, model misuse).

**Rationale**: corteX provides detection and prevention controls, but customers
must have response procedures for when incidents occur.

### CUEC-9: SDK Version Management

**Requirement**: Customers must keep their corteX SDK installation updated to
the latest stable version to receive security patches and improvements.

**Rationale**: Security fixes are distributed via PyPI package updates. Customers
running outdated versions may be exposed to known vulnerabilities.

### CUEC-10: Compliance Framework Activation

**Requirement**: Customers subject to specific compliance requirements (GDPR,
HIPAA, SOC 2, ISO 27001) must activate the corresponding compliance framework
in their `TenantConfig.compliance` setting.

**Rationale**: corteX includes built-in compliance enforcement for multiple
frameworks, but enforcement is only active when the customer explicitly enables
the relevant framework(s).

---

## 8. Relevant Aspects of the Control Environment

### 8.1 Integrity and Ethical Values

- Code of conduct requires all developers to follow secure coding practices
- No hardcoded secrets policy enforced by code review and automated scanning
- Maximum file size (300 lines) promotes code readability and review quality
- All public functions require type hints for correctness verification

### 8.2 Commitment to Competence

- Engineering team selected for expertise in Python, AI/ML, security, and distributed systems
- Continuous learning through industry conference participation and security training
- Regular code review ensures knowledge transfer and quality standards

### 8.3 Management Philosophy and Operating Style

- **On-Prem First**: Every feature must work without internet connectivity
- **Security as DNA**: Agents never call external tools/APIs unless the tenant explicitly enables them
- **SDK-First**: Clean, documented Python API that developers can understand and trust
- **Test-Driven**: 3,355+ unit tests maintained at 100% passing rate

### 8.4 Organizational Structure

Questo Ltd. operates as a focused SDK product company. The engineering team is
responsible for all aspects of development, security, testing, documentation,
and release management.

---

## 9. Risk Assessment Summary

Key risks identified and mitigated:

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| Prompt injection attack | High | High | SafetyPolicy with 12 injection patterns, input validation |
| API key leakage | Medium | Critical | KeyVault encryption, detect_leak() scanning, no env fallback |
| Goal drift / infinite loops | Medium | Medium | GoalDNA, DriftEngine, MultiResolutionLoopDetector, AdaptiveBudget |
| Unauthorized data exposure to cloud models | Medium | High | DataClassifier enforcement, INTERNAL+ data blocked from cloud |
| Dependency supply chain attack | Low | High | Minimal dependencies (4 core), version pinning, Dependabot monitoring |
| License bypass | Low | Medium | Ed25519 cryptographic validation, no network-only checks |
| Cross-tenant data leakage | Low | Critical | TenantContext via contextvars, per-tenant KeyVault, isolated PluginRegistry |
| Audit log tampering | Low | High | SHA-256 hash-chained entries, append-only storage, integrity verification |

---

## 10. Complementary Information

### 10.1 Package Details

- **Package Name**: `cortex-ai`
- **Current Version**: 1.0.0
- **Python Compatibility**: 3.10, 3.11, 3.12, 3.13
- **License**: Proprietary (LicenseRef-Proprietary)
- **Core Dependencies**: pydantic (>=2.0), numpy (>=1.24), python-dotenv (>=1.0), cryptography (>=41.0)

### 10.2 Test Coverage

- **Unit Tests**: 3,355+ tests across 33+ test files
- **Test Framework**: pytest with pytest-asyncio
- **Coverage Target**: 80% minimum (enforced by CI)
- **Integration Tests**: Live API tests (marked, deselectable)

### 10.3 Documentation

- **97 pages** organized as: Getting Started (6), Tutorials (6), How-To Guides (14), Concepts (26), Enterprise (8), API Reference (35), Root (2)
- **Framework**: MkDocs Material with mkdocstrings
- **Domain**: docs.cortex-ai.com

---

*This system description is maintained under version control alongside the corteX
SDK source code. Changes to the system that materially affect this description
trigger a review and update as part of the change management process.*
