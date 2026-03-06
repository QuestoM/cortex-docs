# Data Protection Impact Assessment (DPIA) Template

**Document ID**: GDPR-002
**Version**: 1.1
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-23
**Next Review**: 2026-08-16
**Owner**: CTO, Questo Ltd
**Approved By**: CEO, Questo Ltd
**Classification**: CONFIDENTIAL
**GAP Reference**: GAP-G09
**Legal Basis**: GDPR Article 35 -- Data Protection Impact Assessment

---

## 1. Purpose

This template assists Data Controllers (corteX customers) in conducting a Data Protection Impact Assessment (DPIA) for their deployment of the corteX AI Agent SDK. Under GDPR Article 35, a DPIA is required when processing is likely to result in a high risk to the rights and freedoms of natural persons. Sections pre-filled by Questo describe the corteX platform's technical capabilities. Sections marked with [brackets] must be completed by the customer for their specific deployment.

---

## 2. When This DPIA is Required

A DPIA is mandatory for corteX deployments that involve any of the following:

| Trigger | GDPR Basis | Typical corteX Scenario |
|---------|-----------|------------------------|
| Systematic and extensive profiling with significant effects | Art. 35(3)(a) | Brain-inspired learning with per-user adaptation enabled |
| Large-scale processing of special categories of data | Art. 35(3)(b) | Healthcare, legal, or HR domain deployments where users may share sensitive data |
| Systematic monitoring of publicly accessible areas | Art. 35(3)(c) | Customer-facing AI agents monitoring public forums or social media |
| Innovative use of new technologies | EDPB Guidelines | Any deployment using AI agent orchestration with adaptive learning |
| Automated decision-making with legal or significant effects | Art. 22 | Deployments where agent decisions affect employment, credit, insurance, or similar |

If none of these triggers apply to your deployment, document your reasoning and retain it as evidence that a DPIA was considered but deemed unnecessary.

---

## 3. DPIA Assessment

### 3.1 Assessment Metadata

| Field | Content |
|-------|---------|
| **Assessment Title** | [Title of this DPIA -- e.g., "DPIA for Customer Support AI Agent Deployment"] |
| **Assessment ID** | [Unique identifier -- e.g., DPIA-2026-001] |
| **Controller Name** | [Your organization name] |
| **Controller Contact** | [Name, email, phone of responsible person] |
| **Data Protection Officer** | [DPO name and contact details, or state if not required to appoint one] |
| **Assessment Date** | [Date of assessment] |
| **Assessment Author** | [Name and role of person conducting the DPIA] |
| **Reviewer** | [Name and role of person reviewing the DPIA] |
| **Approval Date** | [Date of final approval] |
| **Next Review Date** | [Date -- maximum 12 months from approval] |

### 3.2 Processor Information

| Field | Content |
|-------|---------|
| **Processor Name** | Questo Ltd |
| **Product** | corteX AI Agent SDK |
| **Processor Contact** | privacy@questo.co |
| **DPA Reference** | [Your DPA reference number with Questo] |
| **DPA Effective Date** | [Date your DPA was executed] |

---

## 4. Systematic Description of Processing Operations

### 4.1 Nature of Processing

#### corteX Platform Processing (Pre-filled by Questo)

The corteX AI Agent SDK performs the following processing operations:

**AI Agent Orchestration**: The runtime orchestrator coordinates multi-step AI agent workflows. It decomposes user goals into tasks, plans execution steps, invokes LLM providers for reasoning, executes tools, and verifies results against the original goal. Each step is logged via AuditConfig.

**Conversation Processing**: Natural language conversations between end users and AI agents are processed through the SDK. This includes message parsing, context assembly from memory stores, prompt construction with relevant context, transmission to the configured LLM provider, response parsing, and safety validation of outputs.

**Memory Management**: The SDK manages four tiers of memory:
- Working Memory: Current task context, cleared after each session
- Short-Term Memory: Recent interactions, configurable TTL (default 24 hours)
- Long-Term Memory: Persistent knowledge extracted from interactions, configurable TTL
- Episodic Memory: Experience patterns learned from past agent sessions, configurable TTL

**LLM API Routing**: Constructed prompts are routed to the customer-configured LLM provider (OpenAI, Google Gemini, Anthropic Claude, or local on-prem models). The provider-agnostic architecture allows customers to select providers based on their data residency and security requirements.

**Tool Execution**: AI agents can invoke customer-configured tools (code interpreter, browser, custom integrations) subject to ToolPolicy allowlisting. Default configuration is deny-all; each tool must be explicitly permitted.

**PII Detection and Tokenization**: The DataClassifier detects PII patterns (email, phone, SSN, credit card, Israeli ID with Luhn check digit validation) and the PIITokenizer redacts them before LLM transmission. PII protection is auto-enabled when GDPR compliance is active. Israeli ID detection uses a post-validation algorithm to minimize false positives.

**Brain-Inspired Learning** (when enabled): The adaptive learning engine adjusts synaptic weights and plasticity parameters based on agent performance. When enabled at the per-user level, this constitutes profiling under GDPR Article 4(4).

#### Customer-Specific Processing (Complete this section)

| Field | Your Response |
|-------|---------------|
| **Your application description** | [Describe your application that integrates corteX -- what it does, who uses it] |
| **Specific use case** | [Describe the specific AI agent use case -- e.g., customer support, content generation, data analysis] |
| **Processing scope** | [Describe the scope -- number of users, volume of data, geographic scope] |
| **Custom tools enabled** | [List any custom tools you have configured for the AI agent] |
| **Brain-inspired learning** | [State whether per-user learning is enabled and for what purpose] |
| **Decision-making impact** | [Describe any decisions made by the AI agent that affect individuals -- e.g., recommendations, filtering, scoring] |

### 4.2 Purpose of Processing

#### corteX Platform Purposes (Pre-filled by Questo)

| Purpose | Description | Data Used |
|---------|-------------|-----------|
| AI agent task execution | Executing user-requested tasks through multi-step AI reasoning | Conversation content, task parameters, tool results |
| Context maintenance | Maintaining conversation context across interactions | User messages, AI responses, memory contents |
| Goal verification | Verifying that agent actions align with the user's stated goal | Goal descriptions, step results, confidence scores |
| Safety enforcement | Preventing harmful, unauthorized, or off-policy outputs | Input content, output content, safety patterns |
| Performance optimization | Improving agent response quality through adaptive learning (when enabled) | Behavioral patterns, preference indicators, performance metrics |
| Audit and compliance | Maintaining audit trails for regulatory and security compliance | Access logs, tool execution logs, weight change logs |

#### Customer-Specific Purposes (Complete this section)

| Purpose | Description | Lawful Basis |
|---------|-------------|-------------|
| [Purpose 1] | [Description of your processing purpose] | [Art. 6(1)(a)-(f) -- specify which] |
| [Purpose 2] | [Description of your processing purpose] | [Art. 6(1)(a)-(f) -- specify which] |
| [Purpose 3] | [Description of your processing purpose] | [Art. 6(1)(a)-(f) -- specify which] |

### 4.3 Categories of Data Subjects

#### corteX Platform Data Subjects (Pre-filled by Questo)

| Category | Description |
|----------|-------------|
| End users | Individuals who interact with AI agents built on corteX through the customer's application |

#### Customer-Specific Data Subjects (Complete this section)

| Category | Estimated Number | Description | Vulnerable Individuals? |
|----------|-----------------|-------------|------------------------|
| [Category 1] | [Number] | [Description -- e.g., "Customers of our e-commerce platform"] | [Yes/No -- e.g., children, patients, employees] |
| [Category 2] | [Number] | [Description] | [Yes/No] |

### 4.4 Categories of Personal Data

#### corteX Platform Data Categories (Pre-filled by Questo)

| Data Category | Examples | Classification | Retention |
|--------------|----------|---------------|-----------|
| User identifiers | user_id, session_id, tenant_id | CONFIDENTIAL | Per customer configuration |
| Conversation content | User messages, AI responses | CONFIDENTIAL | Per customer configuration (NONE/SESSION/PERSISTENT/AUDIT_ONLY) |
| Behavioral patterns | Interaction frequency, topic preferences, usage patterns | CONFIDENTIAL | Per customer configuration; deletable per user |
| PII detected in content | Email, phone, SSN, credit card, Israeli ID (Teudat Zehut) | CONFIDENTIAL | Tokenized before LLM transmission; tokens are session-scoped |
| Preference data | Language preferences, response format preferences | INTERNAL | Per customer configuration; resettable to defaults |
| Technical metadata | Timestamps, token counts, model selections | INTERNAL | 90 days operational |
| Audit logs | Tool execution logs, access logs | CONFIDENTIAL | Per AuditConfig (minimum 90 days) |

#### Customer-Specific Data Categories (Complete this section)

| Data Category | Examples | Special Category (Art. 9)? | Source |
|--------------|----------|---------------------------|--------|
| [Category 1] | [Examples] | [Yes/No -- if yes, specify type] | [How this data enters the system] |
| [Category 2] | [Examples] | [Yes/No] | [Source] |
| [Category 3] | [Examples] | [Yes/No] | [Source] |

### 4.5 Data Flows

#### corteX Platform Data Flow (Pre-filled by Questo)

```
End User (Data Subject)
    |
    | User input (message, task request)
    v
Customer Application (Controller)
    |
    | SDK API call with user input + context
    v
corteX SDK (Processor - Questo)
    |
    +-- Safety check (input validation, injection detection)
    |
    +-- Memory retrieval (working, short-term, long-term, episodic)
    |
    +-- Context assembly (user input + memory + system prompt)
    |
    +-- Prompt construction
    |       |
    |       +-- [Optional] PII detection and redaction
    |       |
    |       v
    |   LLM Provider (Sub-Processor)
    |       |
    |       | Model response
    |       v
    +-- Response processing
    |
    +-- Safety check (output validation)
    |
    +-- [Optional] Tool execution (per ToolPolicy)
    |
    +-- Goal verification (step result vs. original goal)
    |
    +-- Memory update (store relevant context)
    |
    +-- [Optional] Weight update (brain-inspired learning)
    |
    +-- Audit logging
    |
    v
Customer Application (Controller)
    |
    | AI response
    v
End User (Data Subject)
```

#### Customer-Specific Data Flows (Complete this section)

[Draw or describe any additional data flows specific to your deployment, including:
- How user data enters your application
- What additional data sources are combined with corteX processing
- Where processed results are stored outside of corteX
- Any downstream systems that receive corteX outputs
- Any cross-system data sharing]

### 4.6 Recipients and Transfers

#### corteX Platform Recipients (Pre-filled by Questo)

| Recipient | Role | Location | Transfer Mechanism | Purpose |
|-----------|------|----------|-------------------|---------|
| OpenAI, Inc. | Sub-Processor | USA | EU-US DPF + SCCs | LLM inference (when selected by customer) |
| Google LLC (Vertex AI) | Sub-Processor | EU/US (configurable) | Adequacy (EU) / DPF + SCCs (US) | LLM inference (when selected by customer) |
| Anthropic, PBC | Sub-Processor | USA | SCCs + TIA | LLM inference (when selected by customer) |
| Questo Ltd | Processor | Israel | EU Adequacy Decision | SDK platform operation |

#### Customer-Specific Recipients (Complete this section)

| Recipient | Role | Location | Transfer Mechanism | Purpose |
|-----------|------|----------|-------------------|---------|
| [Recipient 1] | [Role] | [Location] | [Mechanism] | [Purpose] |
| [Recipient 2] | [Role] | [Location] | [Mechanism] | [Purpose] |

---

## 5. Assessment of Necessity and Proportionality

### 5.1 Necessity Assessment

For each processing purpose, assess whether the processing is necessary to achieve the stated purpose.

#### corteX Platform Necessity (Pre-filled by Questo)

| Processing Activity | Necessity Justification |
|--------------------|-----------------------|
| Conversation processing | Necessary to generate AI responses to user queries; cannot provide AI agent functionality without processing user input |
| Memory management | Necessary to maintain conversation coherence; without memory, every interaction would lack context from prior exchanges |
| LLM API routing | Necessary to access language model capabilities; the core function of the SDK |
| Tool execution | Necessary only when customer enables specific tools; default is deny-all; each tool serves a specific declared purpose |
| Brain-inspired learning | Optional feature; provides improved personalization but is not required for core functionality; disabled by default |
| Audit logging | Necessary for regulatory compliance (SOC 2, GDPR Art. 30) and security incident detection |

#### Customer-Specific Necessity (Complete this section)

| Your Processing Purpose | Why is this processing necessary? | Could the purpose be achieved with less data? |
|------------------------|----------------------------------|----------------------------------------------|
| [Purpose 1] | [Justification] | [Yes/No -- if yes, explain what changes could be made] |
| [Purpose 2] | [Justification] | [Yes/No] |

### 5.2 Proportionality Assessment

#### corteX Platform Proportionality Measures (Pre-filled by Questo)

| Principle | How corteX Addresses It |
|-----------|------------------------|
| **Data minimization** | Configurable retention (NONE/SESSION/PERSISTENT/AUDIT_ONLY); configurable context window limits; PII detection and optional redaction before LLM transmission |
| **Purpose limitation** | Per-tenant purpose configuration; cross-tenant data sharing architecturally impossible; each processing activity mapped to a declared purpose |
| **Storage limitation** | TTL enforcement on all memory tiers; automated purge after configured retention period; session data cleared on session end by default |
| **Accuracy** | Data correction APIs for memory and preferences; goal tracking verifies processing accuracy; feedback loops for continuous improvement |
| **Data subject rights** | DSAR API supports access, rectification, erasure, restriction, portability, and objection; 5-business-day response SLA to controller requests |
| **Lawful basis support** | SDK technically supports all lawful bases; consent management integration points; processing halt on consent withdrawal |

#### Customer-Specific Proportionality (Complete this section)

| Principle | How your deployment addresses it |
|-----------|--------------------------------|
| **Data minimization** | [What measures have you implemented to minimize data processed?] |
| **Purpose limitation** | [How do you ensure data is used only for declared purposes?] |
| **Storage limitation** | [What retention periods have you configured? Why?] |
| **Accuracy** | [How do you ensure data accuracy?] |
| **Data subject rights** | [How do you handle DSAR requests from your users?] |
| **Lawful basis** | [What lawful basis applies to each processing purpose?] |

### 5.3 Safeguards for Automated Decision-Making (Article 22)

If your deployment involves automated decisions with legal or similarly significant effects on individuals, complete this section.

#### corteX Platform Safeguards (Pre-filled by Questo)

| Safeguard | Implementation |
|-----------|---------------|
| Human-in-the-loop | Configurable insertion points in the orchestration pipeline; customers can require human approval before executing consequential actions |
| Explainability | Goal tracking provides step-by-step reasoning; confidence scoring indicates reliability; audit logs record decision chains |
| Contestability | Customers can implement appeal/review processes; corteX provides the data export capabilities to support review |
| Opt-out mechanism | Per-user profiling can be disabled without affecting core functionality |

#### Customer-Specific Safeguards (Complete this section)

| Safeguard | Your Implementation |
|-----------|-------------------|
| **Human oversight** | [Describe how humans oversee AI agent decisions in your deployment] |
| **Right to contest** | [Describe how users can contest automated decisions] |
| **Alternative processing** | [Describe alternatives available to users who object to automated processing] |
| **Meaningful information** | [Describe how you inform users about the logic, significance, and consequences of automated processing] |

---

## 6. Assessment of Risks to Rights and Freedoms

### 6.1 Risk Identification

#### corteX Platform Risks (Pre-filled by Questo)

| Risk ID | Risk | Rights Affected | Likelihood | Severity | Overall Risk |
|---------|------|----------------|------------|----------|-------------|
| DPIA-R01 | Cross-tenant data leakage exposes personal data of one customer's users to another customer | Right to privacy, right to data protection | Low | High | MEDIUM |
| DPIA-R02 | Prompt injection causes AI agent to reveal personal data from context or memory | Right to privacy, right to data protection | Medium | Medium | MEDIUM |
| DPIA-R03 | Brain-inspired learning creates profiles that enable re-identification of anonymized users | Right to privacy, right not to be subject to profiling | Low | High | MEDIUM |
| DPIA-R04 | AI agent makes inaccurate automated decision with significant effect on individual | Right to fair treatment, right to non-discrimination | Medium | High | HIGH |
| DPIA-R05 | LLM provider breach exposes conversation data containing personal information | Right to data protection | Low | High | MEDIUM |
| DPIA-R06 | Failure to delete user data upon erasure request due to distributed storage | Right to erasure | Low | Medium | LOW |
| DPIA-R07 | Special category data (health, political) entered by users is stored in long-term memory | Right to protection of sensitive data | Medium | High | HIGH |
| DPIA-R08 | Cross-border transfer to LLM provider in jurisdiction without adequate protection | Right to data protection | Low | Medium | LOW |
| DPIA-R09 | AI agent output contains discriminatory or biased content affecting individuals | Right to non-discrimination | Medium | Medium | MEDIUM |
| DPIA-R10 | Insufficient transparency about AI processing prevents informed consent | Right to information, right to meaningful consent | Medium | Medium | MEDIUM |

#### Customer-Specific Risks (Complete this section)

| Risk ID | Risk | Rights Affected | Likelihood | Severity | Overall Risk |
|---------|------|----------------|------------|----------|-------------|
| [DPIA-C01] | [Risk description specific to your deployment] | [Rights affected] | [Low/Medium/High] | [Low/Medium/High] | [LOW/MEDIUM/HIGH] |
| [DPIA-C02] | [Risk description] | [Rights] | [L/M/H] | [L/M/H] | [L/M/H] |
| [DPIA-C03] | [Risk description] | [Rights] | [L/M/H] | [L/M/H] | [L/M/H] |

### 6.2 Risk Scoring Methodology

| | Severity: Low | Severity: Medium | Severity: High |
|-|--------------|-----------------|----------------|
| **Likelihood: High** | MEDIUM | HIGH | HIGH |
| **Likelihood: Medium** | LOW | MEDIUM | HIGH |
| **Likelihood: Low** | LOW | LOW | MEDIUM |

---

## 7. Measures to Address Risks

### 7.1 corteX Platform Measures (Pre-filled by Questo)

| Risk ID | Risk | Measures Implemented | Residual Risk |
|---------|------|---------------------|---------------|
| DPIA-R01 | Cross-tenant data leakage | Session isolation with tenant_id enforcement at every data access layer; no shared mutable state; UUID4 session identifiers; penetration testing validates isolation | LOW |
| DPIA-R02 | Prompt injection data disclosure | SafetyPolicy with injection_protection; pattern matching against injection vectors; input validation; output safety checks; continuous pattern library updates | LOW |
| DPIA-R03 | Profile-based re-identification | Per-user weight deltas isolated and deletable; learning is opt-in only; aggregate-level learning preferred; customers can disable per-user adaptation entirely | LOW |
| DPIA-R04 | Inaccurate automated decisions | Goal tracking verifies each step; confidence scoring on outputs; human-in-the-loop insertion points; customer documentation emphasizes output validation; no autonomous high-stakes decisions by default | MEDIUM |
| DPIA-R05 | LLM provider breach | Enterprise API tiers with enhanced security; DPAs with breach notification clauses; on-prem deployment option eliminates provider dependency; PII detection and optional redaction before transmission | LOW |
| DPIA-R06 | Incomplete erasure | DSAR API covers all storage tiers (conversations, all memory levels, weight deltas, preferences, episodic records); deletion verification; audit trail of deletion actions | LOW |
| DPIA-R07 | Special category data in memory | PII detection flags special category data; configurable automatic redaction; customers can prevent special category storage in long-term memory; documented risk in customer privacy materials | MEDIUM |
| DPIA-R08 | Inadequate transfer protection | DPF + SCCs for US providers; EU region selection for Google Vertex; on-prem option eliminates transfers; Transfer Impact Assessments completed; customer can restrict to EU-only providers | LOW |
| DPIA-R09 | Discriminatory AI output | SafetyPolicy output validation; customer-configurable content filters; goal tracking detects deviation; feedback loops for bias correction; customer documentation covers bias monitoring | MEDIUM |
| DPIA-R10 | Insufficient transparency | Comprehensive documentation (97 pages); ROPA maintained; sub-processor list published; customer-facing privacy documentation templates; explainability through goal tracking and audit logs | LOW |

### 7.2 Customer-Specific Measures (Complete this section)

| Risk ID | Risk | Your Additional Measures | Expected Residual Risk |
|---------|------|-------------------------|----------------------|
| DPIA-R04 | Inaccurate automated decisions | [Describe your human oversight and appeal processes] | [LOW/MEDIUM/HIGH] |
| DPIA-R07 | Special category data | [Describe how you handle sensitive data in your use case] | [LOW/MEDIUM/HIGH] |
| DPIA-R09 | Discriminatory output | [Describe your bias monitoring and correction procedures] | [LOW/MEDIUM/HIGH] |
| DPIA-R10 | Transparency | [Describe how you inform your users about AI processing] | [LOW/MEDIUM/HIGH] |
| [DPIA-C01] | [Your specific risk] | [Your measures] | [L/M/H] |
| [DPIA-C02] | [Your specific risk] | [Your measures] | [L/M/H] |

---

## 8. Consultation

### 8.1 Internal Consultation

| Consultee | Role | Date | Input Provided |
|-----------|------|------|---------------|
| [Name] | [DPO / Privacy Lead] | [Date] | [Summary of input] |
| [Name] | [IT / Security Lead] | [Date] | [Summary of input] |
| [Name] | [Business Owner] | [Date] | [Summary of input] |

### 8.2 Data Subject Consultation

GDPR Article 35(9) requires consultation with data subjects or their representatives where appropriate.

| Consultation Method | Date | Summary of Findings |
|--------------------|------|-------------------|
| [Method -- e.g., user survey, focus group, public consultation] | [Date] | [Summary] |

If data subject consultation was not conducted, document the justification:

[Justification for not consulting data subjects -- e.g., "Consultation not practicable due to commercial confidentiality; however, user testing with representative participants was conducted on [date]."]

### 8.3 Supervisory Authority Consultation (Article 36)

If the residual risk remains HIGH after implementing mitigation measures, prior consultation with the supervisory authority is required under Article 36.

| Question | Response |
|----------|----------|
| Is prior consultation required? | [Yes/No -- Yes if any residual risk remains HIGH] |
| Supervisory authority | [Name of relevant DPA -- e.g., Irish DPC, French CNIL, Israeli PPA] |
| Consultation date | [Date, if applicable] |
| Outcome | [Summary of authority's response, if applicable] |

---

## 9. DPIA Outcome and Decision

### 9.1 Overall Assessment

| Question | Response |
|----------|----------|
| Can the processing proceed? | [Yes / Yes with conditions / No] |
| Are all identified risks mitigated to acceptable levels? | [Yes / No -- if No, detail which risks remain unacceptable] |
| Are additional measures required before processing begins? | [Yes / No -- if Yes, list measures and timeline] |
| Is supervisory authority consultation required? | [Yes / No] |

### 9.2 Conditions for Proceeding (if applicable)

| Condition | Responsible Person | Deadline | Status |
|-----------|-------------------|----------|--------|
| [Condition 1 -- e.g., "Enable PII redaction before LLM transmission"] | [Name] | [Date] | [Not Started / In Progress / Complete] |
| [Condition 2] | [Name] | [Date] | [Status] |
| [Condition 3] | [Name] | [Date] | [Status] |

### 9.3 Approval

| Role | Name | Signature | Date |
|------|------|-----------|------|
| DPIA Author | [Name] | _________________ | [Date] |
| DPO / Privacy Lead | [Name] | _________________ | [Date] |
| Business Owner | [Name] | _________________ | [Date] |
| Processor (Questo) Review | [Name] | _________________ | [Date] |

---

## 10. Monitoring and Review

### 10.1 Review Triggers

This DPIA must be reviewed and updated when any of the following occurs:

- The processing operations change in scope, purpose, or method
- New categories of personal data are processed
- New categories of data subjects are affected
- The LLM provider configuration changes
- Brain-inspired learning is enabled or its scope is expanded
- A security incident occurs that affects the processing
- Regulatory guidance or enforcement actions affect similar processing
- At least 12 months have passed since the last review

### 10.2 Review Schedule

| Review Activity | Frequency | Responsible | Next Due |
|----------------|-----------|-------------|----------|
| Full DPIA review | Annually or upon trigger event | [DPIA Author] | [12 months from approval date] |
| Risk assessment update | Semi-annually | [Privacy Lead] | [6 months from approval date] |
| Mitigation measure effectiveness check | Quarterly | [IT/Security Lead] | [3 months from approval date] |

### 10.3 Review Log

| Review Date | Reviewer | Changes Made | Outcome |
|-------------|----------|-------------|---------|
| [Date] | [Name] | [Description of changes] | [Approved / Requires further action] |

---

## 11. Appendices

### Appendix A: corteX Technical Documentation References

| Document | Description | Location |
|----------|-------------|----------|
| corteX SDK Documentation | Complete technical documentation (~115 pages) | docs.cortex-ai.com |
| ROPA | Records of Processing Activities | docs/compliance/ROPA.md |
| Risk Register | Comprehensive risk register with mitigations | docs/compliance/risk_register.md |
| Security Training | Security awareness training materials | docs/compliance/security_training.md |
| Data Flow Diagram | Visual representation of data processing flows | Available on request from Questo |
| Sub-Processor List | Current list of sub-processors and their roles | Published in ROPA Section 6 |
| DPA Template | Standard Data Processing Agreement | Available on request from Questo |

### Appendix B: Glossary

| Term | Definition |
|------|-----------|
| AI Agent | An autonomous software entity that uses LLM capabilities to accomplish user-defined goals through multi-step reasoning and tool execution |
| Brain-Inspired Learning | The corteX adaptive learning engine that uses synaptic weights, plasticity, and prediction models to improve agent performance over time |
| BYOK | Bring Your Own Key -- customers provide their own LLM provider API keys |
| Data Controller | Under GDPR, the entity that determines the purposes and means of processing personal data |
| Data Processor | Under GDPR, the entity that processes personal data on behalf of the controller |
| DPA | Data Processing Agreement between controller and processor per GDPR Article 28 |
| DSAR | Data Subject Access Request -- a request from an individual to exercise their GDPR rights |
| LLM | Large Language Model -- the AI model that generates natural language responses |
| On-Prem | On-premises deployment where the software runs entirely within the customer's own infrastructure |
| PII | Personally Identifiable Information |
| Profiling | Any form of automated processing of personal data to evaluate personal aspects (GDPR Art. 4(4)) |
| ROPA | Records of Processing Activities per GDPR Article 30 |
| SafetyPolicy | The corteX module that validates inputs and outputs against safety rules |
| SCCs | Standard Contractual Clauses -- EU-approved contractual mechanism for international data transfers |
| Sub-Processor | A processor engaged by another processor (e.g., LLM providers engaged by Questo) |
| Tenant Isolation | Architectural separation ensuring one customer's data cannot be accessed by another customer |
| ToolPolicy | The corteX module that controls which tools an AI agent is permitted to execute |
| Weight Deltas | Per-user or per-tenant adjustments to the brain-inspired learning model, stored separately and deletable |

---

## 12. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO, Questo Ltd | Initial DPIA template with pre-filled corteX platform sections and customer fill-in sections |
| 1.1 | 2026-02-23 | CTO, Questo Ltd | Added Israeli ID PII detection (with Luhn validation), updated documentation page count, added PII data category row, noted auto-enable of PII protection under GDPR compliance |
