# GDPR Compliance Guide for corteX AI Agent SDK

> **Document Type**: Permanent compliance reference
> **Audience**: corteX engineering team, customers, legal review
> **Last Updated**: 2026-02-16
> **Regulatory Scope**: EU GDPR (Regulation 2016/679), EU AI Act (Regulation 2024/1689)

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [GDPR Fundamentals for an AI SDK](#gdpr-fundamentals-for-an-ai-sdk)
3. [Key GDPR Articles for AI Systems](#key-gdpr-articles-for-ai-systems)
4. [AI-Specific GDPR Challenges](#ai-specific-gdpr-challenges)
5. [Required Documents and Policies](#required-documents-and-policies)
6. [Technical Requirements](#technical-requirements)
7. [Multi-Tenant AI SDK Requirements](#multi-tenant-ai-sdk-requirements)
8. [2026 Enforcement Landscape](#2026-enforcement-landscape)
9. [Implementation Checklist for corteX](#implementation-checklist-for-cortex)
10. [Sub-Processor Registry](#sub-processor-registry)

---

## 1. Executive Summary

corteX is an enterprise AI Agent SDK that processes personal data on behalf of its customers. Under GDPR, corteX operates as a **Data Processor** while its customers (enterprise SaaS companies) are **Data Controllers**. Those customers may themselves be processors for their own end users, creating a B2B2C chain with layered obligations.

This guide provides comprehensive GDPR compliance guidance covering:

- Legal role definitions and obligations
- Technical implementation requirements
- Required legal documents and templates
- AI-specific challenges unique to brain-inspired learning systems
- Multi-tenant data isolation requirements
- Current enforcement landscape and regulatory trends

**Key Risk Areas for corteX:**

| Risk Area | GDPR Impact | Severity |
|-----------|-------------|----------|
| Conversation history storage | Personal data retention | HIGH |
| Brain-inspired learning/adaptation | Profiling under Article 22 | HIGH |
| LLM API calls to external providers | International data transfers, sub-processor chain | HIGH |
| Per-user memory and preferences | Purpose limitation, data minimization | MEDIUM |
| Synaptic weight adaptation | Right to erasure complexity | MEDIUM |
| Multi-tenant shared infrastructure | Data isolation, breach scope | MEDIUM |

---

## 2. GDPR Fundamentals for an AI SDK

### 2.1 Data Controller vs Data Processor

**Definitions:**

- **Data Controller** (GDPR Art. 4(7)): The entity that determines the purposes and means of processing personal data. In the corteX chain, this is the **customer** (the enterprise SaaS company deploying corteX).

- **Data Processor** (GDPR Art. 4(8)): The entity that processes personal data on behalf of the controller. **corteX (Questo)** is a data processor.

- **Sub-Processor**: A processor engaged by another processor. **LLM providers** (OpenAI, Google, Anthropic) are sub-processors in the corteX chain.

**The B2B2C Chain:**

```
End User (Data Subject)
    |
    v
Enterprise Customer (Data Controller or Processor-for-their-customer)
    |
    v
corteX / Questo (Data Processor)
    |
    v
LLM Provider - OpenAI/Google/Anthropic (Sub-Processor)
    |
    v
Cloud Infrastructure - GKE/AWS/Azure (Sub-Sub-Processor)
```

**Key Implications:**

- corteX must ONLY process data according to the customer's documented instructions
- corteX must maintain a Data Processing Agreement (DPA) with each customer
- corteX must obtain customer authorization before engaging sub-processors
- corteX must notify customers of any sub-processor changes
- corteX must assist the customer in fulfilling data subject rights
- The customer is ultimately responsible for having a lawful basis, but corteX must make compliance technically possible

### 2.2 Legal Bases for Processing (Article 6)

The **customer** (controller) must establish one of these legal bases. corteX must technically support all of them:

| Legal Basis | Description | Common for AI SDK? |
|-------------|-------------|-------------------|
| **Consent** (Art. 6(1)(a)) | Data subject explicitly agrees | Yes - for optional AI features |
| **Contractual Necessity** (Art. 6(1)(b)) | Processing needed to fulfill a contract | Yes - core service delivery |
| **Legitimate Interest** (Art. 6(1)(f)) | Controller has legitimate business reason, balanced against data subject rights | Yes - service improvement, analytics |
| Legal Obligation (Art. 6(1)(c)) | Required by law | Rare for AI processing |
| Vital Interests (Art. 6(1)(d)) | Protecting someone's life | Very rare |
| Public Interest (Art. 6(1)(e)) | Public authority tasks | Not applicable |

**For B2B SaaS AI:**

- **Contractual necessity** is the strongest basis for core AI agent functionality
- **Legitimate interest** requires a documented three-part test (legitimate purpose, necessity, balancing)
- **Consent** is strongest for optional features like learning/personalization
- The EDPB confirmed in its December 2024 opinion that legitimate interest can serve as a legal basis for AI model development and deployment, subject to the three-part test

### 2.3 What Constitutes Personal Data in AI Agent Conversations

Under GDPR, "personal data" means any information relating to an identified or identifiable natural person. In the context of AI agent conversations, this includes:

**Direct Identifiers:**
- Names, email addresses, phone numbers
- Account IDs, user IDs
- Physical addresses
- Financial information (bank accounts, card numbers)

**Indirect Identifiers:**
- IP addresses, device IDs
- Location data
- Cookies and session tokens
- Behavioral patterns that could identify a person

**Conversation-Derived Data:**
- Expressed opinions and preferences
- Health information mentioned in conversation
- Political or religious views expressed
- Employment details
- Family situation details
- Any combination of facts that together identify a person

**Critical Principle:** GDPR applies because conversations themselves ARE data. Even ephemeral processing (data that exists only briefly in RAM during an API call) triggers GDPR obligations. The moment data is read, analyzed, or routed, GDPR relevance begins.

### 2.4 Special Category Data (Article 9)

AI conversations frequently surface special category data incidentally. Under Article 9, the following categories require **explicit consent** or another specific exemption:

- Racial or ethnic origin
- Political opinions
- Religious or philosophical beliefs
- Trade union membership
- Genetic data
- Biometric data (for identification)
- Health data
- Sex life or sexual orientation

**Risk for corteX:** Users may share health concerns, political opinions, or other sensitive data unprompted during AI agent conversations. The SDK must:

1. Enable PII detection that flags special category data
2. Allow customers to configure automatic redaction
3. Prevent special category data from being stored in long-term memory unless explicitly authorized
4. Document the risk in customer-facing privacy documentation

### 2.5 Cross-Border Data Transfers (Articles 44-49)

When personal data leaves the EEA (European Economic Area), GDPR requires one of these transfer mechanisms:

**Adequacy Decision (Art. 45):**
- The European Commission has deemed the destination country provides adequate protection
- Currently adequate: UK, Switzerland, Japan, South Korea, Canada (commercial), Israel, New Zealand, Uruguay, Argentina, and the US (under the EU-US Data Privacy Framework for certified organizations)

**EU-US Data Privacy Framework (DPF):**
- Adopted July 10, 2023 via EU adequacy decision
- Allows free transfer to DPF-certified US organizations
- Survived legal challenge: General Court dismissed annulment action on September 3, 2025
- **2026 Risk Factor:** The Privacy and Civil Liberties Oversight Board (PCLOB) lost Democrat members in January 2025, leaving it without a quorum. This affects the Data Protection Review Court mechanism. Future challenges to the DPF remain possible.
- **Recommendation:** Continue using DPF certification as primary mechanism but maintain SCCs as fallback

**Standard Contractual Clauses (SCCs) (Art. 46(2)(c)):**
- Pre-approved contractual terms adopted by the European Commission
- Current version: June 4, 2021 SCCs (mandatory since December 27, 2022)
- New simplified SCCs expected in 2025-2026 for transfers to non-EU entities already subject to GDPR
- Must be supplemented with a Transfer Impact Assessment (TIA)

**For corteX specifically:**
- On-prem deployments: No transfer issue if data stays within customer's EEA infrastructure
- Cloud (GKE) deployments: Must use EU regions (europe-west1, etc.) OR have transfer mechanisms
- LLM API calls: Each LLM provider's data processing location matters:
  - OpenAI: US-based, DPF-certified, offers EU endpoints for Enterprise
  - Google Gemini: Vertex AI allows EU region selection
  - Anthropic: US-based, moved under Microsoft sub-processor framework in Jan 2026; currently excluded from EU Data Boundary commitments

---

## 3. Key GDPR Articles for AI Systems

### Article 5: Data Processing Principles

The foundational principles that ALL processing must satisfy:

| Principle | GDPR Requirement | corteX Implementation |
|-----------|------------------|----------------------|
| **Lawfulness, fairness, transparency** | Processing must be lawful, fair, and transparent to the data subject | Provide transparency tools; log processing activities; support customer privacy notices |
| **Purpose limitation** | Data collected for specified, explicit purposes; not processed incompatibly | Enforce per-tenant purpose configuration; prevent cross-tenant data leakage; limit learning scope to stated purpose |
| **Data minimization** | Data must be adequate, relevant, and limited to what is necessary | Minimize context windows; strip unnecessary PII before LLM calls; configurable retention levels |
| **Accuracy** | Data must be accurate and kept up to date | Support data correction APIs; allow memory/preference correction; handle rectification requests |
| **Storage limitation** | Data kept only as long as necessary for the purpose | Configurable retention periods per tenant; automatic deletion; TTL on memory stores |
| **Integrity and confidentiality** | Appropriate security measures for data protection | Encryption at rest/transit; access controls; audit logging; tenant isolation |
| **Accountability** | Controller must demonstrate compliance | ROPA support; audit trails; DPA framework; DPIA assistance |

### Article 6: Lawful Basis for Processing

See Section 2.2 above. corteX must technically support all lawful bases that a customer might rely on, including:
- Consent withdrawal mechanisms (immediate cessation of processing)
- Legitimate interest documentation support
- Contractual necessity scope limitation

### Articles 13-14: Transparency Obligations

The controller must inform data subjects about:
- Identity of controller and DPO
- Purposes and legal basis for processing
- Recipients (including sub-processors like LLM providers)
- Transfer mechanisms for international transfers
- Retention periods
- Data subject rights
- Existence of automated decision-making (including profiling)

**corteX obligation:** Provide customers with accurate, current information about:
- What data corteX processes and where
- Sub-processor list (LLM providers, cloud infrastructure)
- Retention practices
- Technical security measures
- Whether and how the brain-inspired learning constitutes "profiling"

### Articles 15-22: Data Subject Rights

| Right | Article | corteX Must Enable |
|-------|---------|-------------------|
| **Access** | Art. 15 | Export all data held about a specific user in machine-readable format |
| **Rectification** | Art. 16 | Correct inaccurate data in memory, preferences, conversation history |
| **Erasure ("Right to be Forgotten")** | Art. 17 | Delete user data from all stores: conversations, memory, learned weights, episodic memory |
| **Restriction** | Art. 18 | Temporarily halt processing while disputes are resolved (but retain data) |
| **Portability** | Art. 20 | Export data in structured, commonly used, machine-readable format (JSON, CSV, XML) |
| **Objection** | Art. 21 | Allow users to object to processing based on legitimate interest or profiling |
| **Automated Decision-Making** | Art. 22 | Provide human-in-the-loop options; explain AI decisions; allow contesting |

**Response Timeline:** Controllers must respond within 1 month (extendable by 2 months for complex requests). corteX must provide technical APIs that enable customers to fulfill requests within this timeframe.

### Article 22: Automated Decision-Making and Profiling

**This is critical for corteX's brain-inspired learning system.**

**The Rule:** Data subjects have the right NOT to be subject to decisions based solely on automated processing (including profiling) that produce legal effects or similarly significantly affect them.

**Exceptions (where automated decisions ARE allowed):**
1. Necessary for contract performance
2. Authorized by EU/Member State law
3. Based on explicit consent

**When exceptions apply, controllers must still provide:**
- Right to obtain human intervention
- Right to express their point of view
- Right to contest the decision
- Meaningful information about the logic involved

**corteX-Specific Analysis:**

The brain-inspired engine (synaptic weights, plasticity, prediction/surprise) constitutes **profiling** under GDPR when it:
- Evaluates personal aspects of a natural person
- Analyzes or predicts behavior, preferences, interests
- Adapts responses based on learned user patterns

**Required corteX Features:**
- Configuration option to disable per-user learning/adaptation
- Explainability API: ability to explain why the agent made a specific decision
- Human-in-the-loop insertion points in the orchestration pipeline
- Clear documentation that brain-inspired learning = profiling under GDPR
- Customer-configurable profiling opt-out mechanism

### Article 25: Data Protection by Design and by Default

**By Design:** Technical and organizational measures must be integrated into processing from the start. For corteX:
- PII detection built into the pipeline (not an afterthought)
- Encryption enabled by default
- Minimum data collection as the default configuration
- Privacy-preserving architecture choices

**By Default:** Only personal data necessary for each specific purpose is processed by default:
- Default retention should be minimal (SESSION or NONE)
- Long-term memory should be opt-in, not opt-out
- Cross-tenant data sharing must be impossible by default
- Learning/profiling features should be explicitly enabled per tenant

### Article 28: Processor Obligations (Core for corteX)

This is corteX's primary obligation. As a processor, corteX must:

1. **Process only on documented instructions** from the controller (customer)
2. **Ensure confidentiality** - all persons authorized to process data must be bound by confidentiality
3. **Implement appropriate security measures** per Article 32
4. **Respect sub-processor conditions** - engage sub-processors only with prior authorization; impose equivalent obligations
5. **Assist the controller** with data subject rights fulfillment
6. **Assist with DPIA** and prior consultation obligations
7. **Delete or return data** after processing ends (at controller's choice)
8. **Make available information** to demonstrate compliance; allow audits
9. **Inform the controller** if an instruction infringes GDPR

**All of this must be formalized in a Data Processing Agreement (DPA).**

### Article 30: Records of Processing Activities (ROPA)

corteX must maintain processor ROPA containing:
- Name and contact details of the processor (Questo) and each controller (customer)
- Categories of processing carried out on behalf of each controller
- International transfers and transfer mechanisms used
- General description of technical and organizational security measures

### Article 32: Security of Processing

Appropriate technical and organizational measures including (as appropriate):
- Pseudonymization and encryption of personal data
- Ability to ensure ongoing confidentiality, integrity, availability, and resilience
- Ability to restore availability and access in a timely manner after incidents
- Regular testing and evaluation of security measures

### Articles 33-34: Breach Notification

**Critical 72-hour rule:**

**Processor to Controller (Art. 33(2)):** corteX must notify affected customers **without undue delay** after becoming aware of a breach. Best practice: within 24 hours to give customers time to meet their 72-hour obligation.

**Controller to Supervisory Authority (Art. 33(1)):** The customer must notify their DPA within **72 hours** of becoming aware. The notification must include:
- Nature of the breach
- Categories and approximate number of affected data subjects
- Name/contact of DPO
- Likely consequences
- Measures taken or proposed

**Controller to Data Subjects (Art. 34):** Required when the breach is likely to result in a **high risk** to rights and freedoms. Must describe the breach in clear, plain language.

### Article 35: Data Protection Impact Assessment (DPIA)

A DPIA is **mandatory** when processing is likely to result in high risk to rights and freedoms. This includes:
- Systematic and extensive profiling with significant effects
- Large-scale processing of special category data
- Systematic monitoring of publicly accessible areas

**For corteX customers:** Use of brain-inspired AI that learns from and adapts to individual users almost certainly requires a DPIA. corteX should:
- Provide a DPIA template tailored to the SDK's features
- Supply technical documentation needed for the DPIA
- Assist customers in conducting DPIAs (processor obligation under Art. 28(3)(f))
- Maintain its own internal DPIA for the SDK platform

---

## 4. AI-Specific GDPR Challenges

### 4.1 EU AI Act Interaction with GDPR

**Current Status (February 2026):**

The EU AI Act (Regulation 2024/1689) is partially in force:
- **February 2, 2025:** Prohibited AI practices became enforceable
- **August 2, 2026:** High-risk AI system requirements become enforceable (Annex III systems)
- **EU Digital Omnibus Proposal (late 2025):** May delay some high-risk requirements to December 2027

**Dual Compliance Requirement:**
AI systems must comply with BOTH GDPR and the AI Act. The AI Act's risk-based structure aligns with GDPR principles:

| AI Act Requirement | GDPR Equivalent |
|-------------------|-----------------|
| Risk assessment (AI Act Art. 9) | Data Protection Impact Assessment (Art. 35) |
| Data governance (AI Act Art. 10) | Data protection by design (Art. 25) |
| Transparency (AI Act Art. 13) | Information obligations (Arts. 13-14) |
| Human oversight (AI Act Art. 14) | Automated decision-making safeguards (Art. 22) |
| Record-keeping (AI Act Art. 12) | Records of processing activities (Art. 30) |
| Post-market monitoring (AI Act Art. 72) | Ongoing compliance (Art. 5(2)) |

**corteX Classification Under AI Act:**
- The SDK itself is likely a **general-purpose AI system component**
- Customer deployments may be classified as **high-risk** depending on use case (HR, credit, healthcare)
- corteX must provide sufficient documentation and transparency to enable customer compliance

**Penalties:**
- AI Act: Up to 35M EUR or 7% global revenue
- GDPR: Up to 20M EUR or 4% global revenue
- Combined exposure is significant

### 4.2 Right to Explanation for AI Decisions

While GDPR does not contain an explicit "right to explanation," Articles 13(2)(f), 14(2)(g), and 15(1)(h) require providing:

> "meaningful information about the logic involved, as well as the significance and the envisaged consequences of such processing"

**For corteX:**
- The orchestrator (prefrontal cortex) must be able to produce decision traces
- Goal tracking, weight adjustments, and tool selections must be loggable
- Customers must be able to explain to their users why the AI agent took specific actions
- The brain-inspired engine's weight system must be interpretable at some level

### 4.3 Data Minimization vs AI Training/Learning

**The Tension:** Brain-inspired learning (synaptic weights, plasticity, episodic memory) inherently wants MORE data to improve. GDPR demands the MINIMUM data necessary.

**Resolution Approaches:**

1. **Strict Purpose Binding:** Each data element used for learning must map to a specific, declared purpose
2. **Aggregation:** Learn patterns, not individual data points. Synaptic weights should abstract away from raw personal data
3. **Differential Privacy:** Add mathematical noise to learning updates to prevent extraction of individual data
4. **Federated Learning:** Keep raw data on-prem; only share anonymized model updates
5. **TTL on Training Data:** Automatically expire training inputs after weights are updated
6. **Opt-In Learning:** Make per-user learning an explicit opt-in feature

### 4.4 Purpose Limitation with Adaptive Learning

When an AI system "learns from interactions," purpose limitation becomes complex:

- **Original Purpose:** Answer the user's question
- **Derived Purpose:** Improve future answers by learning preferences
- **Further Purpose:** Optimize the overall system based on aggregate patterns

Each level requires either:
- Compatibility with the original purpose (Art. 6(4) compatibility test)
- A separate legal basis
- Fresh consent

**corteX Architecture Decision:** The brain-inspired engine should maintain purpose metadata on all stored data, enabling:
- Per-purpose retention policies
- Per-purpose deletion (erase learning data but keep audit logs)
- Purpose compatibility checks before cross-use

### 4.5 LLM Data Flow and Controller/Processor Roles

When data flows through the LLM sub-processor chain:

```
User Input (personal data)
    |
    v
corteX SDK (Processor) - may redact/pseudonymize
    |
    v
LLM API (Sub-Processor) - processes in context window
    |
    v
LLM Provider Infrastructure (Sub-Sub-Processor)
```

**Key Questions and Answers:**

**Q: Who is the controller when data goes to OpenAI/Anthropic?**
A: The customer remains the controller. corteX is the processor. The LLM provider is a sub-processor. The customer's DPA with corteX must authorize this sub-processing. corteX's DPA with the LLM provider must impose equivalent obligations.

**Q: Does the LLM provider become a controller?**
A: Only if they process data for their own purposes (e.g., training). Enterprise API tiers of OpenAI, Google, and Anthropic all commit to NOT training on customer data. This must be contractually guaranteed via DPA.

**Q: What about data in the context window?**
A: Even ephemeral processing in RAM constitutes "processing" under GDPR. However, if the LLM provider does not store the data and uses zero-data-retention policies, the risk is significantly reduced.

**Mitigation Strategy for corteX:**
1. Strip unnecessary PII before LLM API calls (data minimization)
2. Use PII placeholder substitution (replace real names/emails with tokens before sending to LLM)
3. Require enterprise-tier LLM APIs with contractual no-training guarantees
4. Offer on-prem/local model option for maximum data control
5. Document the sub-processor chain in customer DPA

### 4.6 EDPB Opinion on AI Models and Personal Data (December 2024)

The EDPB's Opinion 28/2024 provides critical guidance:

1. **Anonymization of AI Models:** For a model to be considered anonymous (and thus outside GDPR scope), it must be "very unlikely" that individuals can be identified through the model or that personal data can be extracted via queries. This is assessed case-by-case.

2. **Legitimate Interest for AI:** Confirmed as a valid legal basis for AI development and deployment, subject to the three-step test:
   - Identify a legitimate interest
   - Demonstrate necessity of processing
   - Conduct a balancing exercise against data subject rights

3. **Unlawful Training Data:** If an AI model was developed using unlawfully processed personal data, deployment may also be unlawful UNLESS the model has been "duly anonymized."

4. **LLMs Rarely Achieve Anonymization:** The EDPB clarified that large language models rarely achieve true anonymization standards. Controllers deploying third-party LLMs must conduct comprehensive legitimate interest assessments.

---

## 5. Required Documents and Policies

### 5.1 Data Processing Agreement (DPA)

**When Required:** Before any customer deploys corteX with personal data processing.

**Mandatory Clauses (per Article 28):**

1. **Subject Matter and Duration**
   - Description of processing activities
   - Duration of the agreement (tied to service agreement)

2. **Nature and Purpose of Processing**
   - AI agent orchestration
   - Conversation processing
   - Memory management
   - Learning/adaptation (if enabled)
   - Tool execution

3. **Types of Personal Data**
   - Conversation content
   - User identifiers
   - Behavioral patterns and preferences
   - Any data processed through tools

4. **Categories of Data Subjects**
   - End users of the customer's application
   - Customer employees (if applicable)

5. **Controller Instructions**
   - Processing only on documented instructions
   - Customer can specify: retention periods, geographic restrictions, features enabled/disabled

6. **Confidentiality**
   - All personnel authorized to process must be bound by confidentiality

7. **Security Measures (Article 32)**
   - Encryption specifications (AES-256 at rest, TLS 1.3 in transit)
   - Access control mechanisms
   - Audit logging capabilities
   - Incident response procedures

8. **Sub-Processor Management**
   - List of current sub-processors (LLM providers, cloud infrastructure)
   - 30-day advance notification of sub-processor changes
   - Customer objection rights
   - Requirement to impose equivalent data protection obligations on sub-processors

9. **Assistance with Data Subject Rights**
   - Technical and organizational measures to assist controller
   - APIs for data access, rectification, erasure, portability
   - Response time commitments

10. **DPIA Assistance**
    - Obligation to provide information needed for DPIAs
    - Template and documentation support

11. **Breach Notification**
    - Notification timeline (recommended: 24 hours to customer)
    - Content of notification
    - Cooperation obligations

12. **Data Return/Deletion**
    - At end of service: return or delete all personal data (customer's choice)
    - Certification of deletion
    - Timeline for completion

13. **Audit Rights**
    - Customer right to audit or appoint auditor
    - Alternative: SOC 2 Type II / ISO 27001 reports
    - Frequency limitations (e.g., annually, with reasonable notice)

14. **International Transfers**
    - Applicable transfer mechanisms (DPF, SCCs)
    - Transfer Impact Assessment obligations
    - Data residency options

15. **Liability**
    - Allocation of liability for breaches
    - Indemnification terms
    - Liability caps

### 5.2 Privacy Policy

corteX (Questo) needs its own privacy policy covering:
- Processing activities as a processor
- Data collected from website visitors
- Cookie usage
- Marketing communications
- Contact information for privacy inquiries

Additionally, corteX should provide a **privacy policy template/guidance** for customers to adapt, covering how the AI agent processes end-user data.

### 5.3 Records of Processing Activities (ROPA)

**Processor ROPA must include:**

| Field | corteX Entry |
|-------|-------------|
| Processor name & contact | Questo Ltd, [DPO contact] |
| Controller names | [List of all customers] |
| Categories of processing | AI agent orchestration, conversation processing, memory management, LLM API routing, tool execution |
| International transfers | [Per-customer: where data is transferred and mechanism used] |
| Security measures | Encryption (AES-256/TLS 1.3), access controls, audit logging, tenant isolation, PII detection |

**Maintenance:** ROPA must be updated whenever processing activities change. Recommend quarterly review plus event-driven updates.

### 5.4 Data Protection Impact Assessment (DPIA)

**When Required:**
A DPIA is required when processing is "likely to result in a high risk to the rights and freedoms of natural persons." For AI systems, this is nearly always the case when:
- The system profiles individuals
- Special category data may be processed
- Large-scale processing occurs
- New technology is used

**DPIA Template Structure:**

1. **Systematic Description of Processing**
   - What data is processed
   - How data flows through the system (SDK -> LLM -> response)
   - Purposes of processing
   - Data retention periods
   - Recipients (sub-processors)

2. **Necessity and Proportionality Assessment**
   - Why this processing is necessary
   - Could the purpose be achieved with less data?
   - Is the processing proportionate to the purpose?

3. **Risk Assessment**
   - Risks to data subjects (discrimination, identity theft, financial loss, reputational damage)
   - Likelihood and severity of each risk
   - Special risks from AI/profiling
   - Risks from international transfers

4. **Mitigation Measures**
   - Technical measures (encryption, pseudonymization, access controls)
   - Organizational measures (policies, training, audits)
   - Data minimization strategies
   - Human oversight mechanisms

5. **Consultation**
   - DPO opinion
   - If residual risk remains high: prior consultation with supervisory authority (Art. 36)

**The DPIA should be treated as a "living document" and reviewed regularly, especially when processing changes or when "concept drift" occurs in the AI system.**

### 5.5 Sub-Processor List

Must be publicly maintained and include:

| Sub-Processor | Purpose | Location | Transfer Mechanism |
|---------------|---------|----------|-------------------|
| OpenAI | LLM inference (when customer selects OpenAI provider) | USA | EU-US DPF + SCCs |
| Google Cloud (Vertex AI) | LLM inference (Gemini provider), Cloud hosting | EU/US (configurable) | Adequacy (EU regions) / DPF + SCCs (US) |
| Anthropic | LLM inference (when customer selects Claude provider) | USA | SCCs + TIA |
| [Cloud Provider] | Infrastructure hosting | [Per deployment] | [Per deployment] |

**Notification Obligation:** Customers must be notified at least 30 days before any sub-processor addition/change, with the right to object.

### 5.6 Data Breach Response Procedure

**Phase 1: Detection & Assessment (0-4 hours)**
1. Detect or receive report of potential breach
2. Activate incident response team
3. Contain the breach (isolate affected systems)
4. Assess: Is personal data affected? What types? How many records?
5. Determine: Is this a "personal data breach" under GDPR?

**Phase 2: Notification to Controllers (4-24 hours)**
1. Notify all affected customers (controllers) with:
   - Nature of the breach
   - Categories and approximate number of affected data subjects/records
   - Likely consequences
   - Measures taken or proposed to mitigate
2. Provide ongoing updates as investigation progresses

**Phase 3: Support Controller Notification (24-72 hours)**
1. Assist customers in preparing their notification to supervisory authorities
2. Provide technical details needed for the notification
3. Support customers in determining whether data subject notification is required

**Phase 4: Remediation & Documentation (72 hours+)**
1. Complete root cause analysis
2. Implement fixes
3. Document everything (required by Art. 33(5))
4. Review and update security measures
5. Conduct lessons-learned review

### 5.7 Data Subject Request Handling Procedure

corteX's role is to **assist** the customer (controller) in fulfilling requests. The SDK must provide:

**Technical APIs:**

```python
# Data access (Article 15)
await cortex.gdpr.export_user_data(tenant_id, user_id, format="json")

# Data rectification (Article 16)
await cortex.gdpr.rectify_user_data(tenant_id, user_id, corrections={...})

# Data erasure (Article 17)
await cortex.gdpr.erase_user_data(tenant_id, user_id, scope="all")
# scope options: "all", "conversations", "memory", "learning", "preferences"

# Data restriction (Article 18)
await cortex.gdpr.restrict_processing(tenant_id, user_id, restricted=True)

# Data portability (Article 20)
await cortex.gdpr.export_portable_data(tenant_id, user_id, format="json")

# Processing objection (Article 21)
await cortex.gdpr.register_objection(tenant_id, user_id, scope="profiling")
```

**Response SLA:** corteX should respond to customer requests within 5 business days to give customers sufficient time to meet the 1-month deadline.

---

## 6. Technical Requirements

### 6.1 Encryption

**At Rest:**
- AES-256 encryption for all stored personal data
- Per-tenant encryption keys (for cryptographic isolation)
- Key management: HSM or cloud KMS, with key rotation
- Encrypted backups

**In Transit:**
- TLS 1.3 for all API communications
- TLS 1.2 minimum (1.0 and 1.1 must be disabled)
- Certificate pinning for LLM provider API calls
- mTLS for internal service-to-service communication (cloud deployments)

### 6.2 Pseudonymization and Anonymization

**Pseudonymization Techniques (data remains personal data under GDPR):**

| Technique | Use Case in corteX | Implementation |
|-----------|-------------------|----------------|
| Tokenization | Replace PII before LLM API calls | Map real values to tokens; restore in response |
| Hashing | User ID pseudonymization in logs | SHA-256 with salt; store mapping separately |
| Encryption | Conversation storage | AES-256 with per-tenant keys |
| Key-coding | Cross-reference protection | Replace identifiers with codes; store key separately |

**Anonymization Techniques (data falls outside GDPR):**

| Technique | Use Case | Notes |
|-----------|----------|-------|
| Aggregation | Analytics, usage metrics | Combine data from 5+ users minimum |
| Generalization | Demographic analysis | Replace specific values with ranges |
| Differential Privacy | Learning/weight updates | Add calibrated noise to prevent re-identification |
| Data suppression | Removing outliers | Remove records that could identify individuals |

**EDPB Standard:** True anonymization requires that re-identification is not "reasonably likely" using "all means reasonably likely to be used." For AI models, this is a high bar.

### 6.3 Data Retention and Automatic Deletion

**Configurable Retention Tiers:**

```python
class DataRetention(Enum):
    NONE = "none"           # No data persisted (ephemeral processing only)
    SESSION = "session"     # Deleted at session end
    SHORT_TERM = "7d"       # 7-day retention
    MEDIUM_TERM = "30d"     # 30-day retention
    LONG_TERM = "90d"       # 90-day retention
    CUSTOM = "custom"       # Customer-defined period
    AUDIT_ONLY = "audit"    # Only audit logs retained (no personal data)
```

**Automatic Deletion Requirements:**
- Scheduled deletion jobs that run daily
- Cryptographic erasure option (destroy encryption key to render data unreadable)
- Cascading deletion across all data stores (conversations, memory, weights, episodic memory)
- Deletion verification and logging
- Grace period handling (restricted processing during grace period)

### 6.4 Right to Erasure Implementation

**The Machine Unlearning Challenge:**

corteX's brain-inspired learning (synaptic weights, plasticity) presents a unique challenge for the right to erasure. Once data influences model weights, "deleting" that influence is non-trivial.

**Implementation Strategy:**

1. **Conversation and Memory Deletion:** Straightforward - delete records from database
2. **Preference Deletion:** Delete stored preferences and reset to defaults
3. **Weight/Learning Deletion:** Multiple approaches:
   - **Modular Training:** Maintain per-user weight deltas separate from base weights. Deletion = remove the delta.
   - **Re-training from scratch:** Expensive but thorough. Re-train the model excluding the deleted user's data.
   - **Machine Unlearning:** Apply unlearning algorithms to approximately remove the influence of specific data points.
   - **Architectural Isolation:** Design the weight system so per-user adaptations are stored as separate, deletable layers.

4. **Episodic Memory Deletion:** Delete all episodes associated with the user

**GDPR Standard:** Article 17 and Recital 66 require "reasonable steps, taking into account available technology and the cost of implementation." Perfect mathematical unlearning is not required, but good-faith, documented effort is.

**Recommended corteX Architecture:**
```
Base Model Weights (shared, no personal data)
    |
    +-- Tenant Weight Deltas (per-tenant, deletable)
    |       |
    |       +-- User Weight Deltas (per-user, deletable)
    |
    +-- Episodic Memory (per-user, deletable)
    |
    +-- Preference Store (per-user, deletable)
```

This architecture makes deletion straightforward: remove the user's delta layer, episodic memory, and preferences. The base model is unaffected.

### 6.5 Data Portability

**Required Format:** Structured, commonly used, machine-readable. Recommended:

```json
{
  "export_format": "cortex_gdpr_export_v1",
  "export_date": "2026-02-16T12:00:00Z",
  "data_subject": {
    "user_id": "usr_abc123",
    "tenant_id": "tenant_xyz"
  },
  "conversations": [
    {
      "id": "conv_001",
      "created_at": "2026-01-15T10:30:00Z",
      "messages": [
        {"role": "user", "content": "..."},
        {"role": "assistant", "content": "..."}
      ]
    }
  ],
  "memory": {
    "working_memory": {...},
    "short_term": [...],
    "long_term": [...],
    "episodic": [...]
  },
  "preferences": {
    "learned_preferences": {...},
    "explicit_settings": {...}
  },
  "processing_history": {
    "tools_used": [...],
    "decisions_made": [...]
  }
}
```

### 6.6 Access Logging and Audit Trails

**Minimum Audit Requirements:**

| Event | Data Logged | Retention |
|-------|------------|-----------|
| Data access | Who, when, what data, purpose | Per compliance framework |
| Data modification | Who, when, what changed, before/after | Per compliance framework |
| Data deletion | Who, when, what deleted, confirmation | Permanent (audit evidence) |
| LLM API calls | Timestamp, provider, data categories sent (NOT content) | 90 days minimum |
| Authentication | User/service, timestamp, success/failure | 365 days |
| Configuration changes | Who, what changed, before/after | 365 days |
| Data subject requests | Request type, timestamp, fulfillment status | 3 years |

**Important:** Audit logs themselves must not contain personal data beyond what is necessary. Log that "user X's data was accessed" but do not log the actual data content.

### 6.7 Consent Management

**Technical Requirements:**
- Record consent: who, when, what they consented to, how consent was obtained
- Granular consent: separate consent for different processing purposes
- Withdrawal mechanism: as easy to withdraw as to give
- Automatic processing halt: when consent is withdrawn, processing must stop immediately
- Proof of consent: maintain evidence that consent was validly obtained

```python
class ConsentRecord:
    user_id: str
    tenant_id: str
    purpose: str  # e.g., "personalization", "analytics", "learning"
    granted: bool
    granted_at: datetime
    withdrawn_at: Optional[datetime]
    consent_mechanism: str  # e.g., "checkbox", "API", "explicit_opt_in"
    consent_version: str  # version of privacy policy at time of consent
```

### 6.8 Data Minimization in AI Context Windows

**The Problem:** LLM context windows can be very large (1M+ tokens for Gemini). It is tempting to include extensive conversation history and user data for better responses. This conflicts with data minimization.

**Best Practices:**

1. **Relevance Filtering:** Only include conversation history relevant to the current query
2. **PII Stripping:** Remove unnecessary PII from context before sending to LLM
3. **Summary Compression:** Replace old conversations with anonymized summaries
4. **Token Budgeting:** Set maximum context sizes per data category:
   - Recent conversation: 60% of budget
   - Relevant memory: 20% of budget
   - System instructions: 15% of budget
   - User preferences: 5% of budget
5. **Placeholder Substitution:** Replace real PII with consistent placeholders before LLM calls, restore in responses

---

## 7. Multi-Tenant AI SDK Requirements

### 7.1 Per-Tenant Data Isolation

**Isolation Tiers:**

| Tier | Approach | GDPR Suitability | corteX Default |
|------|----------|-------------------|----------------|
| **Logical Isolation** | Shared database, tenant_id column, row-level security | Acceptable for most cases | Yes (Starter/Pro) |
| **Schema Isolation** | Separate database schema per tenant | Better for regulated industries | Available (Enterprise) |
| **Database Isolation** | Separate database per tenant | Strongest isolation | Available (Unlimited) |
| **Infrastructure Isolation** | Separate compute/storage per tenant | Maximum isolation (on-prem equivalent) | Available (Unlimited) |

**Minimum Requirements (All Tiers):**

- Every database query must include tenant_id filter (enforced at ORM/query layer)
- Cross-tenant data access must be architecturally impossible (not just policy)
- Memory systems (working, short-term, long-term, episodic) must be tenant-scoped
- Brain engine weights must be tenant-isolated
- LLM context windows must never contain cross-tenant data
- Audit logs must be tenant-filterable

### 7.2 Per-Tenant Retention Policies

```python
tenant_config = TenantConfig(
    tenant_id="acme_corp",
    data_retention=DataRetention.CUSTOM,
    retention_config=RetentionConfig(
        conversations=timedelta(days=30),
        working_memory=timedelta(hours=24),
        short_term_memory=timedelta(days=7),
        long_term_memory=timedelta(days=90),
        episodic_memory=timedelta(days=90),
        audit_logs=timedelta(days=365),
        learned_preferences=timedelta(days=180),
    ),
)
```

Each tenant must be able to set independent retention periods for every data category.

### 7.3 Per-Tenant Deletion

**Requirements:**

1. **User-Level Deletion:** Delete all data for a specific user within a tenant
2. **Tenant-Level Deletion:** Delete ALL data for an entire tenant (offboarding)
3. **Category-Level Deletion:** Delete specific data categories for a user or tenant
4. **Cascading Deletion:** Ensure deletion propagates across all data stores
5. **Verification:** Confirm deletion completed successfully; produce certification
6. **Backup Handling:** Ensure deleted data is also purged from backups within a defined period

### 7.4 Sub-Processor Chain Management

```
Customer A (Controller)
    |
    v
corteX (Processor) -- DPA with Customer A
    |
    +-> OpenAI (Sub-Processor) -- DPA between corteX and OpenAI
    |       |
    |       +-> Azure (Sub-Sub-Processor)
    |
    +-> Google Vertex AI (Sub-Processor) -- DPA via Google CDPA
    |       |
    |       +-> Google Cloud (Sub-Sub-Processor)
    |
    +-> Anthropic (Sub-Processor) -- DPA between corteX and Anthropic
            |
            +-> AWS/Microsoft (Sub-Sub-Processor)
```

**Management Requirements:**

- Maintain up-to-date sub-processor list (publicly accessible)
- 30-day advance notice to customers before adding/changing sub-processors
- Customer objection mechanism (right to terminate if they object)
- Equivalent data protection obligations imposed on each sub-processor via DPA
- Regular review of sub-processor compliance
- Customers can choose which LLM providers are allowed for their tenant

### 7.5 Customer DPA Requirements

Every customer must sign a DPA before processing begins. The DPA should be:

- **Available as a standard document** (not negotiated from scratch each time)
- **Customizable per customer** for: retention periods, geographic restrictions, sub-processor approvals, audit frequency
- **Version-controlled** with customer acceptance tracked
- **Reviewed annually** and updated when processing changes

**Common Enterprise Customer Negotiation Points:**

| Point | Customer Wants | Reasonable Position |
|-------|---------------|-------------------|
| Liability caps | Unlimited liability | Cap at contract value or insurance coverage |
| Audit rights | On-site audits anytime | Annual on-site OR SOC 2/ISO 27001 report |
| Breach notification | Immediate | Within 24 hours of awareness |
| Indemnification | Full indemnification | Capped, mutual indemnification |
| Sub-processor approval | Approve each individually | General authorization with objection right |
| Data residency | EU-only | Offer EU region option; default to EU |

### 7.6 Data Residency Options

For multi-tenant deployments, offer:

| Option | Description | Use Case |
|--------|-------------|----------|
| **EU-Only** | All processing within EEA | EU customers with strict requirements |
| **US-Only** | All processing within US | US customers |
| **Regional** | Customer chooses region | Multi-national customers |
| **On-Prem** | Customer's own infrastructure | Maximum control (healthcare, government) |

---

## 8. 2026 Enforcement Landscape

### 8.1 Recent GDPR Fines Relevant to AI/SaaS

| Entity | Fine | Reason | Year | Relevance to corteX |
|--------|------|--------|------|---------------------|
| **OpenAI** | 15M EUR | No lawful basis for ChatGPT training data; transparency failures | 2024 | Direct precedent for AI data processing |
| **Clearview AI** | 100M+ EUR (cumulative) | Facial recognition; scraping; unlawful processing | 2020-2024 | AI + personal data enforcement trend |
| **Meta** | 479M EUR | Consent manipulation for data processing | 2023 | Consent must be freely given |
| **TikTok** | 530M EUR | Illegal data transfers to China | 2023 | Cross-border transfer enforcement |
| **Vodafone** | 45M EUR | Vendor security failures | 2024 | Processor/sub-processor accountability |
| **Google** | 100M EUR (CNIL) | Cookie consent dark patterns | 2022 | Dark patterns in consent mechanisms |

**Key Takeaway:** Enforcement is intensifying. From 2018-2025, over 2,800 fines totaling 6.2B+ EUR were issued. Over 60% (3.8B+ EUR) came since January 2023. AI-specific enforcement is now a priority.

### 8.2 EU AI Act Compliance Timeline

| Date | Milestone | Impact on corteX |
|------|-----------|-----------------|
| Feb 2, 2025 | Prohibited practices enforceable | Ensure no prohibited practices (emotion recognition in workplace, social scoring) |
| Aug 2, 2025 | General-purpose AI obligations | Transparency requirements for GPAI models |
| Aug 2, 2026 | High-risk AI system requirements | Customer deployments in high-risk domains need conformity assessment |
| Aug 2, 2027 | All remaining provisions | Full enforcement |

**Note:** The EU Digital Omnibus proposal (late 2025) may delay some high-risk system requirements to December 2027.

### 8.3 DPA (Data Protection Authority) Guidance on AI

**Key Guidance Documents:**

- **EDPB Opinion 28/2024** (December 2024): AI models and personal data. Covers anonymization, legitimate interest, unlawful training data consequences.

- **CNIL Recommendations** (February 2025):
  - "AI: Informing Data Subjects" - how to provide transparency about AI processing
  - "AI: Complying and Facilitating Individuals' Rights" - implementing data subject rights for AI
  - Key principle: Personal data used to train AI may be memorized by it; individuals must be informed

- **Irish DPC Guidance** (July 2024): "AI, Large Language Models and Data Protection" - practical guidance on LLM compliance

- **CNIL on Legitimate Interest for AI Training** (June 2025): Confirmed that legitimate interest can be a valid basis for AI training, with specific guidance on the balancing test

### 8.4 EDPB (European Data Protection Board) Guidelines

**Current and Upcoming:**

- **Opinion 28/2024:** AI models and data protection (adopted December 2024)
- **Upcoming:** Guidelines on anonymization, pseudonymization, and data scraping in generative AI
- **2026 Priority:** Transparency enforcement - DPAs are coordinating joint enforcement actions focused on transparency in 2026
- **EDPB April 2025 Report:** LLMs rarely achieve anonymization standards; controllers must conduct comprehensive legitimate interest assessments

---

## 9. Implementation Checklist for corteX

### Phase 1: Foundation (Must-Have Before Any EU Customer)

- [ ] **DPA template** drafted and reviewed by legal counsel
- [ ] **Sub-processor list** published and maintained
- [ ] **ROPA** created and maintained
- [ ] **Privacy policy** published
- [ ] **Encryption** at rest (AES-256) and in transit (TLS 1.3) implemented
- [ ] **Tenant isolation** enforced at data layer
- [ ] **Data deletion API** for user-level and tenant-level erasure
- [ ] **Data export API** in machine-readable format (JSON)
- [ ] **PII detection** pipeline implemented and configurable
- [ ] **Configurable retention** policies per tenant
- [ ] **Breach notification** procedure documented and tested
- [ ] **Audit logging** for all data access and modifications

### Phase 2: Enhanced Compliance

- [ ] **DPIA template** for customer use
- [ ] **Internal DPIA** for corteX platform completed
- [ ] **Consent management** API implemented
- [ ] **PII redaction** before LLM API calls (placeholder substitution)
- [ ] **Per-tenant encryption keys** (cryptographic isolation)
- [ ] **Data residency** options (EU-only deployment)
- [ ] **Automated retention** enforcement (scheduled deletion jobs)
- [ ] **Processing restriction** API (Article 18)
- [ ] **Objection handling** API (Article 21)
- [ ] **Decision explainability** API (Articles 13-15, 22)

### Phase 3: Advanced Privacy Engineering

- [ ] **Differential privacy** in learning/weight updates
- [ ] **Machine unlearning** capability for per-user weight removal
- [ ] **Modular weight architecture** (base + tenant delta + user delta)
- [ ] **Federated learning** option for on-prem deployments
- [ ] **Automated DPIA** generation based on tenant configuration
- [ ] **Privacy dashboard** for customers (real-time compliance status)
- [ ] **SOC 2 Type II** certification
- [ ] **ISO 27001** certification
- [ ] **EU AI Act** conformity documentation

---

## 10. Sub-Processor Registry

### Current Sub-Processors

| Sub-Processor | Entity | Processing Activities | Data Location | Transfer Mechanism | DPA Status |
|---------------|--------|----------------------|---------------|-------------------|------------|
| OpenAI, Inc. | LLM inference | Conversation processing, response generation | USA (Oregon, Iowa) | EU-US DPF + SCCs | Required before use |
| Google LLC (Vertex AI) | LLM inference, hosting | Conversation processing, response generation | EU (configurable), USA | Adequacy (EU) / DPF + SCCs (US) | Google CDPA |
| Anthropic, PBC | LLM inference | Conversation processing, response generation | USA | SCCs + TIA | Required before use |
| Google Cloud Platform | Infrastructure | Compute, storage, networking (GKE deployments) | EU (configurable), USA | Adequacy (EU) / DPF + SCCs (US) | Google CDPA |

### LLM Provider GDPR Comparison

| Feature | OpenAI Enterprise | Google Vertex AI | Anthropic Business |
|---------|------------------|------------------|--------------------|
| No-training commitment | Yes (API) | Yes | Yes |
| EU data residency | UK + regional options | Yes (EU regions) | No (US only, as of Feb 2026) |
| Zero data retention option | Yes (ZDR endpoint) | Yes | Yes |
| DPA available | Yes | Yes (CDPA) | Yes |
| SOC 2 Type II | Yes | Yes | Yes |
| HIPAA BAA | Yes | Yes | Yes |
| Sub-processor list published | Yes | Yes | Yes |
| EU-US DPF certified | Yes | Yes | Not confirmed |

---

## Appendix A: Key Regulatory References

- **GDPR Full Text**: [Regulation (EU) 2016/679](https://gdpr-info.eu/)
- **EU AI Act Full Text**: [Regulation (EU) 2024/1689](https://artificialintelligenceact.eu/)
- **EDPB Opinion 28/2024**: [AI Models and Data Protection](https://www.edpb.europa.eu/our-work-tools/our-documents/opinion-board-art-64/opinion-282024-certain-data-protection-aspects_en)
- **CNIL AI Recommendations**: [CNIL AI Compliance Guide](https://www.cnil.fr/en/ai-system-development-cnils-recommendations-to-comply-gdpr)
- **EU-US Data Privacy Framework**: [DPF Official Site](https://www.dataprivacyframework.gov/Program-Overview)
- **Standard Contractual Clauses**: [European Commission SCCs](https://commission.europa.eu/law/law-topic/data-protection/international-dimension-data-protection/standard-contractual-clauses-scc_en)
- **ICO Guidance on Automated Decision-Making**: [ICO ADM Guidance](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/individual-rights/automated-decision-making-and-profiling/)

## Appendix B: Glossary

| Term | Definition |
|------|-----------|
| **CDPA** | Cloud Data Processing Addendum (Google's DPA) |
| **DPA** | Data Processing Agreement (GDPR Art. 28 contract) |
| **DPF** | EU-US Data Privacy Framework |
| **DPIA** | Data Protection Impact Assessment (GDPR Art. 35) |
| **DPO** | Data Protection Officer |
| **EDPB** | European Data Protection Board |
| **EEA** | European Economic Area |
| **GDPR** | General Data Protection Regulation (EU 2016/679) |
| **PII** | Personally Identifiable Information |
| **ROPA** | Records of Processing Activities (GDPR Art. 30) |
| **SCCs** | Standard Contractual Clauses |
| **TIA** | Transfer Impact Assessment |
| **ZDR** | Zero Data Retention |

## Appendix C: Sources

- [The SaaS DPA Guide](https://secureprivacy.ai/blog/data-processing-agreements-dpas-for-saas)
- [Complete GDPR Compliance Guide (2026-Ready)](https://secureprivacy.ai/blog/gdpr-compliance-2026)
- [CNIL AI System Development Recommendations](https://www.cnil.fr/en/ai-system-development-cnils-recommendations-to-comply-gdpr)
- [AI Privacy Rules: GDPR, EU AI Act, and U.S. Law](https://www.parloa.com/blog/AI-privacy-2026/)
- [AI and SaaS Contracts 2026](https://www.naumovic-partners.com/en/ai-and-saas-contracts-in-2026/)
- [EU AI Act 2026 Compliance Guide](https://secureprivacy.ai/blog/eu-ai-act-2026-compliance)
- [EDPB Opinion on AI Models](https://www.edpb.europa.eu/news/news/2024/edpb-opinion-ai-models-gdpr-principles-support-responsible-ai_en)
- [GDPR Article 28: Processor](https://gdpr-info.eu/art-28-gdpr/)
- [GDPR Compliant AI Chat: Requirements, Architecture & Setup 2026](https://blog.premai.io/gdpr-compliant-ai-chat-requirements-architecture-setup-2026/)
- [GDPR Data Processing Agreement Template](https://gdpr.eu/data-processing-agreement/)
- [Multi-Tenant SaaS Privacy: Data Isolation and Compliance](https://complydog.com/blog/multi-tenant-saas-privacy-data-isolation-compliance-architecture)
- [GDPR Breach Notification: 72-Hour Rule](https://www.glocertinternational.com/resources/articles/gdpr-breach-notification-72-hour-rule/)
- [GDPR Fines and Lessons 2025-2026](https://www.dsalta.com/resources/articles/gdpr-fines-2025-2026-lessons-how-to-avoid)
- [Italy Fines OpenAI 15M EUR](https://www.euronews.com/next/2024/12/20/italys-privacy-watchdog-fines-openai-15-million-after-probe-into-chatgpt-data-collection)
- [EU-US Data Privacy Framework](https://www.dataprivacyframework.gov/EU-US-Framework)
- [GDPR Compliance Showdown: AI Assistants](https://dovetail.team/blog/ai-assistant-gdpr-compliance-showdown)
- [Navigating GDPR Compliance in LLM Lifecycle](https://www.private-ai.com/en/blog/gdpr-llm-lifecycle)
- [Large Language Models GDPR Compliance](https://gdprlocal.com/large-language-models-llm-gdpr/)
- [Automated Decision Making: GDPR Article 22](https://gdprlocal.com/automated-decision-making-gdpr/)
- [Conducting a DPIA: Best Practices for AI Systems](https://gdprlocal.com/conducting-a-dpia-best-practices-for-ai-systems/)
- [EDPB Guidelines on Pseudonymisation (2025)](https://www.edpb.europa.eu/system/files/2025-01/edpb_guidelines_202501_pseudonymisation_en.pdf)
- [GDPR for Machine Learning](https://gdprlocal.com/gdpr-machine-learning/)
- [Machine Unlearning and the Right to Erasure](https://www.sciencedirect.com/science/article/pii/S026736492300095X)
- [GDPR for SaaS Companies: Complete Compliance Guide](https://complydog.com/blog/gdpr-for-saas-companies-complete-compliance-guide)
