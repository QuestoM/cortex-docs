# Data Classification Policy

**Document ID**: POL-008
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Data Classification Policy defines the framework for categorizing all information assets owned, managed, or processed by Questo Ltd according to their sensitivity and the impact of unauthorized disclosure. It establishes a four-tier classification system that maps directly to the `DataLevel` enum implemented programmatically in `corteX/security/classification.py`, ensuring that policy and code are aligned.

This policy governs how data is labeled, handled, stored, transmitted, and destroyed across its lifecycle. It addresses the unique data types inherent to an AI agent platform, including customer prompts, LLM API keys, agent outputs, brain-inspired learning weights, and tenant configuration data.

**SOC 2 Mapping**: CC6.1 (Logical Access), CC6.5 (Data Disposal), C1.1-C1.2 (Confidentiality)

## 2. Scope

This policy applies to:

- **Data**: All data created, received, stored, transmitted, or processed by Questo Ltd, regardless of format (digital, physical, verbal).
- **Systems**: All systems that store, process, or transmit Questo data, including development environments, production infrastructure, cloud services, communication platforms, and personal devices used for Questo business.
- **Personnel**: All employees, contractors, and third parties with access to Questo data.
- **corteX SDK Data Flows**: Data processed by the corteX SDK on behalf of tenants, including prompts, API keys, agent outputs, memory contents, audit logs, and learning weights.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Data Classification** | The process of categorizing data based on its sensitivity and the impact of unauthorized disclosure |
| **Data Level** | One of the four classification tiers: PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED |
| **DataLevel Enum** | The programmatic representation of classification levels in `corteX/security/classification.py` |
| **DataClassifier** | The corteX module that automatically detects PII and assigns classification levels |
| **Data Owner** | The person or role responsible for the accuracy of classification for specific data |
| **Data Custodian** | The person or role responsible for implementing the technical controls required by the classification |
| **PII** | Personally Identifiable Information; data that can identify a natural person |
| **Classification Escalation** | The principle that data classification can only increase (never decrease) as it flows through processing stages |
| **Labeling** | The act of marking data with its classification level |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and ensures its implementation across all systems and data flows
- Approves classification decisions for ambiguous data
- Reviews classification accuracy during quarterly security reviews
- Ensures the `DataClassifier` module aligns with this policy

### 4.2 Security Lead (Data Custodian)

- Implements and monitors technical controls for each classification level
- Conducts periodic classification audits to verify accuracy
- Maintains the data inventory with current classifications
- Manages the PII detection patterns in `corteX/security/classification.py`

### 4.3 Data Owners (Designated per Data Category)

- Assign the initial classification to data within their domain
- Review classifications annually or when data characteristics change
- Authorize access to data within their domain per the classification requirements

### 4.4 All Personnel

- Handle data in accordance with its assigned classification level
- Report suspected misclassification to the Security Lead
- Do not downgrade data classification without authorization
- Apply the highest applicable classification when uncertain

## 5. Policy Statements

### 5.1 Classification Levels

Questo Ltd uses four classification levels, ordered from least to most sensitive. These levels correspond directly to the `DataLevel` IntEnum in `corteX/security/classification.py`:

#### 5.1.1 PUBLIC (DataLevel.PUBLIC = 0)

**Definition**: Information approved for unrestricted public access. Disclosure poses no risk to Questo Ltd or its customers.

| Attribute | Requirement |
|-----------|-------------|
| **Access** | Unrestricted |
| **Storage** | No special requirements |
| **Transmission** | No encryption required (HTTPS preferred) |
| **LLM Processing** | May be sent to any model, including cloud-hosted LLMs |
| **Labeling** | Optional |
| **Disposal** | Standard deletion |

**Examples**: Published SDK documentation, public API specifications, marketing materials, published blog posts, open-source license text, PyPI package metadata.

#### 5.1.2 INTERNAL (DataLevel.INTERNAL = 1)

**Definition**: Information intended for Questo Ltd personnel only. Disclosure would cause moderate business impact but no direct harm to customers.

| Attribute | Requirement |
|-----------|-------------|
| **Access** | Questo employees and authorized contractors only |
| **Storage** | Access-controlled systems; encryption at rest recommended |
| **Transmission** | TLS 1.2+ required |
| **LLM Processing** | On-premises models only; cloud LLMs prohibited without explicit data handling agreement |
| **Labeling** | Required on documents and communications |
| **Disposal** | Secure deletion (overwrite or cryptographic erasure) |

**Examples**: Internal meeting notes, project plans, non-sensitive configuration files, internal procedures, this policy document, draft documentation, engineering design documents.

#### 5.1.3 CONFIDENTIAL (DataLevel.CONFIDENTIAL = 2)

**Definition**: Sensitive business information or customer data whose disclosure would cause significant harm to Questo Ltd or its customers.

| Attribute | Requirement |
|-----------|-------------|
| **Access** | Need-to-know basis; documented authorization required |
| **Storage** | Encryption at rest mandatory (AES-256 minimum) |
| **Transmission** | TLS 1.3 required; end-to-end encryption for sensitive transfers |
| **LLM Processing** | On-premises models only; full audit trail required per `AuditConfig` |
| **Labeling** | Required on all documents, files, and communications |
| **Disposal** | Cryptographic erasure or certified destruction |

**Examples**: Source code, customer prompts and agent outputs, architecture documents, employee personal data, financial records, vendor contracts, customer lists, non-public roadmaps, brain-inspired weight data (tenant-scoped), customer configuration data.

#### 5.1.4 RESTRICTED (DataLevel.RESTRICTED = 3)

**Definition**: The highest sensitivity level. Disclosure would cause severe, potentially irreversible harm to Questo Ltd or its customers. Compromise of RESTRICTED data could result in direct financial loss, legal liability, or existential threat.

| Attribute | Requirement |
|-----------|-------------|
| **Access** | Explicitly authorized individuals only; CTO approval required; logged access |
| **Storage** | Encryption at rest mandatory (Fernet: AES-128-CBC + HMAC-SHA256, PBKDF2-derived 256-bit key, in KeyVault); access logging |
| **Transmission** | TLS 1.3 required; end-to-end encryption mandatory |
| **LLM Processing** | On-premises models only; explicit approval and full audit trail required |
| **Labeling** | Required; conspicuous marking on all instances |
| **Disposal** | Cryptographic erasure with documented verification |

**Examples**: Customer API keys (stored in KeyVault), Ed25519 license signing keys, Fernet encryption master keys, PBKDF2-derived keys, database credentials, cloud provider root credentials, incident forensic evidence.

### 5.2 Classification Escalation Principle

Data classification can only escalate (increase in sensitivity), never downgrade, as data flows through the corteX processing pipeline. This is enforced programmatically through the `DataLevel` IntEnum, where `max()` always selects the most restrictive level.

For example:
- A PUBLIC prompt that contains a detected email address (PII) is escalated to CONFIDENTIAL.
- An INTERNAL document referencing an API key is escalated to RESTRICTED.
- Agent output derived from CONFIDENTIAL input remains CONFIDENTIAL or higher.

Downgrading a classification requires written authorization from the Data Owner and CTO approval, with documented justification.

### 5.3 Automatic PII Detection

The `DataClassifier` module (`corteX/security/classification.py`) automatically scans data for PII patterns, including:

| PII Type | Detection Pattern | Auto-Classification |
|----------|------------------|-------------------|
| Email addresses | RFC-compliant email regex | CONFIDENTIAL |
| US phone numbers | 10-digit with formatting variants | CONFIDENTIAL |
| Social Security Numbers | 9-digit SSN format | RESTRICTED |
| Credit card numbers | Visa, MasterCard, Amex, Discover patterns | RESTRICTED |
| Israeli ID numbers | 9-digit Teudat Zehut format | CONFIDENTIAL |

When PII is detected, the data classification is automatically escalated to the corresponding level. This detection operates on all data entering the corteX processing pipeline, including prompts, tool inputs, and agent outputs.

### 5.4 Data Handling Requirements by Category

#### 5.4.1 Customer API Keys (RESTRICTED)

- Stored exclusively in per-tenant `KeyVault` instances (`corteX/security/vault.py`).
- Encrypted with Fernet (AES-128-CBC + HMAC-SHA256) using a PBKDF2-derived 256-bit master key (600,000 iterations). Fernet uses AES-128-CBC per its specification; the 256-bit derived key is split into a 128-bit AES key and a 128-bit HMAC key.
- Keys never touch disk or logs. `KeyVault.detect_leak()` monitors all outputs for accidental exposure.
- Rotation is atomic: old key material is zeroed before new key is stored.
- In BYOK deployments, Questo never accesses or possesses tenant API keys.

#### 5.4.2 corteX Source Code (CONFIDENTIAL)

- Stored in private GitHub repositories with branch protection.
- Access restricted to authorized personnel per POL-002 RBAC definitions.
- Copying to personal devices requires CTO authorization.
- Open-sourcing any component requires CEO and CTO approval.

#### 5.4.3 Customer Prompts and Outputs (CONFIDENTIAL)

- Processed in tenant-isolated sessions. Cross-tenant access is architecturally impossible.
- Not stored by Questo infrastructure unless the customer explicitly enables audit logging.
- In on-premises deployments, prompts never leave the customer's environment.

#### 5.4.4 Brain-Inspired Weight Data (CONFIDENTIAL)

- Synaptic weights (`corteX/engine/weights.py`) and plasticity state (`corteX/engine/plasticity.py`) are scoped to individual tenants.
- Weight data may encode behavioral patterns and is classified as CONFIDENTIAL.
- Weight export and import operations are logged in the audit trail.

### 5.5 Data Inventory

The Security Lead maintains a data inventory that catalogs:

- Data category and description
- Classification level
- Data owner
- Storage location(s)
- Retention period
- Disposal method
- Associated compliance requirements (GDPR, SOC 2)

The data inventory is reviewed and updated at least annually, or whenever a new data category is introduced.

### 5.6 Data Retention and Disposal

| Classification | Minimum Retention | Maximum Retention | Disposal Method |
|----------------|-------------------|-------------------|-----------------|
| PUBLIC | None | As needed | Standard deletion |
| INTERNAL | Per business need | 3 years after last use | Secure deletion |
| CONFIDENTIAL | Per contractual/regulatory requirement | 5 years or as required | Cryptographic erasure |
| RESTRICTED | Per regulatory requirement | 7 years or as required | Cryptographic erasure with verification |

Audit logs are retained for a minimum of 3 years per SOC 2 requirements, regardless of the classification of the data they reference.

## 6. Procedures

### 6.1 Data Classification Procedure

1. The Data Owner identifies new data or data whose characteristics have changed.
2. The Data Owner evaluates the data against the classification criteria in Section 5.1.
3. Apply the highest applicable classification level. When in doubt, classify higher.
4. Label the data per the labeling requirements of the assigned level.
5. Record the classification in the data inventory.
6. Implement the technical controls required for the classification level.

### 6.2 Classification Review Procedure

1. The Security Lead schedules annual classification reviews for all data categories in the inventory.
2. Data Owners review their assigned data and confirm or update classifications.
3. The Security Lead verifies that technical controls match the assigned classifications.
4. Changes are documented in the data inventory with the review date and reviewer.
5. The CTO reviews and approves any classification changes affecting RESTRICTED data.

### 6.3 Data Disposal Procedure

1. The Data Owner or custodian identifies data that has reached the end of its retention period.
2. Confirm that no legal holds, regulatory requirements, or contractual obligations require continued retention.
3. Execute disposal per the method specified for the classification level (Section 5.6).
4. For CONFIDENTIAL and RESTRICTED data: document the disposal action with date, method, data description, and person executing disposal.
5. For RESTRICTED data: a second person verifies that disposal is complete.
6. Update the data inventory to reflect the disposal.

### 6.4 PII Discovery Procedure

1. The `DataClassifier` module automatically scans data flowing through the corteX pipeline.
2. When PII is detected, the classification is automatically escalated per Section 5.3.
3. Detection events are logged in the audit trail.
4. Monthly, the Security Lead reviews PII detection logs for patterns indicating systematic data handling issues.
5. New PII patterns identified through regulatory changes or incident reviews are added to the `_PII_PATTERNS` dictionary in `corteX/security/classification.py` through the standard change management process (POL-003).

## 7. Exceptions

1. Classification downgrades require written approval from the Data Owner and CTO.
2. Temporary exceptions to handling requirements (e.g., transmitting CONFIDENTIAL data over a channel that does not meet TLS 1.3) require CTO approval, documented compensating controls, and a maximum duration of 7 days.
3. No exceptions are permitted for: RESTRICTED data handling requirements, the classification escalation principle, or PII auto-detection.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Handling data in a manner inconsistent with its classification is a policy violation subject to disciplinary action.
- Intentional misclassification (classifying data lower than warranted to avoid controls) is a serious policy violation.
- Failure to label CONFIDENTIAL or RESTRICTED data is a policy violation.
- The Security Lead monitors classification compliance through periodic audits and automated detection.
- Violations involving RESTRICTED data are reported to the CTO immediately and may result in access suspension.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-005 | Risk Assessment and Mitigation Policy |
| POL-011 | Encryption Policy |
| POL-017 | Logging and Monitoring Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
