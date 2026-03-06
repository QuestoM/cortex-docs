# Privacy Policy

**Questo Ltd -- corteX AI Agent SDK**

---

| Document Control |  |
|---|---|
| **Document ID** | GDPR-PP-001 |
| **Version** | 1.0 |
| **Effective Date** | 2026-02-16 |
| **Classification** | Public |
| **Owner** | Questo Ltd, Legal & Compliance |
| **Last Updated** | 2026-02-16 |
| **Publication URL** | https://docs.cortex-ai.com/privacy |

---

## 1. Introduction

Questo Ltd ("Questo", "we", "us", or "our") is an Israeli technology company that develops and operates the corteX AI Agent SDK, an enterprise-grade platform for building reliable AI agent capabilities into software applications.

This Privacy Policy explains how Questo processes personal data in connection with the corteX platform. It applies to:

- **Our customers** ("Controllers") -- the businesses and developers who integrate corteX into their applications;
- **End users** -- the natural persons who interact with applications powered by corteX; and
- **Visitors** to our website and documentation.

Questo is committed to protecting personal data in accordance with the General Data Protection Regulation (EU) 2016/679 ("GDPR"), applicable national data protection laws, and industry best practices.

---

## 2. Who We Are

| | |
|---|---|
| **Legal Entity** | Questo Ltd |
| **Role under GDPR** | Data Processor (for customer data processed through the corteX platform); Data Controller (for our own business data, website analytics, and customer account data) |
| **Registered Address** | Israel |
| **Privacy Contact** | privacy@questo.io |
| **DPA Inquiries** | dpa@questo.io |

Israel has been recognised by the European Commission as providing an adequate level of data protection (Commission Decision 2011/61/EU), meaning transfers of personal data from the EEA to Questo in Israel are lawful without additional safeguards.

---

## 3. Our Role in Data Processing

### 3.1 The B2B2C Data Chain

corteX operates within a layered data processing chain:

| Entity | GDPR Role | Responsibility |
|---|---|---|
| **End User** | Data Subject | The individual whose personal data is processed |
| **Customer** (your business) | Data Controller (or Processor for their own customers) | Determines the purposes and means of processing; provides privacy notices to end users; handles data subject requests |
| **Questo Ltd** (corteX) | Data Processor | Processes personal data only on the customer's instructions; provides technical infrastructure for data protection |
| **LLM Providers** (OpenAI, Google, Anthropic) | Sub-Processors | Process data on behalf of Questo, as configured by the customer |

### 3.2 What This Means for You

- **If you are a customer:** We process your end users' data strictly according to your instructions. Your Data Processing Agreement (DPA) with us governs this processing. You remain responsible for ensuring you have a lawful basis and for providing privacy notices to your end users.

- **If you are an end user:** The application you are using is operated by our customer, not by Questo directly. Our customer is responsible for informing you about how your data is processed. If you have questions about your data, please contact the operator of the application you are using. If you believe your data is being processed incorrectly, you may also contact us at privacy@questo.io and we will direct your inquiry to the appropriate customer.

- **If you are a website visitor:** We process limited data about your visit to our website and documentation, as described in Section 8 of this policy.

---

## 4. What Personal Data We Process and Why

### 4.1 Customer Platform Data (Questo as Processor)

When our customers use the corteX platform, the following personal data may be processed:

| Data Category | Examples | Purpose | Legal Basis | Retention |
|---|---|---|---|---|
| **Conversation Content** | Natural language inputs and outputs between end users and AI agents | Providing AI agent orchestration services as instructed by the customer | Contract (Art. 6(1)(b)) -- necessary for the performance of our contract with the customer | As configured by the customer: SESSION (deleted at end of session), PERSISTENT (retained until deleted), or NONE (not stored) |
| **User Identifiers** | User IDs, session IDs, tenant IDs | Associating processing with specific users for the customer's service delivery | Contract (Art. 6(1)(b)) | For the duration of the customer relationship, or as configured by the customer |
| **Behavioral Patterns** | Interaction preferences, usage patterns, learned adaptation weights | Improving AI agent performance for the specific user, where per-user learning is enabled by the customer | Contract (Art. 6(1)(b)); Legitimate Interest (Art. 6(1)(f)) for service improvement | As configured by the customer; deletable via DSAR API |
| **Memory Data** | Working memory, short-term memory, long-term memory, episodic memory records | Maintaining conversation context and continuity for the customer's service | Contract (Art. 6(1)(b)) | Per customer's retention configuration; TTL-enforced |
| **Tool Execution Data** | Data processed through customer-configured tools (e.g., code interpreter outputs, web search results) | Executing tools as part of AI agent task completion, as instructed by the customer | Contract (Art. 6(1)(b)) | SESSION or as configured by the customer |
| **Technical Metadata** | Timestamps, API call logs, model selection data, processing durations, error logs | Service operation, debugging, and audit compliance | Contract (Art. 6(1)(b)); Legitimate Interest (Art. 6(1)(f)) for service reliability | 90 days for operational logs; longer for audit logs as required |

**We do not use customer platform data for our own purposes.** We do not train models on customer data. We do not sell customer data. We do not use customer data for marketing or advertising.

### 4.2 Customer Account Data (Questo as Controller)

| Data Category | Examples | Purpose | Legal Basis | Retention |
|---|---|---|---|---|
| **Account Information** | Company name, contact person name, email address, billing information | Managing the customer relationship, providing support, invoicing | Contract (Art. 6(1)(b)) | Duration of the customer relationship plus 7 years for accounting records |
| **Support Communications** | Support tickets, emails, meeting notes | Providing customer support and resolving technical issues | Contract (Art. 6(1)(b)); Legitimate Interest (Art. 6(1)(f)) | 3 years after resolution |
| **Usage Analytics** | Aggregate platform usage metrics (API call volumes, error rates, feature usage) | Service improvement, capacity planning, and product development | Legitimate Interest (Art. 6(1)(f)) | Aggregated and anonymised; retained indefinitely |

### 4.3 Website Visitor Data (Questo as Controller)

| Data Category | Examples | Purpose | Legal Basis | Retention |
|---|---|---|---|---|
| **Technical Data** | IP address (anonymised), browser type, operating system, referral source | Website operation and security | Legitimate Interest (Art. 6(1)(f)) | 90 days |
| **Documentation Usage** | Pages viewed, search queries within documentation | Improving documentation quality and developer experience | Legitimate Interest (Art. 6(1)(f)) | Aggregated and anonymised |

---

## 5. Legal Bases for Processing

We rely on the following legal bases under Article 6 of the GDPR:

### 5.1 Contract Performance (Article 6(1)(b))
We process personal data where it is necessary for the performance of our contract with the customer (the Principal Agreement and DPA). This is the primary legal basis for all processing of customer platform data.

### 5.2 Legitimate Interest (Article 6(1)(f))
We process personal data where we have a legitimate interest that is not overridden by the data subject's rights and freedoms. Our legitimate interests include:

- Ensuring the security, integrity, and availability of our platform;
- Detecting and preventing fraud, abuse, and security incidents;
- Improving our platform based on aggregate, anonymised usage patterns;
- Maintaining audit logs for compliance and accountability purposes.

We conduct a balancing test for each legitimate interest to ensure that the processing is proportionate and that appropriate safeguards are in place.

### 5.3 Legal Obligation (Article 6(1)(c))
We process personal data where necessary to comply with legal obligations, such as financial record-keeping requirements, tax obligations, and responses to lawful requests from public authorities.

### 5.4 Consent (Article 6(1)(a))
We rely on consent only in limited circumstances, such as optional marketing communications. Where consent is the legal basis, it may be withdrawn at any time by contacting privacy@questo.io.

---

## 6. Sub-Processors

We engage the following sub-processors to provide specific processing activities as part of the corteX platform. Sub-processors are engaged only when the customer enables the corresponding provider for its tenant.

| Sub-Processor | Purpose | Location | Transfer Mechanism |
|---|---|---|---|
| **OpenAI, Inc.** | LLM inference (when customer selects OpenAI models) | United States | EU-US Data Privacy Framework (DPF) + Standard Contractual Clauses (SCCs) |
| **Google LLC (Vertex AI)** | LLM inference (when customer selects Gemini models) | EU or US (customer-configurable) | Adequacy Decision (EU regions) / DPF + SCCs (US regions) |
| **Anthropic, PBC** | LLM inference (when customer selects Claude models) | United States | Standard Contractual Clauses (SCCs) + Transfer Impact Assessment (TIA) |

The complete, up-to-date sub-processor list is available at: https://docs.cortex-ai.com/compliance/gdpr/sub-processors

Customers receive 30 days' advance notice before any sub-processor changes and have the right to object. See the sub-processor list for full details on the notification and objection process.

**On-Premise Option:** For customers who require that no data leave their infrastructure, corteX supports fully on-premise deployment with local language models. In this configuration, no sub-processors are engaged for LLM inference.

---

## 7. International Data Transfers

### 7.1 Where Data May Be Transferred

| Destination | Reason | Transfer Mechanism |
|---|---|---|
| **Israel** (Questo Ltd) | Primary platform operations | Adequacy Decision (Commission Decision 2011/61/EU) |
| **United States** (OpenAI, Anthropic, Google) | LLM inference when US-based providers are enabled | EU-US Data Privacy Framework (DPF) where the recipient is DPF-certified, supplemented by Standard Contractual Clauses (SCCs) |
| **European Union** (Google Vertex AI) | LLM inference when EU region is selected | No transfer required (data remains in EEA) |
| **Customer's infrastructure** | On-premise deployments | No transfer by Questo (data remains under customer's control) |

### 7.2 Safeguards for International Transfers

For all transfers to countries without an adequacy decision, we implement:

- **Standard Contractual Clauses (SCCs)** as approved by the European Commission in Implementing Decision (EU) 2021/914;
- **Transfer Impact Assessments (TIAs)** evaluating the legal framework of the recipient country;
- **Technical supplementary measures** including encryption in transit (TLS 1.3) and at rest (AES-256), access controls, and contractual prohibitions on government access disclosure;
- **EU-US Data Privacy Framework** certification verification for US-based sub-processors where applicable.

### 7.3 EU-Only Processing Option

Customers may configure corteX to restrict all processing to EEA-located infrastructure and EU-region LLM endpoints (e.g., Google Vertex AI Europe regions). When this option is selected, no personal data is transferred outside the EEA.

### 7.4 Data Privacy Framework Considerations

As of February 2026, the EU-US Data Privacy Framework remains in effect and has survived its initial legal challenge (September 2025). However, given the evolving regulatory landscape, Questo maintains SCCs as a supplementary safeguard for all US-based transfers, ensuring continuity of lawful transfers regardless of future changes to the DPF's status.

---

## 8. Data Subject Rights

### 8.1 Your Rights Under GDPR

Under the GDPR, data subjects have the following rights:

| Right | Description | How to Exercise |
|---|---|---|
| **Right of Access** (Art. 15) | You have the right to obtain confirmation of whether your personal data is being processed and, if so, to receive a copy of that data along with information about the processing. | Contact the operator of the application you use (our customer). For data we hold as controller, contact privacy@questo.io. |
| **Right to Rectification** (Art. 16) | You have the right to have inaccurate personal data corrected and incomplete data completed. | Contact the operator of the application you use. For data we hold as controller, contact privacy@questo.io. |
| **Right to Erasure** (Art. 17) | You have the right to have your personal data deleted where there is no compelling reason for its continued processing (subject to legal exceptions). | Contact the operator of the application you use. For data we hold as controller, contact privacy@questo.io. |
| **Right to Restriction** (Art. 18) | You have the right to restrict the processing of your personal data in certain circumstances (e.g., while the accuracy of data is being verified). | Contact the operator of the application you use. For data we hold as controller, contact privacy@questo.io. |
| **Right to Data Portability** (Art. 20) | You have the right to receive your personal data in a structured, commonly used, and machine-readable format (JSON) and to transmit it to another controller. | Contact the operator of the application you use. For data we hold as controller, contact privacy@questo.io. |
| **Right to Object** (Art. 21) | You have the right to object to processing based on legitimate interest or for direct marketing purposes. | Contact the operator of the application you use. For data we hold as controller, contact privacy@questo.io. |
| **Rights related to Automated Decision-Making** (Art. 22) | You have the right not to be subject to a decision based solely on automated processing that produces legal or similarly significant effects, and to obtain human intervention, express your point of view, and contest such a decision. | Contact the operator of the application you use. The corteX platform provides goal-tracking and decision-logging capabilities that support explainability. |

### 8.2 How We Support Data Subject Rights

**For end users (Questo as Processor):** Data subject requests should be directed to the customer (Controller) operating the application. When we receive a request from a customer to assist with a data subject request, we provide the following capabilities through our DSAR API:

- **Data export** in structured JSON format (for access and portability requests);
- **Data correction** across all storage layers (for rectification requests);
- **Data deletion** from all storage layers including conversations, memory stores, weight deltas, and preferences (for erasure requests);
- **Processing restriction** for specific users (for restriction requests);
- **Objection registration** for specific processing activities such as profiling and learning (for objection requests).

We respond to customer DSAR assistance requests within five (5) business days.

**For direct requests from end users:** If we receive a data subject request directly from an end user, we will notify the relevant customer and direct the end user to contact the customer. If the end user is unable to identify or reach the customer, we will make reasonable efforts to identify the customer and facilitate the request.

**For our own data (Questo as Controller):** You may exercise your rights by contacting privacy@questo.io. We will respond within one (1) month of receiving your request. If the request is complex or if we receive a large number of requests, we may extend this period by up to two (2) additional months, informing you of the extension within the initial one-month period.

### 8.3 Right to Lodge a Complaint

You have the right to lodge a complaint with a supervisory authority in the EU Member State of your habitual residence, place of work, or place of the alleged infringement. A list of supervisory authorities is available at: https://edpb.europa.eu/about-edpb/about-edpb/members_en

---

## 9. Data Retention

### 9.1 Customer Platform Data

Retention of customer platform data is determined by the customer's configuration:

| Retention Mode | Behaviour |
|---|---|
| **NONE** | Personal data is not persisted beyond the immediate processing operation |
| **SESSION** | Personal data is retained for the duration of the user session and deleted at session end |
| **PERSISTENT** | Personal data is retained until explicitly deleted by the customer or via a DSAR |
| **AUDIT_ONLY** | Only audit log entries are retained; conversation content and memory data are deleted |

The customer configures retention per tenant. Automated TTL (time-to-live) enforcement ensures data is deleted according to the configured schedule.

### 9.2 Customer Account Data

| Data Type | Retention Period |
|---|---|
| Account and billing information | Duration of customer relationship plus 7 years (legal/accounting obligation) |
| Support communications | 3 years after resolution |
| Contracts and DPAs | Duration of customer relationship plus 10 years |

### 9.3 Website Data

| Data Type | Retention Period |
|---|---|
| Server logs (anonymised IP) | 90 days |
| Documentation usage analytics | Aggregated and anonymised; retained indefinitely |

### 9.4 Deletion Upon Termination

When a customer relationship ends, we follow the data return and deletion procedures specified in the DPA:

- The customer may request return of all data in JSON format or deletion of all data;
- If no instruction is received within 30 days of termination, all customer data is deleted;
- Backup copies are deleted within 90 days of the deletion instruction;
- We provide a written deletion certificate upon completion.

---

## 10. Security

We implement appropriate technical and organisational measures to protect personal data, including:

| Security Domain | Measures |
|---|---|
| **Encryption** | Encryption at rest for all stored personal data. TLS 1.3 encryption in transit for all communications. Fernet encryption (AES-128-CBC + HMAC-SHA256, PBKDF2-derived 256-bit key) with tenant-derived keys for API key material in KeyVault. |
| **Access Control** | Role-based access control (RBAC) with least-privilege principles. Multi-factor authentication (MFA) for all personnel accessing production systems. Quarterly access reviews. |
| **Tenant Isolation** | Architectural isolation at the data layer. Cross-tenant data access is prevented by design. Separate cryptographic key material per tenant. |
| **Monitoring** | Comprehensive audit logging of all data access and processing activities. Automated alerting for security events. Minimum 90-day log retention. |
| **PII Protection** | DataClassifier pipeline for detecting personal data in processing streams. Configurable redaction before LLM transmission. Special category data detection. |
| **Vulnerability Management** | Automated dependency scanning. Defined remediation timelines. Annual external penetration testing. Continuous code review. |
| **Incident Response** | Documented incident response plan with severity classification (P0-P3). Breach notification to affected customers within 24 hours of awareness. Post-incident review process. |

For full details on our technical and organisational measures, please refer to Annex B of the Data Processing Agreement.

---

## 11. Profiling and Automated Decision-Making

### 11.1 Brain-Inspired Learning

The corteX platform includes a brain-inspired learning system with features such as synaptic weights, plasticity, and prediction/surprise mechanisms. When per-user learning is enabled by the customer, this system may constitute profiling under Article 4(4) of the GDPR, as it:

- Evaluates personal aspects of end users;
- Analyses and predicts interaction preferences and behavior; and
- Adapts AI agent responses based on learned patterns.

### 11.2 Customer Control

Per-user learning and profiling are not enabled by default. The customer controls whether these features are active through the Service's configuration. Customers can:

- Enable or disable per-user learning at the tenant level;
- Allow individual end users to opt out of learning and profiling;
- Delete per-user learning data (weight deltas, episodic memory, preferences) via the DSAR API;
- Access explanations of agent decisions through goal-tracking and decision-logging capabilities.

### 11.3 Data Subject Rights Regarding Profiling

Where profiling is active, end users have the right to:

- Be informed that profiling is taking place (customer's responsibility under Articles 13-14);
- Object to profiling under Article 21;
- Not be subject to decisions based solely on automated processing that produce legal or similarly significant effects, under Article 22;
- Obtain human intervention, express their point of view, and contest such decisions.

---

## 12. Special Category Data

End users may share special category data (as defined in Article 9 of the GDPR) in conversations with AI agents, including health information, political opinions, religious beliefs, or other sensitive data.

The corteX platform provides the following technical safeguards:

- **PII Detection (DataClassifier):** Capable of detecting special category data in conversation streams;
- **Configurable Redaction:** Customers can configure automatic redaction of special category data before it is transmitted to LLM providers;
- **Memory Controls:** Customers can prevent special category data from being stored in long-term memory;
- **Awareness:** This Privacy Policy and customer-facing documentation inform about the possibility of special category data being present in conversations.

Customers are responsible for determining whether the processing of special category data is lawful and for implementing appropriate safeguards.

---

## 13. Children's Data

The corteX platform is a B2B service intended for integration into enterprise applications. Questo does not knowingly collect or process personal data from children under 16 years of age.

If a customer's application is directed at or accessible to children, the customer is responsible for ensuring compliance with applicable age-related data protection requirements (including obtaining parental consent where required under Article 8 of the GDPR) and for configuring appropriate safeguards within the Service.

---

## 14. Cookies and Tracking Technologies

### 14.1 Our Website

The Questo website (cortex-ai.com) and documentation site (docs.cortex-ai.com) use the following technologies:

| Category | Technology | Purpose | Legal Basis |
|---|---|---|---|
| **Strictly Necessary** | Session cookies | Website functionality, security | Legitimate Interest (exempt from consent requirement under ePrivacy Directive) |
| **Analytics** | Privacy-focused analytics (no personal data collected) | Understanding documentation usage to improve developer experience | Legitimate Interest; anonymised data only |

We do not use third-party advertising cookies or tracking technologies on our website.

### 14.2 The corteX Platform

The corteX SDK is a backend service and does not set cookies in end-user browsers. The SDK communicates via API calls and does not include any browser-side tracking technologies.

---

## 15. Changes to This Privacy Policy

We may update this Privacy Policy from time to time to reflect changes in our processing activities, legal requirements, or business practices. When we make material changes:

- We will update the "Last Updated" date at the top of this policy;
- We will notify customers via email at least 30 days before the changes take effect;
- We will maintain an archive of previous versions of this policy, available upon request.

We encourage you to review this Privacy Policy periodically.

---

## 16. Contact Information

For questions, concerns, or requests regarding this Privacy Policy or our data processing practices:

| | |
|---|---|
| **Privacy Inquiries** | privacy@questo.io |
| **DPA Requests** | dpa@questo.io |
| **Mailing Address** | Questo Ltd, Israel |
| **Response Time** | We aim to respond to all privacy inquiries within five (5) business days. Data subject rights requests will be actioned within one (1) month, as required by the GDPR. |

For urgent matters related to a suspected data breach, please contact privacy@questo.io with "URGENT: Data Breach" in the subject line.

---

## 17. Supervisory Authority

As an Israeli company, Questo is subject to the Israeli Privacy Protection Authority (PPA). For matters relating to the processing of personal data of EEA residents, the lead supervisory authority is determined by the customer's (Controller's) establishment in the EEA.

Data subjects in the EEA may lodge a complaint with:

- The supervisory authority in the Member State of their habitual residence, place of work, or place of the alleged infringement;
- A full list of supervisory authorities is available at: https://edpb.europa.eu/about-edpb/about-edpb/members_en

---

*This Privacy Policy is effective as of 2026-02-16. Questo Ltd reserves the right to update this policy in accordance with Section 15 above.*
