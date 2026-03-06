# corteX SDK -- Data Flow Diagram

> **SOC 2 Type 1 -- GAP-S15: Data Flow Diagram**
>
> **Document ID**: DFD-001
> **Version**: 1.0
> **Classification**: CONFIDENTIAL
> **Owner**: Questo Ltd. Engineering
> **Last Updated**: 2026-02-16
> **Review Cycle**: Annual or upon material change

---

## 1. Overview

This document maps every data flow within the corteX SDK, from user input to
final response. Each processing step is annotated with the applicable SOC 2
Common Criteria (CC) reference, data classification checkpoints, PII detection
points, and audit logging triggers.

---

## 2. Primary Request Processing Flow

This diagram traces a single user request through the complete corteX processing
pipeline.

```mermaid
graph TD
    classDef input fill:#FFF3CD,stroke:#856404,color:#333,stroke-width:2px
    classDef safety fill:#F8D7DA,stroke:#721C24,color:#333,stroke-width:2px
    classDef brain fill:#E8D5F5,stroke:#6F42C1,color:#333,stroke-width:2px
    classDef llm fill:#CCE5FF,stroke:#004085,color:#333,stroke-width:2px
    classDef output fill:#D4EDDA,stroke:#155724,color:#333,stroke-width:2px
    classDef audit fill:#E2E3E5,stroke:#383D41,color:#333,stroke-width:2px
    classDef memory fill:#D1ECF1,stroke:#0C5460,color:#333,stroke-width:2px

    START["User Input<br/>(Natural language message)"]:::input

    subgraph PHASE_1 ["Phase 1: Input Validation & Classification"]
        direction TB
        SAFETY_INPUT["SafetyPolicy.check_input()<br/>-- Prompt injection detection (12 patterns)<br/>-- Blocked content patterns<br/>-- Blocked topics<br/><b>CC5.1, CC6.6, PI1.1</b>"]:::safety
        PII_INPUT["DataClassifier.classify()<br/>-- PII detection (email, phone, SSN, CC)<br/>-- Secret detection (7 patterns)<br/>-- Tenant keyword escalation<br/><b>C1.1, CC6.6</b>"]:::safety
        COMPLIANCE_PRE["ComplianceEngine.pre_check()<br/>-- GDPR: consent + data minimization<br/>-- HIPAA: encryption + BAA check<br/>-- SOC2: access controls + logging<br/>-- ISO27001: activity logging<br/><b>CC5.1, CC9.1</b>"]:::safety
        AUDIT_INPUT["AuditLogger.log_event()<br/>event_type: session_start<br/>Captures: user_id, session_id,<br/>classification_level, tenant_id<br/><b>CC4.1, CC7.1, CC7.2</b>"]:::audit
    end

    subgraph PHASE_2 ["Phase 2: Context Assembly & Brain Processing"]
        direction TB
        MEMORY_RETRIEVE["MemoryFabric.get_relevant_context()<br/>-- Working Memory (session state)<br/>-- Episodic Memory (past experiences)<br/>-- Semantic Memory (factual knowledge)<br/><b>PI1.1</b>"]:::memory
        CTX_COMPILE["ContextCompiler.compile()<br/>-- 4-Zone Assembly:<br/>  System (12%) | Persistent (8%)<br/>  Working (40%) | Recent (40%)<br/><b>PI1.1</b>"]:::brain
        GOAL_CHECK["GoalTracker + GoalDNA<br/>-- Goal alignment verification<br/>-- Token fingerprint drift detection<br/>-- Loop detection (4 detectors)<br/><b>PI1.1, PI1.3</b>"]:::brain
        BRAIN_STATE["BrainStateInjector<br/>-- Behavioral weights to prompt<br/>-- Active column context<br/>-- Surprise/prediction context<br/><b>PI1.1</b>"]:::brain
        PARAM_RESOLVE["BrainParameterResolver<br/>-- temperature, top_p, top_k<br/>-- max_tokens, penalties<br/>-- thinking_budget, stop_sequences<br/><b>PI1.1</b>"]:::brain
    end

    subgraph PHASE_3 ["Phase 3: LLM Routing & Inference"]
        direction TB
        CAPS_CHECK["CapabilitySet.has()<br/>-- Permission verification<br/>-- Expired capability check<br/><b>CC6.3</b>"]:::safety
        RISK_CHECK["RiskAttenuator.compute_risk()<br/>-- Tool risk + data level + confidence + drift<br/>-- Capability attenuation if risk > 0.3<br/>-- Human approval required if risk > 0.8<br/><b>CC6.3, CC9.1</b>"]:::safety
        DATA_ENFORCE["DataClassifier.enforce()<br/>-- PUBLIC: any model<br/>-- INTERNAL: local only<br/>-- CONFIDENTIAL: local + audit<br/>-- RESTRICTED: local + approval + audit<br/><b>C1.1, CC6.6</b>"]:::safety
        KEYVAULT_DECRYPT["KeyVault.retrieve()<br/>-- Fernet decryption (AES-128-CBC)<br/>-- PBKDF2-derived 256-bit master key<br/>-- Key held in memory only<br/>-- No env fallback<br/><b>CC6.1, CC6.2</b>"]:::safety
        LLM_ROUTE["LLMRouter.generate()<br/>-- Cognitive classification (zero-LLM)<br/>-- Weight-based model selection<br/>-- Circuit breaker check<br/>-- Rate limiter check<br/>-- Budget enforcement<br/><b>PI1.1, CC9.1</b>"]:::llm
        LLM_CALL["LLM Provider API Call<br/>-- HTTPS/TLS 1.2+<br/>-- Retry with exponential backoff<br/>-- Automatic fallback on failure<br/><b>CC6.6</b>"]:::llm
        COST_RECORD["CostTracker.record_call()<br/>-- Per-tenant cost tracking<br/>-- Session-level budgets<br/><b>CC7.1</b>"]:::audit
        AUDIT_LLM["AuditLogger.log_event()<br/>event_type: llm_call<br/>Captures: provider, model, tokens,<br/>latency, cost, routing_reason<br/><b>CC4.1, CC7.1, CC7.2</b>"]:::audit
    end

    subgraph PHASE_4 ["Phase 4: Response Validation & Output"]
        direction TB
        SAFETY_OUTPUT["SafetyPolicy.check_output()<br/>-- PII detection in output<br/>-- Blocked content patterns<br/>-- Blocked topics<br/><b>CC5.1, PI1.2, C1.1</b>"]:::safety
        LEAK_DETECT["KeyVault.detect_leak()<br/>-- Scan output for API keys<br/>-- Full decrypted key comparison<br/><b>CC6.1, C1.1</b>"]:::safety
        QUALITY_CHECK["PopulationQualityEstimator<br/>-- Ensemble quality scoring<br/>-- Agreement analysis<br/><b>PI1.2</b>"]:::brain
        SURPRISE_CHECK["PredictionEngine<br/>-- Actual vs predicted comparison<br/>-- Bayesian surprise signal<br/><b>PI1.2, PI1.3</b>"]:::brain
    end

    subgraph PHASE_5 ["Phase 5: Learning & Persistence"]
        direction TB
        PLASTICITY["PlasticityManager<br/>-- Hebbian learning (LTP/LTD)<br/>-- Homeostatic regulation<br/>-- Weight consolidation<br/><b>PI1.3</b>"]:::brain
        WEIGHT_UPDATE["WeightEngine update<br/>-- 7 weight categories<br/>-- Bayesian posterior updates<br/><b>PI1.3</b>"]:::brain
        MEMORY_STORE["Memory consolidation<br/>-- Working memory update<br/>-- Episodic memory formation<br/>-- Context summarization (L2/L3)<br/><b>PI1.1</b>"]:::memory
        AUDIT_RESPONSE["AuditLogger.log_event()<br/>event_type: session_response<br/>Captures: response_quality,<br/>goal_alignment, drift_score<br/><b>CC4.1, CC7.1</b>"]:::audit
    end

    RESPONSE["Validated Response<br/>(content + metadata + artifacts)"]:::output

    %% Main flow
    START --> SAFETY_INPUT
    SAFETY_INPUT -->|"Allowed"| PII_INPUT
    SAFETY_INPUT -->|"BLOCKED<br/>(injection detected)"| AUDIT_INPUT
    PII_INPUT --> COMPLIANCE_PRE
    COMPLIANCE_PRE -->|"Allowed"| AUDIT_INPUT
    COMPLIANCE_PRE -->|"BLOCKED<br/>(compliance violation)"| AUDIT_INPUT
    AUDIT_INPUT --> MEMORY_RETRIEVE

    MEMORY_RETRIEVE --> CTX_COMPILE
    CTX_COMPILE --> GOAL_CHECK
    GOAL_CHECK --> BRAIN_STATE
    BRAIN_STATE --> PARAM_RESOLVE

    PARAM_RESOLVE --> CAPS_CHECK
    CAPS_CHECK --> RISK_CHECK
    RISK_CHECK --> DATA_ENFORCE
    DATA_ENFORCE -->|"Classification allows<br/>selected provider"| KEYVAULT_DECRYPT
    DATA_ENFORCE -->|"BLOCKED<br/>(sensitive data,<br/>cloud provider)"| AUDIT_LLM
    KEYVAULT_DECRYPT --> LLM_ROUTE
    LLM_ROUTE --> LLM_CALL
    LLM_CALL --> COST_RECORD
    COST_RECORD --> AUDIT_LLM

    AUDIT_LLM --> SAFETY_OUTPUT
    SAFETY_OUTPUT -->|"Allowed"| LEAK_DETECT
    SAFETY_OUTPUT -->|"BLOCKED<br/>(PII in output)"| AUDIT_RESPONSE
    LEAK_DETECT -->|"No leak"| QUALITY_CHECK
    LEAK_DETECT -->|"LEAK DETECTED<br/>(API key in output)"| AUDIT_RESPONSE
    QUALITY_CHECK --> SURPRISE_CHECK

    SURPRISE_CHECK --> PLASTICITY
    PLASTICITY --> WEIGHT_UPDATE
    WEIGHT_UPDATE --> MEMORY_STORE
    MEMORY_STORE --> AUDIT_RESPONSE
    AUDIT_RESPONSE --> RESPONSE

    style PHASE_1 fill:#FFF5F5,stroke:#721C24,stroke-width:3px
    style PHASE_2 fill:#F5F0FF,stroke:#6F42C1,stroke-width:2px
    style PHASE_3 fill:#F0F7FF,stroke:#004085,stroke-width:2px
    style PHASE_4 fill:#F0FFF4,stroke:#155724,stroke-width:2px
    style PHASE_5 fill:#FAFAFA,stroke:#383D41,stroke-width:2px
```

---

## 3. Tool Execution Data Flow

When the LLM requests a tool call, a secondary data flow is triggered.

```mermaid
graph TD
    classDef tool fill:#D1ECF1,stroke:#0C5460,color:#333,stroke-width:2px
    classDef safety fill:#F8D7DA,stroke:#721C24,color:#333,stroke-width:2px
    classDef audit fill:#E2E3E5,stroke:#383D41,color:#333,stroke-width:2px

    LLM_RESP["LLM Response<br/>(contains tool_call request)"]:::tool

    TOOL_POLICY["ToolPolicy.is_tool_allowed()<br/>-- Allowlist / blocklist check<br/>-- Default-deny mode support<br/><b>CC6.3</b>"]:::safety

    CAPS_TOOL["CapabilitySet.has()<br/>resource: tool:{name}<br/>action: execute<br/><b>CC6.3</b>"]:::safety

    REPUTATION["ReputationSystem<br/>-- Tool trust score check<br/>-- Quarantine if untrusted<br/><b>CC9.1</b>"]:::safety

    POLICY_APPROVAL["PolicyEngine guardrails<br/>-- IntentGuard<br/>-- ToolApproval<br/>-- ToolGuide<br/><b>CC5.1, CC6.3</b>"]:::safety

    TOOL_EXEC["ToolExecutor.execute()<br/>-- Timeout enforcement<br/>-- Error handling<br/>-- Sandboxed execution<br/><b>PI1.1</b>"]:::tool

    AUDIT_TOOL["AuditLogger.log_event()<br/>event_type: tool_execution<br/>Captures: tool_name, args,<br/>result, duration, outcome<br/><b>CC7.1, CC7.2</b>"]:::audit

    DATA_CLASS_RESULT["DataClassifier.classify()<br/>-- Classify tool output<br/>-- PII/secret scan<br/><b>C1.1</b>"]:::safety

    TOOL_RESULT["Tool Result<br/>(fed back to LLM<br/>for next turn)"]:::tool

    LLM_RESP --> TOOL_POLICY
    TOOL_POLICY -->|"Allowed"| CAPS_TOOL
    TOOL_POLICY -->|"BLOCKED"| AUDIT_TOOL
    CAPS_TOOL -->|"Permitted"| REPUTATION
    CAPS_TOOL -->|"DENIED"| AUDIT_TOOL
    REPUTATION -->|"Trusted"| POLICY_APPROVAL
    REPUTATION -->|"QUARANTINED"| AUDIT_TOOL
    POLICY_APPROVAL -->|"Approved"| TOOL_EXEC
    POLICY_APPROVAL -->|"BLOCKED / NEEDS HUMAN APPROVAL"| AUDIT_TOOL
    TOOL_EXEC --> AUDIT_TOOL
    AUDIT_TOOL --> DATA_CLASS_RESULT
    DATA_CLASS_RESULT --> TOOL_RESULT
```

---

## 4. Data at Rest -- Memory Architecture

```mermaid
graph LR
    classDef hot fill:#F8D7DA,stroke:#721C24,color:#333,stroke-width:2px
    classDef warm fill:#FFF3CD,stroke:#856404,color:#333,stroke-width:2px
    classDef cold fill:#CCE5FF,stroke:#004085,color:#333,stroke-width:2px
    classDef persist fill:#E2E3E5,stroke:#383D41,color:#333,stroke-width:2px

    subgraph IN_MEMORY ["In-Memory (Session Lifetime)"]
        direction TB
        WORKING["Working Memory<br/>Current session state<br/>Recent messages<br/><i>Classification: inherits from content</i>"]:::hot
        BRAIN_WEIGHTS["Brain Weights<br/>7-category Bayesian weights<br/><i>Classification: INTERNAL</i>"]:::hot
        AUDIT_BUFFER["Audit Buffer<br/>Last 10,000 entries<br/>Hash-chained<br/><i>Classification: INTERNAL</i>"]:::hot
    end

    subgraph CONTEXT_TIERS ["Context Window Management"]
        direction TB
        HOT["Hot Context<br/>(Recent turns)<br/>Full messages<br/><b>CC7.1</b>"]:::hot
        WARM["Warm Context<br/>(Older turns)<br/>L2 LLM summaries<br/><b>CC7.1</b>"]:::warm
        COLD["Cold Context<br/>(Distant history)<br/>L3 JSON digests<br/><b>CC7.1</b>"]:::cold
    end

    subgraph PERSISTENT ["Persistent Storage (Customer Filesystem)"]
        direction TB
        WEIGHT_FILES["weight_state.json<br/>Learned Bayesian posteriors<br/><i>Classification: INTERNAL</i><br/><b>CC6.1</b>"]:::persist
        AUDIT_FILES["audit_{tenant}_{date}.jsonl<br/>Hash-chained audit entries<br/>Daily rotation, 50MB max<br/><i>Classification: INTERNAL</i><br/><b>CC4.1, CC7.2</b>"]:::persist
        CONFIG_FILES["tenant_config.json<br/>Safety, model, tool policies<br/><i>Classification: CONFIDENTIAL</i><br/><b>CC6.1</b>"]:::persist
        LICENSE_STATE["license_state.json<br/>Activation + usage meters<br/><i>Classification: CONFIDENTIAL</i><br/><b>CC6.2</b>"]:::persist
        EPISODE_FILES["episode_store/<br/>Past experience summaries<br/><i>Classification: inherits</i><br/><b>C1.2</b>"]:::persist
    end

    WORKING -->|"Periodic<br/>summarization"| HOT
    HOT -->|"L2 compression"| WARM
    WARM -->|"L3 compression"| COLD

    BRAIN_WEIGHTS -->|"Persistence"| WEIGHT_FILES
    AUDIT_BUFFER -->|"Append-only"| AUDIT_FILES
    WORKING -->|"Episode formation"| EPISODE_FILES

    style IN_MEMORY fill:#FFF5F5,stroke:#721C24,stroke-width:2px
    style CONTEXT_TIERS fill:#FFFEF0,stroke:#856404,stroke-width:2px
    style PERSISTENT fill:#F5F5F5,stroke:#383D41,stroke-width:2px
```

### Data Retention Policy

| Data Type | Default Retention | Configurable | CC Reference |
|-----------|------------------|-------------|-------------|
| Working Memory | Session lifetime | Via `DataRetention` enum | C1.2 |
| Audit Logs (online) | 90 days | `AuditConfig.retention_days` | CC7.2, C1.2 |
| Audit Logs (archive) | 365 days | Configurable | CC7.2, C1.2 |
| Weight State | Indefinite (until overwritten) | `weight_persistence_path` | N/A |
| License State | Indefinite (until deactivation) | `persistence_path` | CC6.2 |
| Episodic Memory | Configurable | Tenant policy | C1.2 |
| Tenant Config | Indefinite (until tenant removal) | Manual | CC6.1 |

---

## 5. Data in Transit -- LLM Provider Communication

```mermaid
sequenceDiagram
    participant App as Customer App
    participant SDK as corteX SDK
    participant DC as DataClassifier
    participant KV as KeyVault
    participant RT as LLMRouter
    participant CB as CircuitBreaker
    participant RL as RateLimiter
    participant LLM as LLM Provider API

    App->>SDK: session.run("user message")
    SDK->>DC: classify(user_message)
    DC-->>SDK: ClassificationResult(level=CONFIDENTIAL, pii=["email"])

    alt Data level = CONFIDENTIAL + Cloud provider
        SDK->>SDK: BLOCKED -- reroute to local model
        Note over SDK: CC6.6, C1.1: Sensitive data<br/>cannot go to cloud
    end

    SDK->>KV: retrieve("openai")
    Note over KV: Fernet (AES-128-CBC) decrypt<br/>PBKDF2-derived 256-bit key<br/>CC6.1: Encryption at rest
    KV-->>SDK: decrypted_api_key (in-memory only)

    SDK->>CB: is_available("openai")
    CB-->>SDK: true (circuit closed)

    SDK->>RL: acquire("openai")
    RL-->>SDK: true (under rate limit)

    SDK->>RT: generate(messages, model, params)
    RT->>LLM: POST /chat/completions<br/>Authorization: Bearer {key}<br/>Content-Type: application/json<br/>TLS 1.2+

    Note over RT,LLM: CC6.6: Encrypted in transit<br/>Only PUBLIC-classified data<br/>(unless tenant overrides)

    LLM-->>RT: HTTP 200 + JSON response

    RT->>SDK: LLMResponse(content, usage, model)
    SDK->>DC: classify(response_content)
    DC-->>SDK: ClassificationResult(level=PUBLIC)
    SDK->>KV: detect_leak(response_content)
    Note over KV: C1.1: Scan for credential<br/>leakage in LLM output
    KV-->>SDK: None (no leak detected)

    SDK-->>App: Response(content, metadata)
```

---

## 6. Audit Logging Data Flow

```mermaid
graph TD
    classDef event fill:#FFF3CD,stroke:#856404,color:#333,stroke-width:2px
    classDef process fill:#CCE5FF,stroke:#004085,color:#333,stroke-width:2px
    classDef store fill:#E2E3E5,stroke:#383D41,color:#333,stroke-width:2px
    classDef verify fill:#D4EDDA,stroke:#155724,color:#333,stroke-width:2px

    subgraph EVENT_SOURCES ["Audit Event Sources"]
        direction TB
        E1["Session Events<br/>session_start, session_end<br/><b>CC7.1</b>"]:::event
        E2["LLM Call Events<br/>llm_call, llm_failure<br/><b>CC7.1, CC7.2</b>"]:::event
        E3["Tool Events<br/>tool_execution, tool_failure<br/><b>CC7.1, CC7.2</b>"]:::event
        E4["Security Events<br/>security_block, input_validation<br/><b>CC7.2</b>"]:::event
        E5["Weight Events<br/>weight_change<br/><b>CC7.1</b>"]:::event
        E6["Policy Events<br/>policy_enforcement, compliance_check<br/><b>CC5.1, CC7.2</b>"]:::event
        E7["Data Events<br/>data_classification, data_access<br/><b>C1.1, CC7.1</b>"]:::event
        E8["Goal Events<br/>goal_drift, goal_completion<br/><b>PI1.3</b>"]:::event
    end

    subgraph AUDIT_PIPELINE ["Audit Processing Pipeline"]
        direction TB
        FILTER["Event Filter<br/>-- AuditConfig determines<br/>  which events to log<br/>-- log_messages, log_tool_calls,<br/>  log_weight_changes, log_model_routing"]:::process
        ENRICH["Entry Enrichment<br/>-- Add tenant_id, session_id, user_id<br/>-- Add timestamp (UTC)<br/>-- Assign severity level<br/>-- Assign sequence number"]:::process
        HASH_CHAIN["Hash Chain<br/>-- SHA-256(entry_data + previous_hash)<br/>-- Genesis hash for first entry<br/>-- Tamper-evident linking<br/><b>CC4.1, CC7.2</b>"]:::process
    end

    subgraph AUDIT_STORAGE ["Audit Storage"]
        direction TB
        MEMORY_BUF["In-Memory Buffer<br/>Ring buffer (10,000 entries)<br/>Fast query support"]:::store
        JSONL_FILE["JSONL File<br/>audit_{tenant}_{date}.jsonl<br/>Append-only writes<br/>Daily + size rotation (50MB)"]:::store
        STRUCT_LOG["Structured Logger<br/>Python logging backend<br/>corteX.security.audit"]:::store
        CALLBACK["Optional Callback<br/>Custom event handler<br/>(webhook, SIEM, etc.)"]:::store
    end

    subgraph AUDIT_OPS ["Audit Operations"]
        direction TB
        QUERY["query_logs()<br/>Filter by: time, event_type,<br/>user_id, session_id, severity<br/><b>CC4.2</b>"]:::verify
        VERIFY["verify_integrity()<br/>Re-compute full hash chain<br/>Detect any tampering<br/><b>CC7.2</b>"]:::verify
        EXPORT["export_logs()<br/>JSON or CSV format<br/>For compliance auditors<br/><b>CC4.2</b>"]:::verify
        ROTATE["rotate_and_archive()<br/>Online retention: 90 days<br/>Archive retention: 365 days<br/>Delete beyond archive<br/><b>C1.2</b>"]:::verify
    end

    E1 --> FILTER
    E2 --> FILTER
    E3 --> FILTER
    E4 --> FILTER
    E5 --> FILTER
    E6 --> FILTER
    E7 --> FILTER
    E8 --> FILTER

    FILTER --> ENRICH
    ENRICH --> HASH_CHAIN

    HASH_CHAIN --> MEMORY_BUF
    HASH_CHAIN --> JSONL_FILE
    HASH_CHAIN --> STRUCT_LOG
    HASH_CHAIN --> CALLBACK

    MEMORY_BUF --> QUERY
    MEMORY_BUF --> VERIFY
    MEMORY_BUF --> EXPORT
    JSONL_FILE --> ROTATE

    style EVENT_SOURCES fill:#FFFEF0,stroke:#856404,stroke-width:2px
    style AUDIT_PIPELINE fill:#F0F7FF,stroke:#004085,stroke-width:2px
    style AUDIT_STORAGE fill:#F5F5F5,stroke:#383D41,stroke-width:2px
    style AUDIT_OPS fill:#F0FFF4,stroke:#155724,stroke-width:2px
```

---

## 7. Data Classification Decision Tree

```mermaid
graph TD
    classDef check fill:#FFF3CD,stroke:#856404,color:#333,stroke-width:2px
    classDef level fill:#CCE5FF,stroke:#004085,color:#333,stroke-width:2px
    classDef action fill:#F8D7DA,stroke:#721C24,color:#333,stroke-width:2px

    INPUT["Input Data"]:::check

    SECRET_CHECK{"Secrets detected?<br/>(API keys, tokens,<br/>passwords, private keys)"}:::check
    PII_CHECK{"PII detected?<br/>(email, phone,<br/>SSN, credit card)"}:::check
    TENANT_KW{"Tenant keyword<br/>escalation?"}:::check
    DEFAULT_CHECK{"Tenant default<br/>level set?"}:::check

    RESTRICTED["RESTRICTED<br/>Local models only<br/>Human approval required<br/>Full audit trail<br/><b>CC6.6, C1.1</b>"]:::action
    CONFIDENTIAL["CONFIDENTIAL<br/>Local models only<br/>Audit trail required<br/><b>CC6.6, C1.1</b>"]:::level
    INTERNAL["INTERNAL<br/>Local models only<br/><b>CC6.6</b>"]:::level
    PUBLIC["PUBLIC<br/>Any model allowed<br/><b>No restrictions</b>"]:::level

    ESCALATE["Classification can<br/>only ESCALATE<br/>(never downgrade)<br/><b>C1.1</b>"]:::check

    INPUT --> SECRET_CHECK
    SECRET_CHECK -->|"Yes"| RESTRICTED
    SECRET_CHECK -->|"No"| PII_CHECK
    PII_CHECK -->|"Yes"| CONFIDENTIAL
    PII_CHECK -->|"No"| TENANT_KW
    TENANT_KW -->|"Yes (escalation level)"| ESCALATE
    TENANT_KW -->|"No"| DEFAULT_CHECK
    DEFAULT_CHECK -->|"Yes (tenant default)"| ESCALATE
    DEFAULT_CHECK -->|"No"| PUBLIC
    ESCALATE -->|"max(current, new)"| CONFIDENTIAL

    style RESTRICTED fill:#F8D7DA,stroke:#721C24,stroke-width:3px
    style CONFIDENTIAL fill:#FFE0B2,stroke:#E65100,stroke-width:2px
    style INTERNAL fill:#FFF3CD,stroke:#856404,stroke-width:2px
    style PUBLIC fill:#D4EDDA,stroke:#155724,stroke-width:2px
```

---

## 8. PII Detection Points Summary

PII detection occurs at multiple checkpoints in the data flow:

| Checkpoint | Component | Direction | Action on Detection | CC Reference |
|-----------|-----------|-----------|-------------------|-------------|
| **Input validation** | `SafetyPolicy.check_input()` | Inbound | Log warning (does not block input PII by default) | CC5.1, PI1.1 |
| **Data classification** | `DataClassifier.classify()` | Inbound | Escalate to CONFIDENTIAL; restrict to local models | C1.1, CC6.6 |
| **Output validation** | `SafetyPolicy.check_output()` | Outbound | **BLOCK** response delivery; return error | C1.1, PI1.2 |
| **Credential leak scan** | `KeyVault.detect_leak()` | Outbound | **BLOCK** response; log CRITICAL security event | CC6.1, C1.1 |
| **Tool output classification** | `DataClassifier.classify()` | Tool result | Escalate classification for tool outputs | C1.1 |
| **Compliance pre-check** | `ComplianceEngine` (GDPR/HIPAA) | Both | GDPR: data minimization action; HIPAA: encryption + logging required | CC5.1 |

---

## 9. Cross-Reference: CC Criteria to Data Flow Points

| CC Criteria | Description | Data Flow Points |
|------------|-------------|-----------------|
| **CC4.1** | Monitoring activities | AuditLogger (all phases), hash chain integrity |
| **CC4.2** | Evaluation of monitoring | query_logs(), export_logs(), verify_integrity() |
| **CC5.1** | Control activities | SafetyPolicy (Phase 1, 4), ComplianceEngine (Phase 1), PolicyEngine (tool flow) |
| **CC6.1** | Logical access controls | KeyVault encryption, CapabilitySet, TenantContext isolation |
| **CC6.2** | Access authentication | Ed25519 license validation, tenant-derived KeyVault keys |
| **CC6.3** | Access authorization | CapabilitySet.has(), ToolPolicy, ModelPolicy, RiskAttenuator |
| **CC6.6** | System boundary protection | DataClassifier.enforce(), cloud routing blocks, TLS enforcement |
| **CC7.1** | System monitoring | AuditLogger events at all phases, CostTracker |
| **CC7.2** | Security event monitoring | Security block events, hash chain verification, tamper detection |
| **CC8.1** | Change management | (Development flow -- not in request processing) |
| **CC9.1** | Risk mitigation | RiskAttenuator, DriftEngine, LoopDetector, CircuitBreaker |
| **C1.1** | Confidential information identification | DataClassifier (Phases 1, 3, 4), PII detection, secret detection |
| **C1.2** | Confidential information disposal | Audit rotation, KeyVault.destroy(), data retention config |
| **PI1.1** | Processing completeness and accuracy | GoalTracker, input validation, context compilation, memory retrieval |
| **PI1.2** | Processing output accuracy | Output validation, quality estimation, leak detection |
| **PI1.3** | Processing integrity monitoring | DriftEngine, PredictionEngine surprise, AuditLogger integrity verification |

---

*This data flow diagram is maintained under version control alongside the corteX
SDK source code. Changes to data flows that materially affect this document
trigger a review and update as part of the change management process.*
