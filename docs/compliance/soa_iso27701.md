# Statement of Applicability (SoA) - ISO/IEC 27701:2025

**Document ID**: SOA-001
**Version**: 1.0
**Effective Date**: 2026-02-23
**Last Reviewed**: 2026-02-23
**Next Review**: 2026-08-23
**Owner**: CTO, Questo Ltd
**Approved By**: CEO, Questo Ltd
**Classification**: CONFIDENTIAL

---

## 1. Scope

**Organization**: Questo Ltd, an Israeli software company.

**Scope of PIMS**: Development, maintenance, and distribution of the corteX AI Agent SDK
(`cortex-ai` on PyPI) as a Privacy Information Management System (PIMS) operating
primarily in the role of **PII Processor** under GDPR Article 28.

**Role Determination**: Questo acts as a PII Processor. Customers (SaaS developers and
their enterprise clients) are PII Controllers. Questo develops and distributes the SDK
package; it does not host, operate, or access customer deployments. All PII processing
occurs within the customer's own infrastructure using the SDK.

**Applicable Regulations**: GDPR (EU 2016/679), Israeli Privacy Protection Law 5741-1981,
EU AI Act (anticipated Aug 2026), SOC 2 Type 1 (Security, Confidentiality, Processing
Integrity).

**Standard Reference**: ISO/IEC 27701:2025 - Privacy information management systems -
Requirements and guidance.

---

## 2. Management System Clauses (4-10)

These clauses define the PIMS management framework. All are applicable to Questo Ltd.

| Clause | Title | Applicable | Implementation Reference |
|--------|-------|------------|------------------------|
| 4 | Context of the organization | Yes | System Description (SD-001), ROPA (GDPR-001), Risk Register (REG-001) |
| 4.1 | Understanding the organization and its context | Yes | SD-001 Section 1; privacy risks in REG-001 Section 3.5 |
| 4.2 | Understanding the needs and expectations of interested parties | Yes | SD-001 Section 7 (CUECs); ROPA Section 3 (Controller Registry) |
| 4.3 | Determining the scope of the PIMS | Yes | This document, Section 1 |
| 4.4 | Privacy information management system | Yes | 20 policies (POL-001 through POL-020); compliance code modules |
| 5 | Leadership | Yes | CEO/CTO approval on all policies; CTO as acting DPO |
| 5.1 | Leadership and commitment | Yes | POL-001 Section 1 (management commitment); CEO sign-off on policies |
| 5.2 | Policy | Yes | POL-001 Information Security Policy; POL-014 Confidentiality Policy |
| 5.3 | Organizational roles, responsibilities, and authorities | Yes | SD-001 Section 3.3 (People); POL-013 Code of Conduct |
| 6 | Planning | Yes | REG-001 Risk Register; POL-005 Risk Assessment Policy |
| 6.1 | Actions to address risks and opportunities | Yes | REG-001 Sections 3.1-3.5 (24 risks identified and scored) |
| 6.2 | Privacy objectives and planning to achieve them | Yes | POL-001 Section 5 (security objectives); compliance roadmap |
| 7 | Support | Yes | SD-001 Section 3; security training materials |
| 7.1 | Resources | Yes | SD-001 Section 3.3 (engineering team roles) |
| 7.2 | Competence | Yes | SD-001 Section 8.2; security awareness training (security_training.md) |
| 7.3 | Awareness | Yes | POL-013 Code of Conduct; security_training.md |
| 7.4 | Communication | Yes | POL-004 Incident Response Plan (communication templates) |
| 7.5 | Documented information | Yes | 20 policies, ROPA, DPIA template, risk register, system description |
| 8 | Operation | Yes | SDK codebase; CI/CD pipeline; release management |
| 8.1 | Operational planning and control | Yes | POL-003 Change Management; .github/workflows/ci.yml |
| 8.2 | Privacy risk assessment | Yes | REG-001; DPIA Template (DPIA_Template.md) |
| 8.3 | Privacy risk treatment | Yes | REG-001 Section 2.4 (treatment options); audit_report_2026-02-17.md |
| 9 | Performance evaluation | Yes | Audit report (quarterly); CI/CD test results (9,300+ tests) |
| 9.1 | Monitoring, measurement, analysis, and evaluation | Yes | AuditLogger hash-chain verification; CI test dashboards |
| 9.2 | Internal audit | Yes | audit_report_2026-02-17.md (22 PASS, 10 gaps all resolved) |
| 9.3 | Management review | Yes | Quarterly review cycle (POL-001 Section 6.3) |
| 10 | Improvement | Yes | Gap remediation process; audit findings tracked to resolution |
| 10.1 | Nonconformity and corrective action | Yes | 10 gaps identified, all resolved with 95 new tests |
| 10.2 | Continual improvement | Yes | Quarterly audits; continuous test suite expansion |

---

## 3. Annex A.2 - PII Processor Controls

As the primary role for Questo is PII Processor, these controls are fully in scope.

### 3.1 Conditions for Collection and Processing

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.2.1 | Customer agreement | Yes | DPA template covers all processing terms per GDPR Art. 28. Processing instructions defined in customer agreements. | DPA template; ROPA (GDPR-001) Section 3 |
| A.2.2 | Organization's purposes | Yes | SDK processes PII solely per customer instructions. No secondary use. LLM calls use only customer-provided prompts. | POL-001 Section 5.2; ComplianceEngine enforces purpose limitation |
| A.2.3 | Marketing and advertising use | Yes | Questo does not use customer PII for marketing. SDK collects no telemetry or analytics from customer deployments. | POL-014 Confidentiality Policy; no telemetry in SDK code |
| A.2.4 | Infringing instruction | Yes | SDK provides ComplianceEngine pre-check to flag instructions that may violate active compliance frameworks. | `compliance.py` pre_check(); POL-001 Section 5.10 |
| A.2.5 | Customer obligations | Yes | CUEC documentation defines customer responsibilities. Customers must configure TenantConfig appropriately. | SD-001 Section 7 (10 CUECs documented) |
| A.2.6 | Records related to processing PII | Yes | ROPA maintained per GDPR Art. 30(2). AuditLogger records all processing events. | ROPA (GDPR-001); `audit_logger.py` hash-chained JSONL |

### 3.2 Obligations to PII Principals

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.2.7 | Obligations to PII principals | Yes | GDPRManager provides full DSAR API (7 rights). Processor assists controller in fulfilling data subject requests. | `gdpr.py` (Art. 15-22 coverage); POL-001 Section 5.4 |

### 3.3 Privacy by Design and by Default

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.2.8 | Temporary files | Yes | SDK does not create persistent temporary files with PII. Working memory is in-process only. Retention enforcer manages data lifecycle. | `retention.py`; POL-008 Section 5.6 |
| A.2.9 | Return, transfer, or disposal of PII | Yes | TenantKeyManager.destroy_tenant_keys() provides cryptographic erasure. GDPRManager.erase_user_data() cascades deletion. RetentionEnforcer automates purges. | `tenant_encryption.py` destroy_tenant_keys(); `gdpr.py` erase_user_data(); `retention.py` |
| A.2.10 | PII transmission controls | Yes | DataClassifier blocks INTERNAL+ data from cloud providers. Data residency enforcement validates provider regions. PII tokenization redacts PII before LLM calls. | `classification.py`; `data_residency.py`; `pii_tokenizer.py`; router.py classification gate |
| A.2.11 | PII processing basis | Yes | Processing occurs only on documented legal basis. ComplianceEngine validates consent before processing when GDPR active. | `compliance.py` _resolve_consent(); `consent.py` |

### 3.4 PII Sharing, Transfer, and Disclosure

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.2.12 | Sub-processor management | Yes | SDK routes to LLM sub-processors (OpenAI, Gemini, Anthropic). Sub-processor list maintained. DPA template requires customer notification of sub-processor changes. | SD-001 Section 6 (Subservice Organizations); POL-009 Vendor Management |
| A.2.13 | Disclosure of sub-processors | Yes | Sub-processor registry maintained in system description. Changes communicated to customers. | SD-001 Section 6; DPA template notification clause |
| A.2.14 | Change of sub-processor | Yes | Questo notifies customers before adding LLM providers. Customers can object per DPA terms. | DPA template; POL-009 Section 5 |
| A.2.15 | Countries and international organizations for PII transfer | Yes | DataResidencyManager validates provider regions against tenant restrictions. Provider-to-region mapping covers all supported providers. | `data_residency.py` lines 67-113; POL-008 Section 5.5 |
| A.2.16 | Basis for PII transfer between jurisdictions | Yes | EU-US Data Privacy Framework (DPF) + Standard Contractual Clauses (SCCs) as backup for US providers. On-prem deployment eliminates transfer. | REG-001 R-CMP-003; DPA template transfer mechanism |
| A.2.17 | Records of PII disclosure to third parties | Yes | AuditLogger records all LLM API calls including provider and model used. Hash-chained for tamper evidence. | `audit_logger.py` LLM_CALL events; `audit_types.py` |
| A.2.18 | Notification of PII disclosure requests | Yes | DPA template requires Questo to notify controller of any legally compelled disclosure. | DPA template; POL-004 Section 5.4 |
| A.2.19 | Legally binding PII disclosures | Yes | DPA template addresses law enforcement and government access requests. Questo notifies controller unless legally prohibited. | DPA template; POL-014 Section 5.3 |
| A.2.20 | Joint controller | Partial | Questo is primarily a processor, not a joint controller. However, the SDK architecture supports configuring roles per customer agreement. | DPA template role definitions; ROPA role documentation |
| A.2.21 | PII breach notification | Yes | POL-004 Incident Response Plan defines 72-hour GDPR breach notification. Questo notifies controllers without undue delay. | POL-004 Section 5.4 (GDPR notification); REG-001 R-OPS-006 |

---

## 4. Annex A.1 - PII Controller Controls

Questo is primarily a PII Processor. Controller controls are addressed where Questo may
act as controller for limited purposes (e.g., employee data, website visitors, customer
contact data for contract management).

### 4.1 Conditions for Collection and Processing

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.1.1 | Identification and documentation of purpose | Partial | Applies only to Questo's own data processing (employee data, customer contacts). SDK processing purposes defined by customers. | ROPA (GDPR-001); employment contracts |
| A.1.2 | Identification of lawful basis | Partial | For Questo's own processing: legitimate interest (business contacts), consent (website), contract (employees). SDK does not determine lawful basis for customers. | ROPA Section 4; POL-001 |
| A.1.3 | Determining when and how consent is to be obtained | Partial | Applies to Questo's website and marketing. SDK provides ConsentManager for customers to implement consent. | `consent.py`; Questo website privacy policy |
| A.1.4 | Obtaining and recording consent | Partial | ConsentManager module available for customer use. Questo records consent for its own activities. | `consent.py` record_consent(), export_consents() |
| A.1.5 | Privacy impact assessment | Yes | DPIA template provided. Questo conducts DPIAs for SDK features processing PII. | DPIA_Template.md; POL-005 Risk Assessment |
| A.1.6 | Contracts with PII processors | Yes | Questo engages LLM providers as sub-processors with DPAs. | SD-001 Section 6; POL-009 Vendor Management |
| A.1.7 | Joint PII controller | Partial | Not typical for Questo. Addressed in DPA where necessary. | DPA template |
| A.1.8 | Records related to processing PII | Yes | ROPA maintained per GDPR Art. 30. | ROPA (GDPR-001) |

### 4.2 Obligations to PII Principals

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.1.9 | Providing notice to PII principals | Partial | Questo provides privacy notice for its own data collection. SDK provides GDPRManager.get_processing_info() for customers. | `gdpr.py` get_processing_info(); Questo privacy policy |
| A.1.10 | Providing mechanism to modify or withdraw consent | Partial | ConsentManager.withdraw_consent() implemented. Applies to Questo's own consent processing. | `consent.py` withdraw_consent(), withdraw_all() |
| A.1.11 | Providing mechanism to object to PII processing | Partial | ProfilingManager.opt_out() implemented. SDK supports objection registration. | `profiling.py` opt_out(); `gdpr.py` register_objection() |
| A.1.12 | Obligation to inform PII principals | Partial | For Questo's own processing. SDK provides transparency features for customers. | `gdpr.py` get_processing_info() (Art. 13-14) |
| A.1.13 | PII principal access request | Partial | GDPRManager.export_user_data() for Art. 15 access. Applies to SDK functionality. | `gdpr.py` export_user_data() |
| A.1.14 | Right to rectification and erasure | Partial | GDPRManager provides rectify_user_data() and erase_user_data(). Cryptographic erasure via TenantKeyManager. | `gdpr.py` Art. 16-17; `tenant_encryption.py` |
| A.1.15 | PII principal obligations | Partial | Customers inform PII principals through their own applications. SDK provides tools, not direct end-user interfaces. | CUEC documentation; SD-001 Section 7 |

### 4.3 Privacy by Design and by Default

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.1.16 | Limit collection | Yes | PII tokenization redacts unnecessary PII before LLM processing. Data minimization enforced by DataClassifier. | `pii_tokenizer.py`; `classification.py` |
| A.1.17 | Limit processing | Yes | ComplianceEngine enforces purpose limitation. Processing restricted to customer-defined scope. | `compliance.py` pre_check(); TenantConfig |
| A.1.18 | Accuracy and quality | Partial | GoalTracker and PredictionEngine verify processing accuracy. Output validation via SafetyPolicy. | `goal_tracker.py`; SafetyPolicy.check_output() |
| A.1.19 | PII minimization objectives | Yes | SDK design minimizes PII in LLM context. PII tokenization replaces PII with reversible tokens. | `pii_tokenizer.py`; POL-008 Section 5.3 |
| A.1.20 | PII de-identification and deletion at end of processing | Yes | RetentionEnforcer manages data lifecycle. Cryptographic erasure on tenant offboarding. | `retention.py`; `tenant_encryption.py` destroy_tenant_keys() |
| A.1.21 | Temporary files | Yes | SDK manages in-memory working state. No persistent temporary PII files. | Memory architecture (working memory is transient) |

### 4.4 PII Sharing, Transfer, and Disclosure

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.1.22 | Identifying basis for PII transfer | Yes | DataResidencyManager validates all cross-border transfers. Transfer mechanisms documented. | `data_residency.py`; DPA template |
| A.1.23 | Countries and international organizations for PII transfer | Yes | Provider-to-region mapping maintained. Tenant-level region restrictions enforced. | `data_residency.py` lines 67-113; POL-008 |
| A.1.24 | Records of PII transfer | Yes | AuditLogger records all LLM calls with provider and region metadata. | `audit_logger.py` LLM_CALL events |
| A.1.25 | Records of PII disclosure to third parties | Yes | Hash-chained audit log captures all external data transmissions. | `audit_logger.py`; `audit_types.py` |
| A.1.26 | PII disclosure notification | Yes | DPA template defines notification obligations. Incident response plan covers breach disclosure. | POL-004; DPA template |

---

## 5. Annex A.3 - Shared Information Security Controls

These controls apply to both PII controllers and processors. They align with
ISO/IEC 27002:2022 themes: organizational, people, physical, and technological.

### 5.1 Organizational Controls

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.3.1 | Policies for information security | Yes | 20 formal policies (POL-001 through POL-020) covering all aspects of information security and privacy. | POL-001 through POL-020 in docs/compliance/policies/ |
| A.3.2 | Information security roles and responsibilities | Yes | Roles defined in SD-001 Section 3.3. CTO as acting DPO. | SD-001; POL-013 Code of Conduct |
| A.3.3 | Segregation of duties | Partial | Small team with overlapping roles. Mitigated by code review requirements and branch protection. | POL-003 Section 5.2; GitHub branch protection |
| A.3.4 | Management responsibilities | Yes | CEO/CTO approve policies. Management commitment documented. | POL-001 Section 1; all policy approvals |
| A.3.5 | Contact with authorities | Yes | DPO contact established. Supervisory authority notification procedures in place. | POL-004 Section 5.4; ROPA processor information |
| A.3.6 | Contact with special interest groups | Partial | Monitoring of AI regulation developments (EU AI Act). No formal special interest group memberships. | REG-001 R-CMP-002 (EU AI Act tracking) |
| A.3.7 | Threat intelligence | Yes | GitHub Dependabot for vulnerability monitoring. Security advisory tracking. | POL-018 Section 5.7; .github/workflows/sbom.yml |
| A.3.8 | Information security in project management | Yes | Security requirements in SDLC. ComplianceEngine integrated into development lifecycle. | POL-018 SDLC Policy; POL-003 Change Management |
| A.3.9 | Inventory of information and other associated assets | Partial | SBOM generated for software components. Hardware asset management not applicable (SDK only). | scripts/generate_sbom.py; .github/workflows/sbom.yml |
| A.3.10 | Acceptable use of information and other associated assets | Yes | Acceptable use policy covers employee and developer conduct. | POL-010 Acceptable Use Policy |
| A.3.11 | Return of assets | Partial | Applies to employee offboarding. SDK assets are code-based. GitHub access revocation on departure. | POL-010; GitHub access management |
| A.3.12 | Classification of information | Yes | 4-tier classification (PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED) enforced by DataClassifier. | `classification.py`; POL-008 Data Classification Policy |
| A.3.13 | Labelling of information | Partial | Data classification levels applied programmatically. Document classification headers on policies. | `classification.py` DataLevel; policy document headers |
| A.3.14 | Information transfer | Yes | DataResidencyManager and classification gate enforce transfer controls. TLS 1.2+ for all external communications. | `data_residency.py`; `router.py` classification gate; POL-011 |

### 5.2 People Controls

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.3.15 | Screening | Yes | Background checks on employees. NDA and confidentiality agreements. | REG-001 R-FRD-001 mitigations; POL-013 |
| A.3.16 | Terms and conditions of employment | Yes | Employment contracts include confidentiality and security clauses. | POL-014 Confidentiality Policy; employment agreements |
| A.3.17 | Information security awareness, education, and training | Yes | Security awareness training program. Phishing simulation exercises. | security_training.md; REG-001 R-FRD-002 |
| A.3.18 | Disciplinary process | Partial | Addressed in employment contracts and code of conduct. | POL-013 Code of Conduct |
| A.3.19 | Responsibilities after termination or change of employment | Yes | GitHub access revocation. NDA obligations survive termination. | POL-014; POL-016 Remote Access Policy |

### 5.3 Physical Controls

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.3.20 | Physical security perimeters | No | Not applicable - Questo does not operate data centers or server rooms. SDK runs on customer infrastructure. | N/A (SDK-only business model) |
| A.3.21 | Physical entry controls | No | Not applicable - no Questo-operated physical processing facilities. | N/A (SDK-only business model) |
| A.3.22 | Securing offices, rooms, and facilities | Partial | Developer workstations secured per POL-020. No dedicated server facilities. | POL-015 Physical Security; POL-020 Workstation Security |
| A.3.23 | Clear desk and clear screen | Yes | Clean desk policy documented. Screen lock requirements. | POL-015 Physical Security Policy; POL-020 |

### 5.4 Technological Controls

| Control | Description | Applicable | Justification | Implementation Reference |
|---------|-------------|------------|---------------|------------------------|
| A.3.24 | User endpoint devices | Yes | Workstation security policy covers developer machines. | POL-020 Workstation Security Policy |
| A.3.25 | Privileged access rights | Yes | GitHub repository access restricted via SSO. Branch protection enforces review. | POL-002 Access Control; POL-016 Remote Access |
| A.3.26 | Access control | Yes | CapabilitySet provides immutable, attenuatable permissions. ToolPolicy/ModelPolicy enforce least privilege with default-deny. | `capabilities.py`; `attenuation.py`; POL-002 |
| A.3.27 | Secure authentication | Yes | Ed25519 license validation. GitHub SSO with MFA. Per-tenant KeyVault derivation. | `vault.py` PBKDF2; POL-012 Password Policy |
| A.3.28 | Use of cryptography | Yes | KeyVault (Fernet AES-128-CBC + HMAC-SHA256), PBKDF2 600K iterations, HKDF-SHA256 per-tenant keys, Ed25519 license signing. | `vault.py`; `tenant_encryption.py`; POL-011 Encryption Policy |
| A.3.29 | Secure development lifecycle | Yes | 8-stage CI/CD pipeline. 9,300+ tests. Ruff linting. Type hints required. Max 300 lines per file. | POL-018 SDLC Policy; .github/workflows/ci.yml |
| A.3.30 | Logging and monitoring | Yes | AuditLogger with SHA-256 hash chains. 16 event types. Configurable retention (90 days online, 365 archive). | `audit_logger.py`; `audit_storage.py`; POL-017 |
| A.3.31 | Backup | Partial | SDK code backed up via Git version control and GitHub mirrors. Customer data backup is customer responsibility (CUEC-5). | POL-019 Backup Policy; Git version control |
| A.3.32 | Data masking and PII protection | Yes | PIITokenizer redacts PII before LLM calls with reversible tokens. DataClassifier detects PII patterns (email, phone, SSN, credit card, Israeli ID). KeyVault.detect_leak() scans for exposed secrets. | `pii_tokenizer.py`; `classification.py`; `vault.py` detect_leak() |

---

## 6. Controls Not Applicable - Summary

The following controls are not applicable with justification:

| Control | Reason |
|---------|--------|
| A.3.20 | Physical security perimeters - Questo does not operate data centers. SDK is pip-installable and runs entirely on customer premises. |
| A.3.21 | Physical entry controls - No Questo-operated processing facilities exist. Customer infrastructure is out of scope (CUEC-3). |

---

## 7. Summary Statistics

### 7.1 Applicability by Section

| Section | Total Controls | Fully Applicable | Partially Applicable | Not Applicable |
|---------|---------------|-----------------|---------------------|---------------|
| Clauses 4-10 (Management System) | 29 | 29 | 0 | 0 |
| A.1 (PII Controller) | 26 | 13 | 13 | 0 |
| A.2 (PII Processor) | 21 | 20 | 1 | 0 |
| A.3 (Shared Security) | 32 | 22 | 8 | 2 |
| **Total** | **108** | **84** | **22** | **2** |

### 7.2 Implementation Status

| Status | Count | Percentage |
|--------|-------|------------|
| Fully Applicable and Implemented | 84 | 77.8% |
| Partially Applicable (limited scope) | 22 | 20.4% |
| Not Applicable | 2 | 1.8% |
| Not Implemented (gap) | 0 | 0% |

### 7.3 Key Implementation Evidence

| Evidence Type | Reference |
|--------------|-----------|
| Policies | POL-001 through POL-020 (20 policies) |
| Code Modules | vault.py, classification.py, compliance.py, audit_logger.py, gdpr.py, consent.py, pii_tokenizer.py, data_residency.py, retention.py, profiling.py, capabilities.py, attenuation.py, tenant_encryption.py |
| Compliance Documents | System Description (SD-001), ROPA (GDPR-001), Risk Register (REG-001), DPIA Template, DPA Template, Audit Report |
| Testing | 9,300+ unit tests, 290+ compliance-specific tests, quarterly audit cycle |
| CI/CD | .github/workflows/ci.yml (8-stage pipeline), .github/workflows/sbom.yml (CycloneDX) |

---

## 8. Partial Applicability Notes

Controls marked "Partial" are applicable in limited scope due to Questo's business model
as an SDK vendor rather than a SaaS operator:

1. **Controller controls (A.1.x)**: Questo is primarily a processor. Controller controls
   apply only to Questo's own limited data processing (employee data, website visitors,
   customer business contacts) and to the SDK features that customers use to implement
   controller obligations.

2. **Segregation of duties (A.3.3)**: Small team size limits full segregation. Mitigated
   by automated controls (CI/CD gates, branch protection, code review requirements).

3. **Physical controls (A.3.20-A.3.23)**: Questo does not operate server infrastructure.
   Physical controls apply only to developer workstations and office premises.

4. **Backup (A.3.31)**: Source code is backed up via distributed Git. Customer data
   backup is a customer responsibility documented in CUECs.

---

## 9. Gap Summary and Remediation Plan

All previously identified gaps from the SOC 2 audit (audit_report_2026-02-17.md) have
been resolved as of 2026-02-23. The following items require attention for full ISO 27701
certification readiness:

| Item | Priority | Description | Target Date |
|------|----------|-------------|-------------|
| EU Representative appointment | HIGH | GDPR Art. 27 requires EU representative before first EU customer. ROPA lists this as pending. | Before first EU customer |
| Formal DPO appointment | MEDIUM | CTO currently serves as acting DPO. Formal appointment with supervisory authority registration pending. | 2026-Q2 |
| Sub-processor register publication | MEDIUM | Formalize and publish sub-processor list (OpenAI, Google, Anthropic) on company website. | 2026-Q2 |
| Annual management review | LOW | First annual PIMS management review to be conducted. | 2026-Q3 |
| Internal audit program formalization | LOW | Transition from ad-hoc audits to scheduled ISO 27701 internal audit program. | 2026-Q3 |

---

## 10. Cross-Reference to Existing Certifications

| Framework | Status | Relevance to ISO 27701 |
|-----------|--------|----------------------|
| SOC 2 Type 1 | Audit completed 2026-02-17 (22 PASS, 0 open gaps) | CC controls map to A.3 shared security controls |
| GDPR | Implemented in SDK (7 DSAR rights, consent, profiling opt-out, data residency, retention) | Core regulatory driver for ISO 27701 processor controls |
| ISO 27001:2022 | Not yet certified | A.3 controls designed to align; certification planned |

---

## 11. Document Control

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-23 | CTO, Questo Ltd | Initial SoA for ISO/IEC 27701:2025 |

---

*This Statement of Applicability is a controlled document maintained under version control
alongside the corteX SDK source code. It is reviewed semi-annually or upon material change
to the PIMS scope, organizational role, or regulatory environment.*
