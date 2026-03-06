# Gap Analysis: ISO 27701:2025 and SOC 2 Type I

**Document ID**: GAP-ANALYSIS-001
**Version**: 1.0
**Date**: 2026-02-23
**Author**: Compliance Review (Automated + Manual)
**Classification**: INTERNAL - FOR CERTIFICATION PLANNING ONLY
**Owner**: CTO, Questo Ltd
**Scope**: corteX AI Agent SDK (cortex-ai) and Questo Ltd organizational controls

---

## Executive Summary

This gap analysis evaluates Questo Ltd's readiness for two certifications:

1. **ISO 27701:2025** - Privacy Information Management System (PIMS), focusing on PII Processor requirements (Annex A.2)
2. **SOC 2 Type I** - Trust Service Criteria across Security (CC1-CC9), Confidentiality (C1), Privacy (P1-P8), and Processing Integrity (PI1)

**Current State Summary:**
- 20 formal compliance policies (POL-001 through POL-020)
- 22 PASS controls in internal SOC 2 code audit (all 10 gaps resolved)
- 9,300+ unit tests, 100% passing
- Comprehensive GDPR documentation (DPA, ROPA, Privacy Policy, DPIA Template, Sub-Processor List)
- 6 GitHub Actions workflows (CI, SBOM, CodeQL, PyPI Publish, Release Drafter, Dependabot)
- Risk register with 24 entries across 5 categories
- Mature security codebase (KeyVault, DataClassifier, ComplianceEngine, AuditLogger, PII Tokenizer)

**Honest Assessment:**
The SDK-level controls are strong. The primary gaps are organizational and operational - Questo is a single-founder startup with no live production infrastructure, no customer deployments yet, and no operational history to evidence. Both certifications require more than code; they require demonstrated organizational maturity, tested procedures, and operational evidence over time.

**Gap Counts:**

| Category | Critical | High | Medium | Low | Total |
|----------|----------|------|--------|-----|-------|
| ISO 27701:2025 | 2 | 5 | 8 | 3 | 18 |
| SOC 2 Type I | 1 | 6 | 9 | 4 | 20 |
| Shared (counted in both) | 1 | 4 | 6 | 2 | 13 |
| **Net unique gaps** | **2** | **7** | **11** | **5** | **25** |

---

## Part 1: ISO 27701:2025 Gap Analysis

ISO 27701:2025 is a stand-alone Privacy Information Management System (PIMS) standard. Questo Ltd would certify as a **PII Processor** (Annex A.2 controls) since the SDK processes personal data on behalf of customer Data Controllers.

### 1.1 Privacy Governance (Clauses 4-7 / Management System)

| Req ID | Requirement | What We Have | What Is Missing | Priority | Effort |
|--------|-------------|-------------|-----------------|----------|--------|
| ISO-001 | Formal PIMS scope statement defining organizational context, interested parties, and boundaries | System description (SD-001) defines SOC 2 scope. Privacy Policy identifies B2B2C chain. | No formal PIMS scope document. The system description is SOC 2-focused, not structured as an ISO management system scope statement. Need to define the PIMS scope, organizational context (Clause 4), and interested parties explicitly. | High | 2-3 days |
| ISO-002 | Privacy leadership and commitment - top management demonstrating accountability | CTO acts as DPO (noted in ROPA). CEO approves risk register. | No formal management review meeting minutes. No documented evidence of top management privacy commitment (e.g., signed privacy commitment statement, management review records). Single founder means the roles of CEO, CTO, and DPO are conflated - need documented role separation or justification. | Medium | 1-2 days |
| ISO-003 | Privacy policy as a standalone document (distinct from information security policy) | Privacy Policy (GDPR-PP-001) exists. POL-001 is an Information Security Policy. | No standalone "Privacy Policy for the PIMS" as required by ISO 27701. The existing Privacy Policy is customer/user-facing (GDPR Art. 13-14), not an internal organizational privacy policy that establishes the PIMS framework, objectives, and management commitment. These are different documents. | High | 2-3 days |
| ISO-004 | Privacy risk assessment methodology specific to PII processing | POL-005 Risk Assessment Policy exists. Risk register (REG-001) has 24 entries including privacy risks (R-AI-009, R-CMP-001, R-CMP-003). | The risk assessment methodology does not explicitly incorporate privacy impact as a separate dimension. The risk register treats privacy risks alongside security risks but does not use a PII-specific risk framework (e.g., likelihood of privacy harm to data subjects, types of privacy impact). No formal Privacy Impact Assessment (PIA) has been conducted for the SDK itself - only a DPIA template for customers. | Medium | 3-5 days |
| ISO-005 | Internal audit program for the PIMS | No internal audit program exists beyond the automated code audit (audit_report_2026-02-17.md). | No formal internal audit schedule, audit procedures, auditor independence requirements, or audit findings tracking system. The existing "audit" was an automated code review, not a management system audit. ISO 27701 requires periodic internal audits of the PIMS by competent, independent auditors. | Critical | 5-10 days to design; ongoing to execute |
| ISO-006 | Management review of the PIMS | No management review has been conducted. | No evidence of management review meetings covering: PIMS performance, audit results, privacy incident trends, PII complaints, corrective actions status, and improvement opportunities. Must be conducted at planned intervals. | High | 1-2 days per review; establish quarterly cadence |
| ISO-007 | Documented competence requirements for privacy roles | Security training materials exist (security_training.md). | No documented competence requirements for personnel handling PII. No training records. No evidence of privacy-specific training (as distinct from security training). Single-founder means the "training program" currently has no participants other than the founder. | Medium | 2-3 days |

### 1.2 PII Processing Conditions (Annex A.2 - Processor Controls)

| Req ID | Requirement | What We Have | What Is Missing | Priority | Effort |
|--------|-------------|-------------|-----------------|----------|--------|
| ISO-008 | Processing only on documented instructions from the controller | DPA template (GDPR-DPA-001) Section 2 establishes processing scope. TenantConfig allows per-tenant configuration. SDK processes data only as configured. | DPA is a template with placeholder fields - no DPAs have been executed with actual customers. The DPA template is comprehensive but untested in practice. Need at least one executed DPA to demonstrate the control works. | High | Depends on customer acquisition |
| ISO-009 | Sub-processor management - written authorization, due diligence, contracts | Sub-Processor List (GDPR-SUB-001) documents OpenAI, Google, Anthropic. POL-009 Vendor Management Policy exists. | **No formal vendor DPAs have been executed with OpenAI, Google, or Anthropic.** The sub-processor list claims "Active DPA in place" for all three, but these are based on the providers' standard API terms, not negotiated bilateral DPAs. No vendor due diligence assessments have been conducted. No evidence of reviewing vendor SOC 2 reports. This is a significant gap for a processor claiming GDPR compliance. | Critical | 10-20 days (legal review + negotiations) |
| ISO-010 | Assistance to controller with data subject requests | GDPRManager implements all 7 DSAR rights (Art. 15-22). DSAR API is fully coded and tested. | Implementation is in code but no operational procedure exists for handling DSAR requests that come through human channels (email to privacy@questo.co). No SLA for DSAR response time. No DSAR tracking register. | Medium | 2-3 days |
| ISO-011 | PII breach notification to controller | POL-004 Incident Response Plan defines breach notification. DPA template Section specifies 72-hour notification. | No breach notification template prepared. No tested communication channel to controllers. No breach register. No incident response drill has ever been conducted to test the notification process. | High | 3-5 days to prepare templates + 1 day for tabletop drill |

### 1.3 PII Controller/Processor Obligations

| Req ID | Requirement | What We Have | What Is Missing | Priority | Effort |
|--------|-------------|-------------|-----------------|----------|--------|
| ISO-012 | Records of processing activities (ROPA) maintained and current | ROPA (GDPR-001) exists with 8 processing activity categories documented in detail. | ROPA Controller Registry is entirely placeholder (all entries are "[Customer Name 1]", "[Contact Details]"). ROPA has never been populated with real data. EU Representative field says "To be appointed before first EU customer engagement" - still not appointed. Company registration number is placeholder. | Medium | 1 day (populate once customers exist) |
| ISO-013 | Cross-border transfer mechanisms documented and implemented | Data residency enforcement in code (data_residency.py). Sub-processor list documents transfer mechanisms (DPF, SCCs). Privacy Policy references Israel adequacy decision. | No Standard Contractual Clauses (SCCs) have been executed. No Transfer Impact Assessments (TIAs) have been conducted for real data flows. The sub-processor list claims TIAs exist for Anthropic but no TIA document is in the repository. | Medium | 5-10 days (legal) |
| ISO-014 | Privacy by design and by default | SDK implements privacy by design: PII auto-detection, data classification, consent management, default-deny policies, PII tokenization. | No formal "privacy by design" assessment document. No documented methodology for evaluating new features against privacy principles. While the code embodies privacy by design, the process for maintaining it is not formalized. | Low | 2-3 days |
| ISO-015 | PII de-identification and anonymization capabilities | PIITokenizer provides reversible pseudonymization. DataClassifier detects PII. | No true anonymization capability (irreversible). Pseudonymization via PIITokenizer is reversible by design (tokens map back to original values). For some use cases, true anonymization may be required. This is a design choice, not a gap per se, but should be documented. | Low | Documentation only |

### 1.4 Security Controls for PII (Annex A.3)

| Req ID | Requirement | What We Have | What Is Missing | Priority | Effort |
|--------|-------------|-------------|-----------------|----------|--------|
| ISO-016 | Cryptographic controls appropriate for PII protection | KeyVault (Fernet/AES-128-CBC + HMAC-SHA256), PBKDF2 key derivation, per-tenant HKDF, TLS for data in transit. | Encryption is AES-128-CBC (Fernet), not AES-256. While adequate, some enterprise customers may require AES-256. No certificate management process. No crypto key lifecycle management beyond rotation. | Low | 5-10 days if AES-256 migration needed |
| ISO-017 | Incident management specific to PII breaches | POL-004 Incident Response Plan includes breach classification and notification. | No PII-specific incident classification (distinguishing PII breaches from general security incidents). No breach simulation drills conducted. No breach register maintained. The incident response plan has never been tested. | Medium | 3-5 days |
| ISO-018 | Supplier relationships and PII protection in supply chain | POL-009 Vendor Management Policy. Sub-Processor List. | No vendor risk assessments conducted. No evidence of reviewing LLM provider security practices. No contractual audit rights established. No vendor security questionnaires completed. | Medium | 5-10 days per vendor |

---

## Part 2: SOC 2 Type I Gap Analysis

SOC 2 Type I evaluates the suitability of the design of controls at a specific point in time. The scope includes Security (mandatory), plus Confidentiality, Privacy, and Processing Integrity.

### 2.1 CC1 - Control Environment

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-001 | CC1.1 - COSO Principle 1: Demonstrates commitment to integrity and ethical values | POL-013 Code of Conduct. POL-010 Acceptable Use Policy. Code review process documented. | No evidence of Code of Conduct acknowledgment by personnel. No ethics hotline or reporting mechanism (single founder, so less relevant, but should be documented). No documented disciplinary procedures for policy violations. | Medium | 1-2 days |
| SOC-002 | CC1.2 - Board exercises oversight of internal controls | Single founder company - no board of directors or advisory board. | **No board or governance body exists.** For a single-founder startup, this is expected but must be addressed in the system description. Document compensating controls: all decisions documented in version control, external advisors consulted for major decisions, planned board formation at funding stage. | Medium | 1 day (document compensating controls) |
| SOC-003 | CC1.3 - Management establishes structure, authority, and responsibility | Role descriptions exist in system description and code review process. | No formal organizational chart. No documented reporting lines. Single founder holds all roles (CEO, CTO, DPO, Security Engineer). Need to formally document this and explain compensating controls. | Medium | 1 day |
| SOC-004 | CC1.4 - Commitment to attract, develop, and retain competent individuals | Security training materials exist. | No formal hiring procedures documented. No onboarding/offboarding process (critical when the company grows). No personnel files or competency records. Single founder means there is literally no segregation of duties. | High | 3-5 days |
| SOC-005 | CC1.5 - Holds individuals accountable | Code review process requires approval before merge. CI pipeline enforces quality gates. | No formal performance management or accountability framework. No documented consequences for policy violations. No signed policy acknowledgments. | Low | 1-2 days |

### 2.2 CC2 - Communication and Information

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-006 | CC2.1 - Obtains and uses relevant quality information | 97+ pages of documentation. CLAUDE.md project guidelines. Comprehensive code comments. | No formal information quality process. No customer communication channels established (no support desk, no status page, no security advisory distribution list). | Medium | 2-3 days |
| SOC-007 | CC2.2 - Communicates internally about internal control objectives | Documentation exists in repository. All policies in docs/compliance/. | No evidence of internal communication about controls (meeting minutes, training records, policy distribution logs). Single founder means internal communication is trivially satisfied, but should be documented. | Low | 1 day |
| SOC-008 | CC2.3 - Communicates with external parties | Privacy Policy is public. Sub-Processor List is public. DPA template exists. CUECs documented in system description. | No customer-facing security documentation portal. No process for communicating security incidents to customers. No security advisory process tested. No public status page. | Medium | 3-5 days |

### 2.3 CC3 - Risk Assessment

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-009 | CC3.1 - Specifies suitable objectives | Risk register defines objectives. Compliance policies define security objectives. | Objectives are implicitly defined across multiple documents but not consolidated into a formal "control objectives" document. | Low | 1-2 days |
| SOC-010 | CC3.2 - Identifies and analyzes risks | Risk register (REG-001) with 24 entries, 5 categories, 4-level scoring. AI-specific risks, operational, fraud, vendor, compliance. | Risk register has never been formally reviewed or updated since creation (2026-02-16). No evidence of risk identification sessions. No risk assessment for the development environment specifically. | Medium | 1-2 days |
| SOC-011 | CC3.3 - Considers potential for fraud | Risk register includes R-FRD-001 (insider threat), R-FRD-002 (social engineering), R-FRD-003 (financial fraud). | No fraud risk assessment specific to the development pipeline (e.g., malicious code injection, package supply chain attacks via compromised developer machine). Single founder reduces insider threat but creates key-person risk. | Medium | 1-2 days |
| SOC-012 | CC3.4 - Identifies and assesses significant changes | Risk register includes R-OPS-007 (regulatory changes). | No formal change impact assessment process. No documented process for evaluating how external changes (new regulations, vendor changes, market changes) affect the control environment. | Low | 1-2 days |

### 2.4 CC4 - Monitoring Activities

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-013 | CC4.1 - Selects, develops, and performs ongoing/separate evaluations | CI pipeline runs on every push. CodeQL security scanning. Dependabot vulnerability monitoring. Internal audit report conducted (2026-02-17). | No ongoing monitoring program defined. No metrics dashboard for control effectiveness. No scheduled control assessments beyond the audit report. The single internal audit was AI-assisted, not conducted by an independent party. | High | 5-10 days |
| SOC-014 | CC4.2 - Evaluates and communicates deficiencies | Audit report documents 10 gaps (all resolved). Gap tracking via commit history. | No formal deficiency tracking system (just markdown files and git). No process for escalating deficiencies to management. No corrective action tracking with deadlines and verification. | Medium | 2-3 days |

### 2.5 CC5 - Control Activities

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-015 | CC5.1 - Selects and develops control activities | 22 verified controls (PASS in audit). ComplianceEngine with multi-framework enforcement. SafetyPolicy, DataClassifier, AuditLogger. | Controls are primarily code-based. No process-level controls matrix mapping controls to risks. No formal control testing schedule. | Medium | 2-3 days |
| SOC-016 | CC5.2 - Selects and develops general controls over technology | CI pipeline (ci.yml). CodeQL scanning. SBOM generation. Branch protection. | **No penetration testing has been performed.** No vulnerability scanning of the SDK package itself (beyond CodeQL static analysis). No security code review by an external party. | High | 10-20 days (external pentest) |
| SOC-017 | CC5.3 - Deploys through policies and procedures | 20 policies (POL-001 through POL-020). 2 procedures (code review, release process). | Policies reference roles that do not exist yet (Security Reviewer, QA Lead, Release Engineer). Policies describe processes for a team of 5+ but the company has 1 person. This creates a credibility gap with auditors. | High | 3-5 days to revise policies for current reality |

### 2.6 CC6 - Logical and Physical Access Controls

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-018 | CC6.1 - Implements logical access security | GitHub repository access via SSO. Branch protection rules. KeyVault encryption. CapabilitySet permissions. Default-deny ToolPolicy and ModelPolicy. | No formal access review process. No access provisioning/deprovisioning procedure. No MFA verification evidence for GitHub. No privileged access management documented. | Medium | 2-3 days |
| SOC-019 | CC6.2 - Authentication mechanisms | Ed25519 license validation. Per-tenant KeyVault derivation. | No formal authentication policy for developer access. GitHub MFA status not evidenced. No password policy enforcement evidence (POL-012 exists but no evidence of compliance). PyPI publishing credentials management not documented. | Medium | 1-2 days |
| SOC-020 | CC6.3 - Authorization mechanisms | Capability-based access control. ToolPolicy/ModelPolicy allowlisting. | No formal role-based access control (RBAC) for the development environment. No documented access matrix showing who has access to what. | Medium | 1-2 days |
| SOC-021 | CC6.6 - Restricts access to system components | Data classification gate blocks INTERNAL+ from cloud LLMs. CapabilitySet attenuation for sub-agents. | No network segmentation documentation for the development environment. No documentation of what services/ports are exposed on developer machines. | Low | 1 day |

### 2.7 CC7 - System Operations

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-022 | CC7.1 - Detects and monitors security events | AuditLogger captures 16 event types. CI pipeline monitors test results. Dependabot monitors dependencies. | No runtime monitoring (SDK runs on customer infrastructure, so this is a CUEC). No alerting mechanism for security events in the development environment. No log aggregation or SIEM for development infrastructure. | Medium | 3-5 days |
| SOC-023 | CC7.2 - Monitors system components for anomalies | AuditLogger with hash-chain integrity verification. CodeQL scanning. | No anomaly detection on the development environment. No file integrity monitoring on source code beyond git. No monitoring of GitHub audit logs for suspicious access. | Medium | 2-3 days |
| SOC-024 | CC7.3 - Evaluates security events | POL-004 Incident Response Plan with P0-P3 severity classification. | **No incident response drill has ever been conducted.** No incident register (no incidents have occurred, but the register should exist). No evidence that the incident response plan has been tested or validated. | High | 1-2 days for tabletop exercise |
| SOC-025 | CC7.4 - Responds to security incidents | POL-004 defines response procedures and SLAs. | No incident response team identified (single founder). No escalation contacts. No communication templates tested. No post-incident review process tested. | High | 2-3 days |

### 2.8 CC8 - Change Management

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-026 | CC8.1 - Manages changes | POL-003 Change Management Policy. GitHub branch protection. CI pipeline. Code review process (PROC-002). Release process (PROC-001). Semantic versioning. | **Code review process requires a reviewer, but single founder cannot review their own code.** This is a fundamental segregation of duties issue. Currently the founder merges their own PRs after CI passes. Need compensating controls (e.g., AI-assisted review, delayed self-review, external reviewer for security-critical changes). | Critical | Ongoing (requires process change) |

### 2.9 CC9 - Risk Mitigation

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-027 | CC9.1 - Identifies and assesses vendor/partner risk | POL-009 Vendor Management Policy. Sub-Processor List. Risk register includes vendor risks. | **No vendor risk assessments have been conducted.** No vendor SOC 2 reports reviewed. No vendor questionnaires completed. No evidence of due diligence on OpenAI, Google, or Anthropic beyond reading their websites. | High | 5-10 days |

### 2.10 Confidentiality Criteria

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-028 | C1.1 - Identifies and maintains confidential information | DataClassifier with 4-level classification (PUBLIC, INTERNAL, CONFIDENTIAL, RESTRICTED). PII detection (email, phone, SSN, credit card, Israeli ID). Secret detection (API keys, tokens). | No formal data inventory for the development environment. No classification applied to internal Questo documents (policies are marked INTERNAL/CONFIDENTIAL but no enforcement). No data flow inventory for development processes. | Medium | 2-3 days |
| SOC-029 | C1.2 - Disposes of confidential information | KeyVault.destroy() for cryptographic erasure. Configurable data retention. Audit log rotation. GDPR Art. 17 erasure support. | No formal data disposal procedure for development assets (old developer machines, removed code branches, deprecated documentation). No evidence of secure deletion practices. | Low | 1-2 days |

### 2.11 Privacy Criteria

| Req ID | Criterion | What We Have | What Is Missing | Priority | Effort |
|--------|-----------|-------------|-----------------|----------|--------|
| SOC-030 | P1 - Notice: Provides notice about privacy practices | Privacy Policy (GDPR-PP-001) published. DPA template with processing descriptions. ROPA documents processing activities. | Privacy notice not yet published at https://docs.cortex-ai.com/privacy (URL referenced in policy but site may not be live). No in-SDK privacy notice mechanism. | Medium | 1-2 days |
| SOC-031 | P2 - Choice and consent | ConsentManager with full lifecycle. ProfilingManager with opt-out. Per-purpose consent recording. | Consent mechanisms are in code but no operational consent collection exists for Questo's own processing. No customer-facing consent mechanism for data Questo processes as controller (website analytics, account data). | Medium | 2-3 days |
| SOC-032 | P3 - Collection limitation | TenantConfig controls what data is retained. DataRetention modes (NONE, SESSION, PERSISTENT, AUDIT_ONLY). | No documented data minimization assessment. No review of what data the SDK collects by default vs. what is strictly necessary. | Low | 1-2 days |
| SOC-033 | P4 - Use, retention, and disposal | RetentionEnforcer with compliance presets. Configurable TTLs. Dry-run support. | Retention policy enforcement has not been tested in a live environment. No evidence of retention schedule compliance. | Low | 1-2 days |
| SOC-034 | P5 - Access: Data subjects can access and correct information | GDPRManager.export_user_data() (Art. 15). GDPRManager.rectify_user_data() (Art. 16). GDPRManager.export_portable_data() (Art. 20). | No customer-facing DSAR portal or process. No documented SLA for responding to data subject requests. No DSAR tracking mechanism. | Medium | 3-5 days |
| SOC-035 | P6 - Disclosure to third parties | Sub-Processor List documents all third-party disclosures. Controller determines which providers are active. ModelPolicy restricts provider selection. | No mechanism to notify controllers when sub-processor list changes (policy says 30-day notice but no notification system exists). | Medium | 1-2 days |
| SOC-036 | P7 - Security for privacy | KeyVault encryption. PII tokenization. Data classification gate. Capability-based access control. | Covered by Security criteria (CC1-CC9). No additional privacy-specific security gaps beyond what is identified above. | Low | N/A (addressed by CC controls) |
| SOC-037 | P8 - Quality of personal information | Goal tracking verifies processing accuracy. SafetyPolicy validates inputs/outputs. | No data quality monitoring for PII specifically. No mechanism to identify stale or inaccurate PII in memory stores. | Low | 1-2 days |

---

## Part 3: Shared Gaps

These gaps affect BOTH ISO 27701:2025 and SOC 2 Type I certification readiness. Addressing them provides the highest return on investment.

### 3.1 Critical Shared Gaps

| Gap ID | Description | ISO 27701 Ref | SOC 2 Ref | Impact | Effort |
|--------|-------------|---------------|-----------|--------|--------|
| SHARED-001 | **Segregation of duties: Single founder holds all roles.** CEO = CTO = DPO = Security Engineer = Release Engineer. No independent code review possible. No independent audit possible. No separation between development, operations, and security. This is the single most significant gap for both certifications. | Clause 5.3 (roles), A.2 (independence) | CC1.3, CC1.4, CC5.2, CC8.1 | Both auditors will flag this. Must document compensating controls and a plan for when the team grows. | Ongoing - 3-5 days to document compensating controls |

### 3.2 High Shared Gaps

| Gap ID | Description | ISO 27701 Ref | SOC 2 Ref | Impact | Effort |
|--------|-------------|---------------|-----------|--------|--------|
| SHARED-002 | **No vendor DPAs executed with LLM providers.** The sub-processor list claims DPAs are "in place" but these are the providers' standard API terms, not negotiated agreements. No vendor due diligence conducted. | A.2 (sub-processor management) | CC9.1, P6 | Auditors will ask to see executed DPAs and due diligence records. | 10-20 days |
| SHARED-003 | **No penetration testing performed.** No external security assessment of the SDK. CodeQL and unit tests provide static analysis but not dynamic security testing. | A.3 (security controls) | CC5.2, CC7.1 | Pentest report is standard evidence for both certifications. | 10-20 days + cost |
| SHARED-004 | **No incident response drill conducted.** Incident response plan exists but has never been tested. No tabletop exercise. No breach simulation. | A.2 (breach notification) | CC7.3, CC7.4 | Auditors will ask "when was the last time you tested your IR plan?" | 1-2 days |
| SHARED-005 | **Policies describe a team that does not exist.** POL-001 through POL-020 reference roles like Security Reviewer, QA Lead, Release Engineer, Documentation Engineer - but only 1 person exists. This creates a credibility gap with auditors. | Clause 5 (documented info) | CC1.3, CC5.3 | Policies should reflect reality, not aspirations. | 3-5 days to revise |

### 3.3 Medium Shared Gaps

| Gap ID | Description | ISO 27701 Ref | SOC 2 Ref | Impact | Effort |
|--------|-------------|---------------|-----------|--------|--------|
| SHARED-006 | **No formal internal audit program.** The automated code audit is valuable but does not constitute a management system audit. No auditor independence, no audit schedule, no findings tracking. | Clause 6.5 (internal audit) | CC4.1 | Both standards require internal audits. | 5-10 days to design |
| SHARED-007 | **No management review evidence.** No meeting minutes, no management review reports, no documented decisions on PIMS/security improvements. | Clause 6.6 (management review) | CC4.1, CC4.2 | Needs quarterly reviews, even if the "meeting" is the founder reviewing metrics and documenting decisions. | 1-2 days per review |
| SHARED-008 | **No formal onboarding/offboarding process.** Currently irrelevant (1 person) but will be critical when hiring. No documented access provisioning/deprovisioning procedures. | Clause 5.2 (competence) | CC1.4, CC6.1 | Document the process now even if unused. | 2-3 days |
| SHARED-009 | **ROPA and DPA contain placeholder data.** Controller registry is empty. Company registration numbers are "[placeholder]". EU Representative not appointed. | Art. 30(2) requirements | P1, P6 | Must be populated before engaging EU customers. | 1-2 days |
| SHARED-010 | **No business continuity testing.** POL-006 (BCP) and POL-007 (DRP) exist but have never been tested. No BCP drill, no DR test, no recovery time validated. | A.3 (business continuity) | CC7.5, CC9.1 | Auditors will ask for BCP test results. | 2-3 days for tabletop |
| SHARED-011 | **No formal training records.** Security training materials exist but no evidence anyone completed them. No training attendance records, no quiz results, no acknowledgment signatures. | Clause 5.2 (competence) | CC1.4, CC1.5 | Even single founder should document self-training. | 1-2 days |

### 3.4 Low Shared Gaps

| Gap ID | Description | ISO 27701 Ref | SOC 2 Ref | Impact | Effort |
|--------|-------------|---------------|-----------|--------|--------|
| SHARED-012 | **No physical security evidence.** POL-015 (Physical Security) exists but there is no evidence of physical security measures for the development environment. | A.3 (physical security) | CC6.4, CC6.5 | For a home-office/single-person setup, document the environment and compensating controls. | 1 day |
| SHARED-013 | **No formal document control system.** Policies are markdown files in git (which provides versioning) but there is no formal document control register, no approval workflow, no distribution tracking. | Clause 5.5 (documented info) | CC2.1 | Git provides version control, but a document register would improve auditability. | 1-2 days |

---

## Part 4: Remediation Roadmap

Ordered by priority and dependencies. Estimated timeline assumes a single person working part-time on compliance alongside development.

### Phase 0: Foundation (Weeks 1-2) - CRITICAL

These items must be addressed before engaging any auditor.

| Priority | Item | Effort | Dependencies | Addresses |
|----------|------|--------|-------------|-----------|
| 1 | **Document compensating controls for single-founder structure.** Write a formal "Organizational Structure and Compensating Controls" document explaining why certain controls are adapted for a single-person company, what compensating controls exist (version control history, AI-assisted review, CI automation), and what triggers the transition to full controls (first hire, funding). | 3 days | None | SHARED-001, SOC-002, SOC-003, SOC-004, SOC-017 |
| 2 | **Revise all 20 policies to reflect current reality.** Replace references to non-existent roles with the actual single-founder structure. Add "Compensating Control" sections where segregation of duties is not possible. Keep the aspirational team structure in an appendix for future reference. | 5 days | Item 1 | SHARED-005, SOC-017 |
| 3 | **Conduct and document first management review.** Create a management review template. Conduct a formal review covering: PIMS/ISMS performance, risk register status, policy compliance, training status, incident summary (none), audit findings, improvement actions. Document decisions. | 1 day | None | SHARED-007, ISO-006 |
| 4 | **Create internal audit program.** Define audit schedule (semi-annual), audit scope rotation, audit methodology, findings tracking template. Conduct first self-audit with documented independence measures (e.g., time separation between creating and auditing controls). | 5 days | None | SHARED-006, ISO-005 |

### Phase 1: Organizational Controls (Weeks 3-4) - HIGH

| Priority | Item | Effort | Dependencies | Addresses |
|----------|------|--------|-------------|-----------|
| 5 | **Create PIMS scope document.** Formal ISO 27701 scope statement defining organizational context, interested parties, PIMS boundaries, applicability of controls. | 2 days | None | ISO-001 |
| 6 | **Create internal privacy policy.** Distinct from the customer-facing Privacy Policy. This is the PIMS policy establishing privacy objectives, management commitment, framework for privacy risk management. | 2 days | Item 5 | ISO-003 |
| 7 | **Conduct incident response tabletop exercise.** Simulate a PII breach scenario. Walk through the IR plan. Document lessons learned. Update the IR plan if needed. Create a breach notification template. | 2 days | None | SHARED-004, ISO-011, SOC-024, SOC-025 |
| 8 | **Conduct business continuity tabletop exercise.** Simulate developer laptop loss, GitHub outage, PyPI compromise. Document recovery steps and times. | 1 day | None | SHARED-010 |
| 9 | **Create training records system.** Even for a single founder, document completion of security awareness training, privacy training, and policy acknowledgment. Create simple attestation forms. | 1 day | None | SHARED-011, ISO-007 |
| 10 | **Create and populate document control register.** List all policies, procedures, compliance documents with version, owner, approval date, next review date, distribution. | 1 day | None | SHARED-013 |

### Phase 2: External Dependencies (Weeks 5-8) - HIGH

These require external parties and may take longer.

| Priority | Item | Effort | Dependencies | Addresses |
|----------|------|--------|-------------|-----------|
| 11 | **Engage penetration testing firm.** Scope: SDK codebase security review, dependency analysis, crypto implementation review. Budget: $5K-$15K for a focused assessment. | 2-3 weeks | Budget approval | SHARED-003, SOC-016 |
| 12 | **Review vendor SOC 2 reports.** Request and review SOC 2 Type II reports from OpenAI, Google Cloud, and Anthropic. Document findings. | 3-5 days | Vendor cooperation | SHARED-002, SOC-027 |
| 13 | **Evaluate vendor DPA adequacy.** Review OpenAI, Google, and Anthropic standard DPA terms. Document whether they meet GDPR Art. 28 requirements. If gaps exist, document risk acceptance or pursue negotiated DPAs. For a startup, the standard API terms + enterprise tier may be sufficient with documented justification. | 5-10 days | Legal review | SHARED-002, ISO-009 |
| 14 | **Conduct Transfer Impact Assessments.** For each US-based LLM provider, document: data types transferred, transfer mechanism (DPF, SCCs), supplementary measures, risk assessment. | 3-5 days | Item 13 | ISO-013 |

### Phase 3: Operational Maturity (Weeks 9-12) - MEDIUM

| Priority | Item | Effort | Dependencies | Addresses |
|----------|------|--------|-------------|-----------|
| 15 | **Establish formal access review process.** Quarterly review of GitHub access, PyPI credentials, cloud service access. Document current access state. | 1 day initial + quarterly | None | SOC-018, SOC-019, SOC-020 |
| 16 | **Create customer communication infrastructure.** Security advisory distribution list, status page (even if minimal), DSAR intake process, breach notification templates. | 3-5 days | None | SOC-008, SOC-034, SOC-035, ISO-010 |
| 17 | **Conduct privacy risk assessment.** Use the DPIA template to assess the SDK's own processing. Document risks specific to PII (not just security risks). Update risk register with privacy-specific scoring. | 3-5 days | None | ISO-004 |
| 18 | **Create controls matrix.** Map every control to the risk it mitigates and the SOC 2/ISO 27701 requirement it satisfies. This becomes the master reference for auditors. | 3-5 days | All above | SOC-015, all |
| 19 | **Create onboarding/offboarding procedure.** Document the process even though it is currently unused. Include: access provisioning, training requirements, NDA signing, equipment setup, deprovisioning checklist. | 2 days | None | SHARED-008 |
| 20 | **Populate ROPA with real data.** Once company registration is complete, fill in all placeholder fields. Appoint EU Representative if targeting EU customers. | 1 day | Company registration | SHARED-009, ISO-012 |

### Phase 4: Pre-Audit Preparation (Weeks 13-16) - MEDIUM/LOW

| Priority | Item | Effort | Dependencies | Addresses |
|----------|------|--------|-------------|-----------|
| 21 | **Conduct second management review.** Review all remediation progress. Document remaining gaps and acceptance decisions. | 1 day | Phases 1-3 | SHARED-007 |
| 22 | **Conduct second internal audit.** Focused on areas remediated in Phases 1-3. Document findings and corrective actions. | 3-5 days | Phases 1-3 | SHARED-006 |
| 23 | **Prepare evidence package for auditor.** Organize all evidence by SOC 2 criterion and ISO 27701 clause. Create evidence index. Prepare management assertion letter. | 5-10 days | All above | All |
| 24 | **Select and engage external auditor.** For SOC 2: CPA firm. For ISO 27701: accredited certification body. Consider combined audits if possible. Budget: $15K-$40K for SOC 2 Type I, $10K-$25K for ISO 27701. | 2-4 weeks | Budget, all above | All |

---

## Part 5: Cost Estimates and Timeline

### Certification Costs (Estimated)

| Item | Low Estimate | High Estimate | Notes |
|------|-------------|---------------|-------|
| Penetration testing | $5,000 | $15,000 | Focused SDK security assessment |
| SOC 2 Type I audit (CPA firm) | $15,000 | $40,000 | Depends on scope, firm size |
| ISO 27701:2025 certification audit | $10,000 | $25,000 | Stage 1 + Stage 2 |
| Legal review (DPAs, TIAs) | $3,000 | $10,000 | Can be reduced with template-based approach |
| Compliance tooling (optional) | $0 | $5,000 | Vanta, Drata, or similar |
| **Total** | **$33,000** | **$95,000** | |

### Timeline to Certification Readiness

| Milestone | Target | Confidence |
|-----------|--------|------------|
| Phase 0 complete (foundation) | Week 2 | High |
| Phase 1 complete (organizational) | Week 4 | High |
| Phase 2 complete (external) | Week 8 | Medium (depends on vendor cooperation, pentest scheduling) |
| Phase 3 complete (operational) | Week 12 | Medium |
| Phase 4 complete (pre-audit) | Week 16 | Medium |
| SOC 2 Type I audit engagement | Week 17-20 | Low (depends on auditor availability and budget) |
| ISO 27701 certification audit | Week 20-24 | Low (depends on auditor availability and budget) |
| **Earliest certification possible** | **Week 24 (~6 months)** | Low |
| **Realistic certification target** | **Week 36 (~9 months)** | Medium |

### Key Decision Points

1. **Should we pursue both certifications simultaneously?** Combined preparation saves ~30% effort since many controls overlap. However, the cost is roughly additive. Recommendation: Start SOC 2 Type I first (more common customer requirement in US SaaS market), then add ISO 27701 once the ISMS foundation is established.

2. **Should we wait until we have customers?** Some gaps (ROPA population, DPA execution, DSAR procedures) require customers to fully evidence. SOC 2 Type I can be obtained with design-only evidence. ISO 27701 is more challenging without operational evidence. Recommendation: Begin preparation now but defer the actual audit engagement until there is at least one customer or a clear customer pipeline.

3. **Should we use a compliance automation tool (Vanta, Drata, Secureframe)?** These tools cost $3K-$15K/year but dramatically reduce evidence collection effort and provide continuous monitoring. Recommendation: Evaluate when preparing for the actual audit engagement. For now, the git-based documentation approach is adequate.

4. **Single-founder audit viability.** Some auditors may be reluctant to issue a SOC 2 Type I report for a single-person company due to the inherent segregation of duties limitations. Recommendation: Be transparent with potential auditors about the company structure. Some firms specialize in startup audits and are comfortable with compensating controls.

---

## Part 6: What We Do Well

For completeness, here is what will likely pass audit with minimal additional work:

| Area | Strength | Evidence |
|------|----------|---------|
| Encryption at rest | Fernet + PBKDF2 600K iterations, per-tenant HKDF key derivation | vault.py, tenant_encryption.py |
| Data classification | 4-level auto-classification with PII and secret detection, escalation-only | classification.py, router.py |
| Audit logging | Hash-chained, append-only, tamper-evident JSONL with retention enforcement | audit_logger.py, audit_storage.py |
| Access control | Capability-based, immutable, attenuatable, risk-based reduction | capabilities.py, attenuation.py |
| GDPR data subject rights | All 7 rights implemented and tested | gdpr.py (Art. 15-22) |
| Consent management | Full lifecycle with ComplianceEngine auto-wiring | consent.py, compliance.py |
| PII tokenization | Reversible pseudonymization integrated into LLM pipeline | pii_tokenizer.py, router.py |
| Data residency | Per-tenant region enforcement with provider mapping | data_residency.py |
| Change management tooling | CI pipeline, branch protection, SBOM, CodeQL, Dependabot | .github/workflows/ |
| Policy documentation | 20 comprehensive policies covering all major domains | docs/compliance/policies/ |
| Risk management | 24-entry risk register with scoring and CC mapping | risk_register.md |
| GDPR documentation | DPA, ROPA, Privacy Policy, DPIA template, Sub-Processor List | docs/compliance/gdpr/ |
| Test coverage | 9,300+ tests, 100% passing | tests/ |
| Zero-trust defaults | ToolPolicy and ModelPolicy default to deny-all | config.py |

---

## Appendix A: Control Mapping Matrix

### ISO 27701:2025 Annex A.2 (PII Processor) to corteX Controls

| ISO 27701 Control | Description | corteX Implementation | Gap Status |
|-------------------|-------------|----------------------|------------|
| A.2.1 | Process PII only on controller instructions | TenantConfig, DPA template | Partial (no executed DPA) |
| A.2.2 | Purposes of processing | ROPA Section 4 (8 activities) | Documented |
| A.2.3 | Marketing and advertising | Not applicable (SDK, not SaaS) | N/A |
| A.2.4 | Infringing instructions | ComplianceEngine pre-check | Implemented |
| A.2.5 | Customer obligations | CUECs in system description | Documented |
| A.2.6 | Records related to processing | ROPA, AuditLogger | Implemented (ROPA has placeholders) |
| A.2.7 | Sub-processor management | Sub-Processor List, POL-009 | Partial (no executed vendor DPAs) |
| A.2.8 | Transfer of PII | data_residency.py, SCCs reference | Partial (no TIAs conducted) |
| A.2.9 | Notification of PII disclosure | Not implemented | GAP |
| A.2.10 | Assist controller with data subject requests | GDPRManager (7 rights) | Implemented in code |
| A.2.11 | Conditions for PII return/disposal | DPA Section 12, retention.py | Documented + implemented |
| A.2.12 | Breach notification | POL-004, DPA notification clauses | Documented (not tested) |
| A.2.13 | Security of processing | KeyVault, DataClassifier, AuditLogger | Implemented |
| A.2.14 | Privacy by design and default | SDK architecture | Implemented (not formally documented) |
| A.2.15 | PII de-identification | PIITokenizer (pseudonymization) | Partial (no true anonymization) |
| A.2.16 | PII transmission controls | TLS enforcement, classification gate | Implemented |
| A.2.17 | Processor PII access controls | CapabilitySet, ToolPolicy, ModelPolicy | Implemented |
| A.2.18 | Records of authorized users | TenantConfig, AuditLogger | Implemented |

### SOC 2 TSC Summary Mapping

| TSC Category | Criteria Count | Fully Met | Partially Met | Not Met |
|-------------|---------------|-----------|---------------|---------|
| CC1 (Control Environment) | 5 | 1 | 3 | 1 |
| CC2 (Communication) | 3 | 1 | 2 | 0 |
| CC3 (Risk Assessment) | 4 | 2 | 2 | 0 |
| CC4 (Monitoring) | 2 | 0 | 2 | 0 |
| CC5 (Control Activities) | 3 | 1 | 2 | 0 |
| CC6 (Logical Access) | 4 | 1 | 3 | 0 |
| CC7 (System Operations) | 4 | 0 | 4 | 0 |
| CC8 (Change Management) | 1 | 0 | 1 | 0 |
| CC9 (Risk Mitigation) | 1 | 0 | 1 | 0 |
| C1 (Confidentiality) | 2 | 1 | 1 | 0 |
| P1-P8 (Privacy) | 8 | 3 | 5 | 0 |
| **Total** | **37** | **10** | **26** | **1** |

---

## Appendix B: References

### Standards and Frameworks
- ISO/IEC 27701:2025 - Privacy Information Management Systems - Requirements and Guidance
- AICPA Trust Services Criteria (2017, with 2022 revised points of focus)
- GDPR (EU) 2016/679
- ISO/IEC 27001:2022 - Information Security Management Systems

### Internal Documents Referenced
- SD-001: System Description (docs/compliance/system_description.md)
- REG-001: Risk Register (docs/compliance/risk_register.md)
- Audit Report 2026-02-17 (docs/compliance/audit_report_2026-02-17.md)
- POL-001 through POL-020 (docs/compliance/policies/)
- GDPR-001: ROPA (docs/compliance/ROPA.md)
- GDPR-DPA-001: DPA Template (docs/compliance/gdpr/DPA_Template.md)
- GDPR-PP-001: Privacy Policy (docs/compliance/gdpr/Privacy_Policy.md)
- GDPR-002: DPIA Template (docs/compliance/DPIA_Template.md)
- GDPR-SUB-001: Sub-Processor List (docs/compliance/gdpr/Sub_Processor_List.md)
- PROC-001: Release Process (docs/compliance/procedures/release_process.md)
- PROC-002: Code Review Process (docs/compliance/procedures/code_review_process.md)

---

*This gap analysis is an internal document for certification planning. It should be updated quarterly and after any significant organizational or technical change. Next review: 2026-05-23.*
