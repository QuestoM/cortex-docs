# Data Processing Agreement

**Between Questo Ltd ("Processor") and [Customer Name] ("Controller")**

---

| Document Control |  |
|---|---|
| **Document ID** | GDPR-DPA-001 |
| **Version** | 1.0 |
| **Effective Date** | [Effective Date] |
| **Classification** | Confidential |
| **Owner** | Questo Ltd, Legal & Compliance |
| **Last Reviewed** | 2026-02-16 |
| **Next Review** | 2026-08-16 |

---

## Preamble

This Data Processing Agreement ("DPA") forms part of the Master Service Agreement or Terms of Service (the "Principal Agreement") between:

**Controller:**
- Legal Name: [Customer Legal Name]
- Registered Address: [Customer Address]
- Registration Number: [Customer Registration Number]
- Contact Person: [Customer Contact Name]
- Contact Email: [Customer Contact Email]

**Processor:**
- Legal Name: Questo Ltd
- Registered Address: Israel
- Registration Number: [Questo Registration Number]
- Contact Person: Privacy Team, Questo Ltd
- Contact Email: privacy@questo.io

(each a "Party" and together the "Parties")

This DPA is entered into pursuant to Article 28 of Regulation (EU) 2016/679 (the "GDPR") and reflects the Parties' agreement with regard to the processing of Personal Data by the Processor on behalf of the Controller in connection with the corteX AI Agent SDK platform (the "Service").

Where this DPA conflicts with the Principal Agreement, this DPA shall prevail with respect to data protection matters.

---

## 1. Definitions

**"Applicable Data Protection Law"** means the GDPR, together with any national implementing legislation in any Member State of the EEA, the UK GDPR and UK Data Protection Act 2018, the Swiss Federal Act on Data Protection, and any other applicable data protection and privacy legislation.

**"Controller"** has the meaning given in Article 4(7) of the GDPR: the natural or legal person which determines the purposes and means of the processing of Personal Data.

**"Data Subject"** means an identified or identifiable natural person whose Personal Data is processed under this DPA.

**"EEA"** means the European Economic Area, comprising EU Member States plus Iceland, Liechtenstein, and Norway.

**"Personal Data"** has the meaning given in Article 4(1) of the GDPR: any information relating to an identified or identifiable natural person.

**"Personal Data Breach"** has the meaning given in Article 4(12) of the GDPR: a breach of security leading to the accidental or unlawful destruction, loss, alteration, unauthorised disclosure of, or access to, Personal Data transmitted, stored or otherwise processed.

**"Processing"** has the meaning given in Article 4(2) of the GDPR: any operation or set of operations performed on Personal Data, whether or not by automated means.

**"Processor"** has the meaning given in Article 4(8) of the GDPR: a natural or legal person which processes Personal Data on behalf of the Controller.

**"Service"** means the corteX AI Agent SDK platform, including all features, APIs, and related services provided by Questo Ltd under the Principal Agreement.

**"Standard Contractual Clauses" or "SCCs"** means the standard contractual clauses approved by the European Commission in Implementing Decision (EU) 2021/914, as may be amended or replaced from time to time.

**"Sub-Processor"** means any third party engaged by the Processor to process Personal Data on behalf of the Controller in connection with the Service.

**"Supervisory Authority"** means an independent public authority established by an EU Member State pursuant to Article 51 of the GDPR.

---

## 2. Subject Matter and Duration of Processing

### 2.1 Subject Matter
This DPA governs the processing of Personal Data by the Processor in connection with the provision of the corteX AI Agent SDK platform. The Processor provides an enterprise-grade AI agent orchestration engine that the Controller integrates into its applications to deliver AI-powered features to its end users.

### 2.2 Duration
This DPA shall remain in effect for the duration of the Principal Agreement between the Parties. Upon termination or expiration of the Principal Agreement, this DPA shall continue to apply until all Personal Data has been returned or deleted in accordance with Section 12 of this DPA.

### 2.3 Governing Law
This DPA shall be governed by and construed in accordance with the laws that govern the Principal Agreement, unless otherwise required by Applicable Data Protection Law.

---

## 3. Nature and Purpose of Processing

### 3.1 Nature of Processing
The Processor performs the following processing operations in connection with the Service:

| Processing Activity | Description |
|---|---|
| **AI Agent Orchestration** | Coordinating multi-step AI agent workflows, including goal tracking, loop detection, and adaptive decision-making on behalf of Controller's end users |
| **Conversation Processing** | Receiving, routing, and processing natural language inputs and outputs between end users and AI models, including prompt construction and response delivery |
| **Memory Management** | Storing and retrieving contextual information across working memory, short-term memory, long-term memory, and episodic memory to maintain conversation continuity and improve agent performance |
| **LLM API Routing** | Transmitting prompts to and receiving responses from large language model providers as configured by the Controller, including OpenAI, Google Vertex AI (Gemini), and Anthropic |
| **Tool Execution** | Executing Controller-configured tools and plugins (such as code interpreters and browser automation) that may process Personal Data as part of agent task completion |
| **Weight and Learning Adaptation** | Adjusting brain-inspired synaptic weights and plasticity parameters based on interaction outcomes, where enabled by the Controller, to improve agent performance over time |
| **Audit Logging** | Recording processing activities, access events, and security-relevant operations for compliance, debugging, and accountability purposes |

### 3.2 Purpose of Processing
The Processor processes Personal Data solely for the purpose of providing the Service to the Controller as described in the Principal Agreement. The Processor shall not process Personal Data for any other purpose, including but not limited to the Processor's own commercial purposes, marketing, profiling for the Processor's benefit, or sale to third parties.

---

## 4. Types of Personal Data

The following categories of Personal Data may be processed under this DPA, as determined by the Controller's configuration and use of the Service:

| Category | Examples | Sensitivity |
|---|---|---|
| **Conversation Content** | Natural language inputs and outputs between end users and AI agents, including questions, instructions, and responses | May include any category of data depending on end-user input |
| **User Identifiers** | User IDs, session IDs, tenant IDs, and other identifiers used to associate processing with specific Data Subjects | Standard personal data |
| **Behavioral Patterns** | Interaction preferences, usage patterns, agent performance metrics per user, and learned adaptation weights (where per-user learning is enabled by the Controller) | May constitute profiling data under Article 22 GDPR |
| **Tool Execution Data** | Data processed through Controller-configured tools, which may include file contents, web search queries, code snippets, or other data types depending on the tools enabled | Varies by tool configuration |
| **Memory Data** | Working memory, short-term memory, long-term memory entries, and episodic memory records associated with specific users or sessions | May include any category of data depending on conversation content |
| **Technical Metadata** | Timestamps, API call logs, model selection data, processing durations, error logs, and audit trail entries | Standard personal data |

### 4.1 Special Category Data
The Controller acknowledges that end users may include special category data (as defined in Article 9 of the GDPR) in conversation inputs. The Service includes configurable PII detection capabilities (DataClassifier) that the Controller may enable to detect and redact such data. The Controller is responsible for determining whether the processing of special category data is lawful and for configuring appropriate technical safeguards within the Service.

---

## 5. Categories of Data Subjects

The Data Subjects whose Personal Data is processed under this DPA are:

| Category | Description |
|---|---|
| **End Users** | Natural persons who interact with the Controller's applications that are powered by the corteX AI Agent SDK |
| **Controller Personnel** | Employees, contractors, or agents of the Controller who configure, administer, or test the Service |

The Controller is responsible for ensuring that it has a lawful basis for the processing of Personal Data of these Data Subjects and for providing appropriate privacy notices to them.

---

## 6. Controller Instructions

### 6.1 Documented Instructions
The Processor shall process Personal Data only on documented instructions from the Controller, unless required to do so by European Union or Member State law to which the Processor is subject, in which case the Processor shall inform the Controller of that legal requirement before processing (unless that law prohibits such information on important grounds of public interest).

### 6.2 Scope of Instructions
The Controller's instructions to the Processor are as follows:

(a) Process Personal Data in accordance with this DPA and the Principal Agreement;

(b) Process Personal Data as necessary to provide the Service, including the processing activities described in Section 3;

(c) Process Personal Data in accordance with the Controller's configuration of the Service, including tenant-level settings for data retention, memory management, learning/adaptation features, tool enablement, and LLM provider selection;

(d) Process Personal Data in accordance with any additional written instructions provided by the Controller, provided such instructions are consistent with the terms of this DPA and the Principal Agreement.

### 6.3 Obligation to Inform
If the Processor considers that an instruction from the Controller infringes the GDPR or other Applicable Data Protection Law, the Processor shall immediately inform the Controller. The Processor shall not be required to assess the lawfulness of the Controller's instructions beyond a reasonable understanding of data protection requirements, but shall not knowingly process Personal Data in a manner that it believes to be unlawful.

### 6.4 Additional Instructions
Any additional instructions beyond the scope of this DPA and the Principal Agreement require a prior written agreement between the Parties, including agreement on any additional fees that may apply.

---

## 7. Confidentiality of Personnel

### 7.1 Confidentiality Obligations
The Processor shall ensure that all persons authorised to process Personal Data under this DPA have committed themselves to confidentiality or are under an appropriate statutory obligation of confidentiality.

### 7.2 Access Limitation
The Processor shall ensure that access to Personal Data is limited to those personnel who require such access for the performance of the Service and that such personnel process Personal Data only in accordance with the Controller's instructions.

### 7.3 Training
The Processor shall ensure that all personnel with access to Personal Data receive appropriate training on data protection obligations, including the requirements of this DPA, at least annually.

---

## 8. Security Measures

### 8.1 Appropriate Technical and Organisational Measures
Pursuant to Article 32 of the GDPR, the Processor shall implement and maintain appropriate technical and organisational measures to ensure a level of security appropriate to the risk, taking into account the state of the art, the costs of implementation, and the nature, scope, context, and purposes of the processing, as well as the risk of varying likelihood and severity for the rights and freedoms of natural persons.

### 8.2 Minimum Security Measures
Without prejudice to the generality of Section 8.1, the Processor shall implement at a minimum the security measures described in Annex B (Technical and Organisational Measures) to this DPA, which include:

| Control Domain | Measures |
|---|---|
| **Encryption at Rest** | AES-256 encryption for all Personal Data stored by the Service, including conversation data, memory stores, and configuration data. Tenant-derived encryption keys via Fernet (AES-128-CBC with HMAC-SHA256) for API key storage in KeyVault. |
| **Encryption in Transit** | TLS 1.3 for all data transmitted between the Controller's application and the Service, between the Service and LLM providers, and between internal service components. |
| **Access Controls** | Role-based access control (RBAC) with least-privilege principles. Multi-factor authentication (MFA) required for all Processor personnel accessing production systems. Quarterly access reviews with management sign-off. |
| **Tenant Isolation** | Architectural tenant isolation at the data layer. Every data access requires a tenant identifier. Cross-tenant data access is prevented by design. Separate KeyVault instances per tenant for cryptographic key material. |
| **Audit Logging** | Comprehensive logging of all data access events, configuration changes, authentication events, and processing activities. Logs retained for a minimum of 90 days. Tamper-evident logging with integrity verification. |
| **PII Detection** | DataClassifier pipeline capable of detecting personal data, special category data, and sensitive content in processing streams, with configurable redaction before LLM transmission. |
| **Input Validation** | SafetyPolicy enforcement including prompt injection detection, output sanitisation, and configurable safety thresholds. |
| **Vulnerability Management** | Automated dependency scanning (Dependabot), regular vulnerability assessments, and defined remediation timelines (critical: 24 hours; high: 7 days; medium: 30 days; low: 90 days). |
| **Incident Response** | Documented incident response plan with defined severity levels (P0-P3), escalation procedures, and post-incident review processes. |

### 8.3 Ongoing Security Assessment
The Processor shall regularly test, assess, and evaluate the effectiveness of the technical and organisational measures for ensuring the security of processing, including through vulnerability scanning, penetration testing, and internal security reviews.

### 8.4 Security Updates
The Processor may update the security measures from time to time, provided that such updates do not materially decrease the overall level of security of the Service.

---

## 9. Sub-Processor Management

### 9.1 General Authorisation
The Controller provides general authorisation for the Processor to engage Sub-Processors for the performance of the Service, subject to the conditions set out in this Section 9.

### 9.2 Current Sub-Processors
The Processor's current Sub-Processors are listed in Annex C (Sub-Processor List) to this DPA and are published at:

**Sub-Processor List URL:** https://docs.cortex-ai.com/compliance/gdpr/sub-processors

The Controller acknowledges that not all listed Sub-Processors will necessarily process Personal Data for the Controller; the specific Sub-Processors engaged depend on the Controller's configuration of the Service (e.g., which LLM providers the Controller enables for its tenant).

### 9.3 Advance Notice of Changes
The Processor shall notify the Controller in writing at least **thirty (30) calendar days** before engaging any new Sub-Processor or making a material change to an existing Sub-Processor engagement. Such notice shall include the identity of the proposed Sub-Processor, the processing to be performed, and the location of the processing.

### 9.4 Objection Right
The Controller may object to a new Sub-Processor by notifying the Processor in writing within fifteen (15) calendar days of receiving the notice under Section 9.3. The Controller's objection must include reasonable grounds related to data protection. Upon receipt of an objection, the Processor shall:

(a) Make reasonable efforts to provide the Controller with a change to the Service that avoids the use of the objected-to Sub-Processor, at no additional cost; or

(b) If such change is not reasonably possible, the Controller may terminate the Principal Agreement and this DPA without penalty, with a pro-rata refund of any prepaid fees for the period following termination.

### 9.5 Sub-Processor Obligations
The Processor shall, by way of a written contract, impose on each Sub-Processor data protection obligations no less protective than those set out in this DPA. The Processor shall remain fully liable to the Controller for the performance of each Sub-Processor's obligations.

### 9.6 Controller Configuration
The Processor provides the Controller with the ability to configure which LLM providers (Sub-Processors) are permitted for the Controller's tenant through the Service's ModelPolicy and ToolPolicy features. The Processor shall not route the Controller's data to any LLM provider that the Controller has not enabled.

---

## 10. Assistance with Data Subject Rights

### 10.1 Data Subject Requests
Taking into account the nature of the processing, the Processor shall assist the Controller by appropriate technical and organisational measures, insofar as this is possible, for the fulfilment of the Controller's obligation to respond to requests for exercising the Data Subject's rights under Chapter III of the GDPR.

### 10.2 DSAR API
The Processor provides the following programmatic interfaces for the Controller to fulfil Data Subject requests:

| Right | GDPR Article | API Capability |
|---|---|---|
| **Right of Access** | Art. 15 | Export all Personal Data associated with a user in structured JSON format via the DSAR API |
| **Right to Rectification** | Art. 16 | Correct or update Personal Data in memory stores, preferences, and conversation records via the DSAR API |
| **Right to Erasure** | Art. 17 | Delete all Personal Data associated with a user from all storage layers (conversations, working memory, short-term memory, long-term memory, episodic memory, preferences, per-user weight deltas) via the DSAR API |
| **Right to Restriction** | Art. 18 | Temporarily halt processing for a specific user while retaining data via the DSAR API |
| **Right to Data Portability** | Art. 20 | Export Personal Data in a structured, commonly used, and machine-readable format (JSON) via the DSAR API |
| **Right to Object** | Art. 21 | Register an objection to specific processing activities (e.g., profiling, learning/adaptation) for a specific user via the DSAR API |
| **Automated Decision-Making** | Art. 22 | Provide explanations of agent decisions through the goal-tracking and decision-logging capabilities of the Service |

### 10.3 Response Timelines
The Processor shall respond to Controller requests for DSAR assistance within **five (5) business days** of receiving the request, enabling the Controller to meet the one-month statutory deadline under the GDPR.

### 10.4 Notification of Direct Requests
If the Processor receives a request directly from a Data Subject, the Processor shall promptly notify the Controller and shall not respond to the Data Subject directly unless instructed to do so by the Controller or required by Applicable Data Protection Law.

### 10.5 Costs
The Processor shall not charge additional fees for reasonable DSAR assistance. If a request requires extraordinary effort (e.g., manual recovery of archived data), the Parties shall agree on reasonable costs before the Processor proceeds.

---

## 11. Data Protection Impact Assessments

### 11.1 DPIA Assistance
Taking into account the nature of the processing and the information available to the Processor, the Processor shall assist the Controller in ensuring compliance with the Controller's obligations under Articles 35 and 36 of the GDPR (Data Protection Impact Assessments and Prior Consultation).

### 11.2 Information Provision
The Processor shall provide the Controller with:

(a) Documentation of the Service's data processing activities, data flows, and technical architecture sufficient for the Controller to conduct a DPIA;

(b) A DPIA template tailored to the Service's features, including the brain-inspired learning system, memory management, and LLM API routing;

(c) Information about the Processor's security measures, Sub-Processors, and international data transfers;

(d) Reasonable cooperation in responding to queries from the Controller or a Supervisory Authority in connection with a DPIA.

### 11.3 Profiling Disclosure
The Processor informs the Controller that the Service's brain-inspired learning system (including synaptic weights, plasticity, and prediction/surprise mechanisms) may constitute profiling under Article 4(4) of the GDPR when per-user learning is enabled. The Controller is responsible for determining whether such processing requires a DPIA and for configuring the Service accordingly (including disabling per-user learning if required).

---

## 12. Personal Data Breach Notification

### 12.1 Notification to Controller
The Processor shall notify the Controller without undue delay, and in any event within **twenty-four (24) hours** of becoming aware of a Personal Data Breach affecting the Controller's Personal Data.

### 12.2 Content of Notification
The notification shall include, to the extent available at the time of notification:

(a) A description of the nature of the Personal Data Breach, including where possible the categories and approximate number of Data Subjects concerned and the categories and approximate number of Personal Data records concerned;

(b) The name and contact details of the Processor's point of contact for further information;

(c) A description of the likely consequences of the Personal Data Breach;

(d) A description of the measures taken or proposed to be taken by the Processor to address the Personal Data Breach, including, where appropriate, measures to mitigate its possible adverse effects.

### 12.3 Supplemental Information
If it is not possible to provide all information at the time of the initial notification, the Processor shall provide information in phases as it becomes available, without further undue delay.

### 12.4 Assistance
The Processor shall cooperate with the Controller and take reasonable commercial steps to assist in the investigation, mitigation, and remediation of the Personal Data Breach.

### 12.5 No Assessment of Risk
The notification obligation under this Section 12 applies regardless of whether the Personal Data Breach is likely to result in a risk to the rights and freedoms of Data Subjects. The Controller remains responsible for assessing the risk level and determining whether notification to the Supervisory Authority and/or Data Subjects is required under Articles 33 and 34 of the GDPR.

### 12.6 Record Keeping
The Processor shall maintain a record of all Personal Data Breaches, including the facts relating to the breach, its effects, and the remedial action taken, and shall make such records available to the Controller upon request.

---

## 13. Data Return and Deletion

### 13.1 Upon Termination
Upon termination or expiration of the Principal Agreement, the Processor shall, at the Controller's written election:

(a) **Return** all Personal Data to the Controller in a structured, commonly used, and machine-readable format (JSON); or

(b) **Delete** all Personal Data from all systems, storage layers, and backups, including conversations, memory stores, weight deltas, preferences, and audit logs.

### 13.2 Timeline
The Processor shall complete the return or deletion within **thirty (30) calendar days** of receiving the Controller's written instruction. If no instruction is received within thirty (30) calendar days of termination, the Processor shall delete all Personal Data.

### 13.3 Certification
Upon completion of deletion, the Processor shall provide the Controller with a written certification confirming that all Personal Data has been deleted from all systems, including backup systems, and that such data is no longer recoverable.

### 13.4 Exceptions
The Processor may retain Personal Data to the extent required by European Union or Member State law, provided that the Processor:

(a) Informs the Controller of such retention and the legal basis for it;

(b) Processes such retained data only for the purpose required by law;

(c) Implements appropriate technical and organisational measures to protect such data; and

(d) Deletes such data as soon as the legal retention obligation expires.

### 13.5 Backup Deletion
Personal Data in backup systems shall be deleted in accordance with the Processor's standard backup rotation schedule, which shall not exceed ninety (90) calendar days from the date of the deletion instruction.

---

## 14. Audit Rights

### 14.1 Audit Reports
The Processor shall make available to the Controller, upon request and no more than once per twelve (12) month period, either:

(a) A copy of the Processor's most recent SOC 2 Type II report (or, if not yet available, a SOC 2 Type I report) covering the Service, provided under a confidentiality obligation; or

(b) A summary of the results of the Processor's most recent independent security audit or penetration test, redacted as necessary to protect confidential information of other customers.

### 14.2 On-Site Audit Right
If the audit reports provided under Section 14.1 are insufficient, or if the Controller is required to conduct an audit by a Supervisory Authority, the Controller (or an independent third-party auditor appointed by the Controller) may conduct an audit of the Processor's processing activities related to this DPA, subject to the following conditions:

(a) The Controller shall provide at least **thirty (30) calendar days** prior written notice of the audit;

(b) The audit shall be conducted during normal business hours and shall not unreasonably disrupt the Processor's operations;

(c) The auditor shall be bound by appropriate confidentiality obligations;

(d) The audit scope shall be limited to the Processor's processing of the Controller's Personal Data under this DPA;

(e) The Controller shall bear the costs of the audit, unless the audit reveals a material breach of this DPA by the Processor, in which case the Processor shall bear the reasonable costs.

### 14.3 Audit Frequency
The Controller may exercise its audit right under Section 14.2 no more than once per twelve (12) month period, unless:

(a) Required by a Supervisory Authority;

(b) A Personal Data Breach has occurred affecting the Controller's Personal Data; or

(c) The Controller has reasonable grounds to believe the Processor is not complying with this DPA.

### 14.4 Contribution to Audits
The Processor shall contribute to and cooperate with audits conducted under this Section 14, including by providing access to relevant facilities, systems, personnel, and records.

---

## 15. International Data Transfers

### 15.1 General Principle
The Processor shall not transfer Personal Data to a country outside the EEA unless:

(a) The European Commission has issued an adequacy decision for the recipient country (Article 45 GDPR);

(b) Appropriate safeguards have been implemented in accordance with Article 46 GDPR (including SCCs); or

(c) A derogation under Article 49 GDPR applies.

### 15.2 Transfer Mechanisms
The following transfer mechanisms apply to the Processor's processing activities:

| Destination | Mechanism | Supplementary Measures |
|---|---|---|
| **Israel** (Questo Ltd) | Adequacy Decision (European Commission Decision 2011/61/EU) | N/A |
| **United States** (OpenAI, Google, Anthropic) | EU-US Data Privacy Framework (DPF) certification where applicable, supplemented by SCCs (Module 3: Processor to Sub-Processor) | Transfer Impact Assessment; encryption in transit and at rest; access controls; contractual prohibitions on government access disclosure |
| **EU/EEA** (Google Vertex AI, when EU region selected) | No transfer (data remains in EEA) | N/A |

### 15.3 Standard Contractual Clauses
Where SCCs are required as a transfer mechanism, the Parties agree that the SCCs in Annex D (Standard Contractual Clauses Module) to this DPA are incorporated by reference. The SCCs shall be deemed completed as follows:

(a) **Module Two** (Controller to Processor) applies to transfers of Personal Data from the Controller to the Processor;

(b) **Module Three** (Processor to Sub-Processor) applies to transfers of Personal Data from the Processor to Sub-Processors;

(c) For each module, the optional clauses and annexes shall be completed as set out in Annex D.

### 15.4 EU-Only Option
The Controller may elect to restrict all processing to EEA-located infrastructure and EU-region LLM endpoints by configuring the Service's data residency settings accordingly. Where the Controller makes such an election, the Processor shall not transfer any of the Controller's Personal Data outside the EEA.

### 15.5 Transfer Impact Assessments
The Processor shall maintain a Transfer Impact Assessment for each jurisdiction to which Personal Data is transferred and shall make such assessments available to the Controller upon request.

### 15.6 Changes in Transfer Mechanisms
If a transfer mechanism relied upon under this Section 15 is invalidated by a court of competent jurisdiction or a Supervisory Authority, the Processor shall promptly implement an alternative valid transfer mechanism. If no valid mechanism is available, the Processor shall, at the Controller's election, either:

(a) Migrate the affected processing to a jurisdiction for which a valid transfer mechanism exists; or

(b) Cease the affected processing and return or delete the relevant Personal Data.

---

## 16. Liability

### 16.1 Liability Cap
Each Party's aggregate liability arising out of or in connection with this DPA (whether in contract, tort, or otherwise) shall not exceed the greater of:

(a) The total fees paid or payable by the Controller to the Processor under the Principal Agreement during the twelve (12) months immediately preceding the event giving rise to the liability; or

(b) [Agreed Minimum Liability Cap] EUR.

### 16.2 Exclusions
The liability cap in Section 16.1 shall not apply to:

(a) Liability arising from the Processor's breach of Section 6 (Controller Instructions), to the extent the Processor processes Personal Data outside the Controller's documented instructions;

(b) Liability arising from the Processor's breach of Section 15 (International Data Transfers);

(c) Either Party's indemnification obligations under this DPA;

(d) Liability that cannot be limited under Applicable Data Protection Law.

### 16.3 Mutual Indemnification
(a) The Processor shall indemnify and hold harmless the Controller against all claims, damages, losses, costs, and expenses (including reasonable legal fees) arising from the Processor's breach of this DPA or Applicable Data Protection Law, provided the Controller promptly notifies the Processor of such claim and provides reasonable cooperation.

(b) The Controller shall indemnify and hold harmless the Processor against all claims, damages, losses, costs, and expenses (including reasonable legal fees) arising from the Controller's breach of this DPA or Applicable Data Protection Law, including claims arising from the Controller's instructions to the Processor that result in a violation of Applicable Data Protection Law, provided the Processor promptly notifies the Controller of such claim and provides reasonable cooperation.

### 16.4 Mitigation
Each Party shall take reasonable steps to mitigate any losses arising from a breach of this DPA.

---

## 17. General Provisions

### 17.1 Entire Agreement
This DPA, together with its Annexes and the Principal Agreement, constitutes the entire agreement between the Parties with respect to the processing of Personal Data in connection with the Service. This DPA supersedes all prior agreements, representations, and understandings with respect to such processing.

### 17.2 Amendments
This DPA may only be amended by a written instrument signed by both Parties. The Processor may update Annex B (Technical and Organisational Measures) and Annex C (Sub-Processor List) in accordance with the procedures set out in this DPA without requiring a formal amendment.

### 17.3 Severability
If any provision of this DPA is held to be invalid or unenforceable, the remaining provisions shall continue in full force and effect. The invalid or unenforceable provision shall be replaced by a valid provision that achieves the closest possible economic effect.

### 17.4 No Third-Party Beneficiaries
This DPA does not confer any rights on any person other than the Parties and, where applicable, their respective successors and permitted assigns, except that Data Subjects may enforce their rights under the SCCs where applicable.

### 17.5 Survival
The obligations under Sections 7, 12, 13, 14, and 16 of this DPA shall survive the termination or expiration of this DPA.

---

## Signatures

**For the Controller:**

| | |
|---|---|
| Name | [Authorised Signatory Name] |
| Title | [Authorised Signatory Title] |
| Date | [Date] |
| Signature | _________________________ |

**For the Processor (Questo Ltd):**

| | |
|---|---|
| Name | [Authorised Signatory Name] |
| Title | [Authorised Signatory Title] |
| Date | [Date] |
| Signature | _________________________ |

---

## Annex A: Processing Details

### A.1 Description of Processing

| Element | Details |
|---|---|
| **Subject Matter of Processing** | Provision of the corteX AI Agent SDK platform for AI agent orchestration |
| **Duration of Processing** | For the term of the Principal Agreement and until all Personal Data is returned or deleted |
| **Nature of Processing** | AI agent orchestration, conversation processing, memory management, LLM API routing, tool execution, weight adaptation, audit logging |
| **Purpose of Processing** | To provide the Controller with AI agent capabilities for integration into the Controller's applications |
| **Categories of Data Subjects** | End users of the Controller's applications; Controller's personnel |
| **Types of Personal Data** | Conversation content, user identifiers, behavioral patterns, tool execution data, memory data, technical metadata |
| **Special Category Data** | May be present in conversation content if provided by Data Subjects; subject to Controller's DataClassifier configuration |

### A.2 Processing Locations

| Location | Purpose | Entity |
|---|---|---|
| Israel | Primary platform operation and development | Questo Ltd |
| United States | LLM inference (when US-based providers are enabled by Controller) | OpenAI, Anthropic, Google |
| European Union | LLM inference (when EU regions are selected by Controller) | Google Vertex AI |
| Controller's Infrastructure | On-premise deployment (when selected by Controller) | Controller |

### A.3 Data Flow Summary

```
End User --> Controller's Application --> corteX SDK API
    |
    +--> Conversation Processing
    |       |
    |       +--> PII Detection (DataClassifier) [optional, Controller-configured]
    |       |
    |       +--> Memory Management (Working/Short-term/Long-term/Episodic)
    |       |
    |       +--> LLM API Routing (to Controller-selected provider)
    |       |       |
    |       |       +--> OpenAI API (USA) [if enabled]
    |       |       +--> Google Vertex AI (EU/US) [if enabled]
    |       |       +--> Anthropic API (USA) [if enabled]
    |       |       +--> Local Model (on-prem) [if configured]
    |       |
    |       +--> Tool Execution [if enabled]
    |       |
    |       +--> Weight Adaptation [if per-user learning enabled]
    |
    +--> Audit Log
    |
    +--> Response --> Controller's Application --> End User
```

---

## Annex B: Technical and Organisational Measures

### B.1 Encryption

| Control | Implementation |
|---|---|
| Encryption at rest | AES-256 encryption for all stored Personal Data. Fernet (AES-128-CBC + HMAC-SHA256) for API key material in KeyVault with tenant-derived keys. |
| Encryption in transit | TLS 1.3 enforced for all external communications. Internal service communications encrypted via TLS 1.2 minimum. |
| Key management | Tenant-derived encryption keys. Key rotation procedures documented. No plaintext key storage. Leak detection scanning on all outputs. |

### B.2 Access Control

| Control | Implementation |
|---|---|
| Authentication | Multi-factor authentication (MFA) required for all Processor personnel on all production systems. |
| Authorisation | Role-based access control (RBAC) with least-privilege principle. Separate roles for development, operations, and support. |
| Access reviews | Quarterly access reviews with management sign-off. Deprovisioning within 24 hours of personnel departure. |
| Session management | Automatic session timeout. Re-authentication required for sensitive operations. |

### B.3 Tenant Isolation

| Control | Implementation |
|---|---|
| Data isolation | Architectural tenant isolation at the data layer. Every data access requires a valid tenant identifier. |
| Cross-tenant prevention | Cross-tenant data access is prevented by design. DataClassifier enforces data boundary controls. |
| Key isolation | Separate KeyVault instances per tenant for cryptographic material. |
| Configuration isolation | Per-tenant configuration for retention policies, memory management, learning features, tool enablement, and model selection. |

### B.4 Monitoring and Logging

| Control | Implementation |
|---|---|
| Audit logging | All data access, configuration changes, authentication events, and processing activities are logged. |
| Log retention | Minimum 90-day retention for security logs. |
| Log integrity | Tamper-evident logging with integrity verification. |
| Alerting | Automated alerting for security-relevant events including failed authentication, access anomalies, and configuration changes. |

### B.5 Vulnerability Management

| Control | Implementation |
|---|---|
| Dependency scanning | Automated scanning via Dependabot with alerts for known vulnerabilities. |
| Remediation timelines | Critical: 24 hours. High: 7 days. Medium: 30 days. Low: 90 days. |
| Penetration testing | Annual external penetration testing by independent security firm. |
| Code review | All code changes require peer review via pull request before merge. |

### B.6 Incident Response

| Control | Implementation |
|---|---|
| Incident response plan | Documented four-phase plan: Detection and Assessment, Containment and Eradication, Recovery, Post-Incident Review. |
| Severity classification | P0 (critical) through P3 (low) with defined escalation timelines and response procedures. |
| Breach notification | Controller notification within 24 hours of awareness, per Section 12 of the DPA. |
| Testing | Annual tabletop incident response exercises. |

### B.7 Data Minimisation and Retention

| Control | Implementation |
|---|---|
| Data minimisation | Configurable context windows. PII stripping before LLM calls (when enabled). Purpose-bound processing. |
| Retention controls | Per-tenant configurable retention: NONE, SESSION, PERSISTENT, or AUDIT_ONLY. TTL enforcement on memory stores. |
| Deletion | Layered deletion across all storage types: conversations, working memory, short-term memory, long-term memory, episodic memory, preferences, user weight deltas. |
| Automated enforcement | Automated retention period enforcement with scheduled cleanup processes. |

### B.8 Business Continuity

| Control | Implementation |
|---|---|
| Backup procedures | Regular automated backups of all data stores. |
| Backup testing | Periodic backup restoration testing to verify data recoverability. |
| Disaster recovery | Documented disaster recovery plan with defined RTO and RPO. |
| Redundancy | Service designed for high availability with failover capabilities. |

---

## Annex C: Sub-Processor List

*Effective Date: 2026-02-16*

The following Sub-Processors are authorised to process Personal Data on behalf of the Controller, subject to the Controller's configuration of the Service:

| Sub-Processor | Legal Entity | Purpose | Data Processed | Location | Transfer Mechanism | DPA Status |
|---|---|---|---|---|---|---|
| **OpenAI** | OpenAI, Inc. | LLM inference for AI agent responses (when enabled by Controller) | Prompts and responses containing conversation content | United States | EU-US Data Privacy Framework (DPF) + Standard Contractual Clauses (SCCs) | Active DPA in place |
| **Google (Vertex AI)** | Google LLC | LLM inference for AI agent responses (Gemini models, when enabled by Controller) | Prompts and responses containing conversation content | EU or US (Controller-configurable via region selection) | Adequacy Decision (EU regions) / DPF + SCCs (US regions) | Google Cloud Data Processing Addendum (CDPA) |
| **Anthropic** | Anthropic, PBC | LLM inference for AI agent responses (Claude models, when enabled by Controller) | Prompts and responses containing conversation content | United States | Standard Contractual Clauses (SCCs) + Transfer Impact Assessment (TIA) | Active DPA in place |

**Notes:**
- The Controller determines which LLM providers are active for its tenant through the Service's ModelPolicy configuration. Personal Data is only transmitted to providers that the Controller has explicitly enabled.
- Cloud infrastructure providers (e.g., AWS, Azure, GCP) used by the above Sub-Processors are covered under each Sub-Processor's own data processing agreements and are classified as sub-sub-processors.
- For on-premise deployments where the Controller hosts local models, no Sub-Processor engagement occurs for LLM inference.

The current Sub-Processor list is maintained at: https://docs.cortex-ai.com/compliance/gdpr/sub-processors

---

## Annex D: Standard Contractual Clauses Module

### D.1 Applicability
The Standard Contractual Clauses (SCCs) adopted by the European Commission in Implementing Decision (EU) 2021/914 are incorporated by reference into this DPA and apply to transfers of Personal Data from the EEA to countries that have not received an adequacy decision from the European Commission (except where the EU-US Data Privacy Framework applies as the primary transfer mechanism).

### D.2 Module Selection

| Module | Applicability |
|---|---|
| **Module Two** (Controller to Processor) | Applies where the Controller transfers Personal Data from the EEA to the Processor (Questo Ltd) in Israel. Note: Israel has an adequacy decision; SCCs serve as a supplementary safeguard. |
| **Module Three** (Processor to Sub-Processor) | Applies where the Processor transfers Personal Data to Sub-Processors located outside the EEA (OpenAI, Anthropic, Google when US regions are used). |

### D.3 Completion of SCCs

The following selections apply to the SCCs:

**Clause 7 (Docking Clause):** Included. Third parties may accede to these SCCs with the consent of the existing Parties.

**Clause 9 (Sub-Processors):**
- **Option 2** (General written authorisation) is selected.
- The Processor shall inform the Controller of any intended changes to the list of Sub-Processors, giving the Controller the opportunity to object in accordance with Section 9 of this DPA.

**Clause 11 (Redress):**
- The optional language permitting Data Subjects to lodge a complaint with an independent dispute resolution body is not included.

**Clause 13 (Supervision):**
- The Supervisory Authority of the EU Member State in which the Controller is established shall act as the competent supervisory authority. Where the Controller is not established in the EU, the Supervisory Authority of the Member State where the Controller's EU representative is established shall be competent.

**Clause 17 (Governing Law):**
- The SCCs shall be governed by the law of [Member State of Controller's establishment or primary EU operations].

**Clause 18 (Choice of Forum and Jurisdiction):**
- Disputes shall be resolved before the courts of [Member State selected under Clause 17].

### D.4 Supplementary Measures
In addition to the safeguards provided by the SCCs, the following supplementary measures apply to all transfers:

(a) **Technical Measures:** All Personal Data is encrypted in transit (TLS 1.3) and at rest (AES-256). Access to Personal Data is restricted to authorised personnel with a legitimate need.

(b) **Organisational Measures:** The Processor maintains strict access controls, conducts regular security training, and requires confidentiality obligations for all personnel.

(c) **Contractual Measures:** Each Sub-Processor is bound by data processing agreements with obligations no less protective than those in this DPA. Sub-Processors are contractually prohibited from disclosing Personal Data to government authorities except as required by law, and must notify the Processor of any government access request (where legally permitted).

(d) **Transfer Impact Assessment:** The Processor maintains a documented Transfer Impact Assessment for each country to which Personal Data is transferred, evaluating the legal framework and practical application of government surveillance powers in the recipient country.

---

*End of Data Processing Agreement*
