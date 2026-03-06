# Records of Processing Activities (ROPA)

**Document ID**: GDPR-001
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2026-08-16
**Owner**: CTO, Questo Ltd (acting DPO)
**Approved By**: CEO, Questo Ltd
**Classification**: CONFIDENTIAL
**GAP Reference**: GAP-G06
**Legal Basis**: GDPR Article 30(2) -- Processor Records of Processing Activities

---

## 1. Purpose

This document constitutes the Records of Processing Activities (ROPA) maintained by Questo Ltd in its capacity as a Data Processor under GDPR Article 30(2). It records all categories of processing activities carried out on behalf of Data Controllers (customers) using the corteX AI Agent SDK platform. This ROPA is a mandatory compliance document and must be made available to supervisory authorities upon request.

---

## 2. Processor Information

| Field | Content |
|-------|---------|
| **Processor Name** | Questo Ltd |
| **Registration Number** | [Israeli Company Registration Number] |
| **Registered Address** | [Registered Address, Israel] |
| **Primary Contact** | CEO, Questo Ltd |
| **Data Protection Contact** | CTO, Questo Ltd (acting DPO) |
| **Contact Email** | privacy@questo.co |
| **Contact Phone** | [Contact Phone Number] |
| **EU Representative (Art. 27)** | [To be appointed before first EU customer engagement] |
| **Processor Role** | Data Processor under GDPR Article 28 |

---

## 3. Controller Registry

The following table records all Data Controllers on whose behalf Questo Ltd processes personal data. Each controller must have an executed Data Processing Agreement (DPA) before processing begins.

| Controller Name | Contact Details | DPA Reference | DPA Effective Date | Processing Scope | Data Residency | Status |
|----------------|----------------|---------------|-------------------|-----------------|----------------|--------|
| [Customer Name 1] | [Contact Details] | DPA-001 | [Date] | [Scope] | [EU/US/On-Prem] | Active |
| [Customer Name 2] | [Contact Details] | DPA-002 | [Date] | [Scope] | [EU/US/On-Prem] | Active |
| [Customer Name 3] | [Contact Details] | DPA-003 | [Date] | [Scope] | [EU/US/On-Prem] | Active |

*Note: This registry is updated within 5 business days of executing or terminating a DPA.*

---

## 4. Categories of Processing Activities

### 4.1 AI Agent Orchestration

| Field | Content |
|-------|---------|
| **Processing Activity** | AI Agent Orchestration |
| **Description** | Coordinating multi-step AI agent workflows including goal decomposition, task planning, step execution, and result verification through the corteX runtime orchestrator |
| **Categories of Data Subjects** | End users of customer applications who interact with AI agents |
| **Categories of Personal Data** | User identifiers (user_id, session_id), conversation context, task parameters, goal descriptions |
| **Purpose of Processing** | Executing AI agent tasks as instructed by the Data Controller to provide intelligent automation and assistance to end users |
| **Legal Basis (Controller's)** | Per controller's determination: typically Contract (Art. 6(1)(b)) or Legitimate Interest (Art. 6(1)(f)) |
| **Retention Period** | Per controller's configuration: SESSION (cleared on session end), PERSISTENT (controller-defined TTL), or AUDIT_ONLY (metadata retained, content purged) |
| **Data Source** | Received from controller's application via corteX SDK API |
| **Automated Decision-Making** | Yes -- AI agent makes autonomous decisions within defined goal parameters; human-in-the-loop configurable per controller |

### 4.2 Conversation Processing

| Field | Content |
|-------|---------|
| **Processing Activity** | Conversation Processing |
| **Description** | Processing natural language conversations between end users and AI agents, including message parsing, context assembly, prompt construction, and response generation |
| **Categories of Data Subjects** | End users of customer applications |
| **Categories of Personal Data** | Conversation content (user messages, AI responses), user identifiers, timestamps, language preferences, conversation metadata |
| **Purpose of Processing** | Generating contextually relevant AI responses as instructed by the Data Controller |
| **Legal Basis (Controller's)** | Per controller's determination |
| **Retention Period** | Per controller's configuration: NONE (not stored), SESSION (cleared on session end), or PERSISTENT (controller-defined TTL) |
| **Data Source** | Received from controller's application via SDK API |
| **Special Category Data Risk** | Users may voluntarily disclose health, political, religious, or other Art. 9 data during conversations; PII detection module flags such content |

### 4.3 Memory Management

| Field | Content |
|-------|---------|
| **Processing Activity** | Memory Management |
| **Description** | Managing four tiers of agent memory: working memory (current task context), short-term memory (recent interactions), long-term memory (persistent knowledge), and episodic memory (experience patterns) |
| **Categories of Data Subjects** | End users of customer applications |
| **Categories of Personal Data** | Derived behavioral patterns, user preferences, interaction history summaries, contextual associations, episodic records |
| **Purpose of Processing** | Maintaining agent context and continuity across interactions as configured by the Data Controller |
| **Legal Basis (Controller's)** | Per controller's determination: typically Consent (Art. 6(1)(a)) for long-term memory, or Contract for session memory |
| **Retention Period** | Working memory: session duration only. Short-term memory: controller-configured TTL (default 24 hours). Long-term memory: controller-configured TTL or until explicit deletion. Episodic memory: controller-configured TTL or until explicit deletion |
| **Data Source** | Derived from conversation processing and agent interactions |
| **Erasure Support** | All memory tiers support per-user deletion via DSAR API; weight deltas support per-user erasure |

### 4.4 LLM API Routing

| Field | Content |
|-------|---------|
| **Processing Activity** | LLM API Routing |
| **Description** | Routing constructed prompts to configured Large Language Model providers (OpenAI, Google Gemini, Anthropic Claude, or local models) and processing responses |
| **Categories of Data Subjects** | End users of customer applications (indirectly, through conversation content included in prompts) |
| **Categories of Personal Data** | Prompt content (may include user messages, context from memory), model responses, routing metadata, token usage metrics |
| **Purpose of Processing** | Generating AI model responses as instructed by the Data Controller through their selected LLM provider configuration |
| **Legal Basis (Controller's)** | Per controller's determination |
| **Retention Period** | Prompts and responses: per controller's retention configuration. Routing metadata: 90 days for operational monitoring |
| **International Transfers** | Per controller's LLM provider selection -- see Section 7 for transfer mechanisms |
| **Sub-Processors Engaged** | OpenAI (when selected), Google (when selected), Anthropic (when selected) -- see Section 6 |
| **Data Minimization** | PII detection and optional redaction before LLM transmission; configurable context window limits; customers can select on-prem models to eliminate external transfers |

### 4.5 Tool Execution

| Field | Content |
|-------|---------|
| **Processing Activity** | Tool Execution |
| **Description** | Executing customer-configured tools and plugins (code interpreter, browser, custom tools) as part of AI agent workflows, subject to ToolPolicy allowlisting |
| **Categories of Data Subjects** | End users of customer applications |
| **Categories of Personal Data** | Tool input parameters (may include user data), tool execution results, execution metadata |
| **Purpose of Processing** | Performing actions and retrieving information as instructed by the Data Controller through tool configuration |
| **Legal Basis (Controller's)** | Per controller's determination |
| **Retention Period** | Tool execution logs: 90 days for audit purposes. Tool results: per controller's retention configuration |
| **Data Source** | Derived from agent orchestration decisions and user requests |
| **Access Control** | ToolPolicy enforces explicit allowlisting; default_deny prevents unauthorized tool execution; per-tenant tool configuration |

### 4.6 Brain-Inspired Learning (When Enabled)

| Field | Content |
|-------|---------|
| **Processing Activity** | Brain-Inspired Adaptive Learning |
| **Description** | Adjusting synaptic weights, plasticity parameters, and prediction models based on agent performance feedback to improve future responses. This constitutes profiling under GDPR Article 22 when applied to individual users |
| **Categories of Data Subjects** | End users of customer applications (when per-user learning is enabled) |
| **Categories of Personal Data** | Behavioral patterns, preference indicators, performance metrics, prediction accuracy data, weight deltas |
| **Purpose of Processing** | Improving AI agent performance and personalization as opted into by the Data Controller |
| **Legal Basis (Controller's)** | Consent (Art. 6(1)(a)) recommended for per-user profiling; Legitimate Interest (Art. 6(1)(f)) may apply for aggregate tenant-level learning |
| **Retention Period** | Tenant weight deltas: per controller's configuration. Per-user weight deltas: until user erasure request or controller-configured TTL |
| **Profiling Implications** | Per-user weight adaptation constitutes profiling under Art. 4(4); requires explicit disclosure; opt-in only; subject to Art. 22 safeguards |
| **Erasure Support** | Per-user weight deltas fully deletable; tenant weight deltas deletable on DPA termination; base model weights contain no personal data |

---

## 5. Types of Personal Data Processed

| Data Category | Examples | Classification Level | Retention | Erasure Method |
|--------------|----------|---------------------|-----------|----------------|
| User Identifiers | user_id, session_id, tenant_id | CONFIDENTIAL | Per controller config | Delete from all stores |
| Conversation Content | User messages, AI responses, context | CONFIDENTIAL | Per controller config | Delete from conversation store and memory |
| Behavioral Patterns | Interaction frequency, topic preferences, usage patterns | CONFIDENTIAL | Per controller config | Delete weight deltas and episodic memory |
| Preference Data | Language preferences, response format preferences, tool preferences | INTERNAL | Per controller config | Reset to defaults and delete preference store |
| Technical Metadata | Timestamps, token counts, model selections, routing decisions | INTERNAL | 90 days operational | Automated purge after retention period |
| Audit Logs | Tool execution logs, access logs, weight change logs | CONFIDENTIAL | Per controller's AuditConfig (minimum 90 days) | Purge after retention period; regulatory hold override |
| API Keys (Controller's) | LLM provider keys stored in KeyVault | RESTRICTED | Duration of service | Secure deletion via KeyVault; key material zeroed |

---

## 6. Sub-Processor Registry

### 6.1 Current Sub-Processors

| Sub-Processor | Purpose | Data Processed | Location | DPA Status | Transfer Mechanism | SOC 2 Report |
|---------------|---------|---------------|----------|------------|-------------------|--------------|
| OpenAI, Inc. | LLM inference when customer selects OpenAI provider | Prompt content, model responses | USA | Enterprise API Terms with DPA | EU-US DPF + SCCs | Available on request |
| Google LLC (Vertex AI) | LLM inference when customer selects Gemini provider; optional cloud hosting | Prompt content, model responses | EU/US (configurable per customer) | Google Cloud Data Processing Addendum (CDPA) | Adequacy (EU regions) / DPF + SCCs (US) | SOC 2 Type II available |
| Anthropic, PBC | LLM inference when customer selects Claude provider | Prompt content, model responses | USA | Enterprise API Terms with DPA | SCCs + Transfer Impact Assessment | Available on request |

### 6.2 Sub-Processor Engagement Conditions

- Each sub-processor is engaged only when the controller explicitly selects that LLM provider in their tenant configuration.
- On-prem model deployments do not engage any sub-processors.
- Controllers are notified 30 days in advance of any sub-processor additions or changes.
- Controllers have the right to object to sub-processor changes and may terminate if the objection is not resolved.
- Equivalent data protection obligations are imposed on all sub-processors through contractual arrangements.

### 6.3 Sub-Processor Change Log

| Date | Change Type | Sub-Processor | Details | Customer Notification Date |
|------|-------------|---------------|---------|---------------------------|
| 2026-02-16 | Initial Registry | All | Initial sub-processor registry established | N/A (pre-launch) |

---

## 7. International Data Transfers

### 7.1 Transfer Mechanisms by Destination

| Destination | Mechanism | Status | Supplementary Measures |
|-------------|-----------|--------|----------------------|
| USA (OpenAI) | EU-US Data Privacy Framework + SCCs (Module 3: Processor to Processor) | Active; DPF survived legal challenge Sep 2025 | Encryption in transit (TLS 1.3); enterprise API tier with no-training clause; Transfer Impact Assessment completed |
| USA (Anthropic) | SCCs (Module 3: Processor to Processor) + Transfer Impact Assessment | Active | Encryption in transit (TLS 1.3); enterprise API tier; contractual no-training clause; data minimization before transfer |
| EU (Google Vertex) | No transfer (data remains in EU) | Active | EU region selection eliminates transfer requirements |
| USA (Google Vertex) | EU-US DPF + SCCs | Active (when US region selected) | EU region recommended as default; customer explicit opt-in for US processing |
| Israel (Questo infrastructure) | EU Adequacy Decision for Israel | Active | Israel holds EU adequacy decision; standard security measures apply |
| Customer On-Prem | No transfer (data remains in customer infrastructure) | Active | Eliminates all transfer concerns; recommended for highest sensitivity |

### 7.2 Transfer Impact Assessment Summary

A Transfer Impact Assessment (TIA) has been conducted for each transfer destination. Key findings:

- **USA via DPF**: DPF provides adequate protection; PCLOB reconstitution status monitored; SCCs maintained as fallback mechanism.
- **Israel**: EU adequacy decision in force; no supplementary measures required beyond standard security.
- **On-Prem**: No transfer occurs; data sovereignty fully maintained by the controller.

---

## 8. Technical and Organizational Security Measures (Article 32)

### 8.1 Encryption

| Measure | Implementation | Standard |
|---------|---------------|----------|
| Encryption at rest | Fernet encryption (AES-128-CBC + HMAC-SHA256, PBKDF2-derived 256-bit key) for API keys in KeyVault; AES-256 for other stored personal data | NIST SP 800-175B |
| Encryption in transit | TLS 1.3 for all API communications and data transfers | RFC 8446 |
| Key management | Centralized key management through KeyVault module; key rotation support | NIST SP 800-57 |
| License validation | Ed25519 cryptographic signatures for license integrity | RFC 8032 |

### 8.2 Access Controls

| Measure | Implementation |
|---------|---------------|
| Role-Based Access Control (RBAC) | Enforced at tenant level via TenantConfig; SafetyLevel.LOCKED prevents override |
| Multi-Factor Authentication | Required on all development and production systems |
| Least Privilege | Default_deny on all tool and model access; explicit allowlisting required |
| Access Reviews | Quarterly access reviews with management sign-off |
| User Deprovisioning | Within 24 hours of role change or departure |

### 8.3 Tenant Isolation

| Measure | Implementation |
|---------|---------------|
| Session Isolation | Each tenant operates in isolated memory space; no shared mutable state |
| Data Isolation | Tenant_id enforced at every data access layer |
| Key Isolation | Per-tenant API keys stored separately in KeyVault |
| Audit Isolation | Audit logs filterable and segregated by tenant |
| Weight Isolation | Brain-inspired learning weights maintained per-tenant and per-user |

### 8.4 Monitoring and Audit

| Measure | Implementation |
|---------|---------------|
| Audit Logging | Configurable via AuditConfig: tool calls, weight changes, model routing, message content |
| PII Detection | Automated PII detection in processing pipeline via DataClassifier |
| Log Sanitization | API keys and sensitive data automatically redacted from all log outputs |
| Secret Scanning | GitHub secret scanning enabled on all repositories |

### 8.5 Data Minimization and Purpose Limitation

| Measure | Implementation |
|---------|---------------|
| Configurable Retention | DataRetention enum (NONE, SESSION, PERSISTENT, AUDIT_ONLY) per tenant |
| Context Window Limits | Configurable maximum context sent to LLM providers |
| PII Redaction | Optional PII stripping before LLM API transmission |
| Purpose Binding | Per-tenant purpose configuration; cross-tenant data sharing architecturally impossible |

---

## 9. Data Subject Rights Support

Questo Ltd, as processor, assists Data Controllers in fulfilling data subject rights requests:

| Right | GDPR Article | Technical Capability | Response SLA |
|-------|-------------|---------------------|--------------|
| Access | Art. 15 | Export all data for a specific user in JSON format via DSAR API | 5 business days |
| Rectification | Art. 16 | Correct data in memory, preferences, and conversation history | 5 business days |
| Erasure | Art. 17 | Delete from all stores: conversations, memory (all tiers), weight deltas, episodic records, preferences | 5 business days |
| Restriction | Art. 18 | Temporarily halt processing for a specific user while retaining data | 5 business days |
| Portability | Art. 20 | Export in structured JSON format (cortex_gdpr_export_v1) | 5 business days |
| Objection | Art. 21 | Halt processing based on legitimate interest for a specific user | 5 business days |
| Automated Decisions | Art. 22 | Explainability through goal tracking; human-in-the-loop insertion points | 5 business days |

---

## 10. Data Protection Impact Assessment

Questo Ltd maintains a DPIA template (DPIA_Template.md) for customers deploying corteX in scenarios that require high-risk processing assessment. Questo assists controllers with DPIA obligations as required under Article 28(3)(f).

Processing activities likely requiring a DPIA:

- Deployment of brain-inspired learning with per-user adaptation (profiling)
- Processing of special category data in conversations (health, political opinions)
- Large-scale processing of employee data (HR domain deployments)
- Automated decision-making that produces legal or similarly significant effects

---

## 11. Breach Notification Procedures

| Phase | Timeline | Action | Responsible |
|-------|----------|--------|-------------|
| Detection | 0-4 hours | Detect breach, activate incident response team, initiate containment | CTO |
| Internal Assessment | 4-12 hours | Assess scope, affected tenants, data categories compromised | CTO + Engineering |
| Controller Notification | Within 24 hours of awareness | Notify affected Data Controllers with breach details per DPA requirements | CTO |
| Controller to DPA | Within 72 hours of awareness (controller's obligation) | Controller notifies supervisory authority per Art. 33 | Controller (Questo assists) |
| Data Subject Notification | If high risk (controller's obligation) | Controller notifies affected individuals per Art. 34 | Controller (Questo assists) |
| Remediation | Ongoing | Root cause analysis, fixes, documentation, process improvements | CTO + Engineering |

Notification content includes: nature of the breach, categories and approximate number of affected data subjects, contact details for Questo's data protection contact, likely consequences, and measures taken or proposed.

---

## 12. Update Log

| Date | Updated By | Section | Change Description |
|------|-----------|---------|-------------------|
| 2026-02-16 | CTO | All | Initial ROPA creation covering all processing activities |

---

## 13. Review Schedule

| Review Activity | Frequency | Responsible | Next Due |
|----------------|-----------|-------------|----------|
| Full ROPA review | Semi-annually | CTO (acting DPO) | 2026-08-16 |
| Controller registry update | Within 5 business days of DPA execution/termination | CTO | Ongoing |
| Sub-processor registry update | Within 5 business days of any change | CTO | Ongoing |
| Transfer mechanism review | Annually or upon regulatory change | CTO | 2027-02-16 |
| Security measures review | Annually | CTO | 2027-02-16 |

---

## 14. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO, Questo Ltd | Initial ROPA covering 6 processing activities, 3 sub-processors, 5 transfer mechanisms |
