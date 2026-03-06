# Risk Register

**Document ID**: REG-001
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2026-08-16
**Owner**: CTO, Questo Ltd
**Approved By**: CEO, Questo Ltd
**Classification**: CONFIDENTIAL
**GAP Reference**: GAP-S07

---

## 1. Purpose

This Risk Register documents all identified risks to Questo Ltd and the corteX AI Agent SDK platform. It provides a centralized, living record of risks along with their likelihood, impact, scores, mitigations, ownership, and status. This document satisfies SOC 2 CC3 (Risk Assessment) requirements and supports GDPR Article 32 (security of processing) and Article 35 (Data Protection Impact Assessment) obligations.

---

## 2. Risk Scoring Methodology

### 2.1 Likelihood Scale

| Rating | Value | Description |
|--------|-------|-------------|
| Low | 1 | Unlikely to occur in the next 12 months; no history of occurrence |
| Medium | 2 | May occur once in the next 12 months; occasional industry occurrence |
| High | 3 | Likely to occur multiple times in the next 12 months; frequent industry occurrence |

### 2.2 Impact Scale

| Rating | Value | Description |
|--------|-------|-------------|
| Low | 1 | Minimal disruption; no data exposure; easily remediated within hours |
| Medium | 2 | Moderate disruption; limited data exposure possible; remediation within days |
| High | 3 | Significant disruption; customer data at risk; remediation within weeks; regulatory notification possible |
| Critical | 4 | Severe business impact; confirmed data breach; regulatory action likely; potential legal liability |

### 2.3 Risk Score Calculation

**Risk Score = Likelihood x Impact**

| Score Range | Risk Level | Treatment Requirement |
|-------------|------------|----------------------|
| 1-2 | LOW | Accept or monitor; review annually |
| 3-4 | MEDIUM | Mitigate within 90 days; assign owner |
| 6-8 | HIGH | Mitigate within 30 days; escalate to CTO |
| 9-12 | CRITICAL | Immediate action required; escalate to CEO; board notification |

### 2.4 Risk Treatment Options

| Option | Description | When to Apply |
|--------|-------------|---------------|
| Mitigate | Implement controls to reduce likelihood or impact | Default treatment for MEDIUM and above |
| Accept | Acknowledge risk without additional controls | Only for LOW risks; requires CTO sign-off |
| Transfer | Shift risk via insurance or contract | For risks outside Questo's direct control |
| Avoid | Eliminate the activity creating the risk | When risk exceeds acceptable threshold |

---

## 3. Risk Register

### 3.1 AI-Specific Risks

| Risk ID | Risk | Category | Likelihood | Impact | Risk Score | Mitigation | Owner | Status | CC Mapping |
|---------|------|----------|------------|--------|------------|------------|-------|--------|------------|
| R-AI-001 | Prompt injection bypasses safety controls, enabling unauthorized actions or data exfiltration through crafted inputs | AI Security | Medium (2) | High (3) | 6 - HIGH | SafetyPolicy with injection_protection enabled; pattern matching against known injection vectors; input validation on all user-provided content; continuous pattern library updates | CTO | Active - Mitigated | CC6.1, CC7.2 |
| R-AI-002 | Cross-tenant data leakage where one tenant's data is exposed to another through shared memory or state | AI Security | Low (1) | Critical (4) | 4 - MEDIUM | Session isolation enforced at runtime; tenant_id mandatory on every data access; no shared mutable state between tenants; architectural isolation verified by penetration testing | CTO | Active - Mitigated | CC6.1, CC6.3 |
| R-AI-003 | API keys exposed in application logs, error messages, or debugging output | AI Security | Medium (2) | Critical (4) | 8 - HIGH | KeyVault module for all key storage; detect_leak() scans all output paths; log sanitization enforced at the logging layer; secret scanning enabled on GitHub repositories | CTO | Active - Mitigated | CC6.1, CC6.7 |
| R-AI-004 | LLM provider trains on customer data submitted through API calls, violating data processing agreements | AI Security | Low (1) | High (3) | 3 - MEDIUM | Enterprise API tiers used exclusively (OpenAI, Gemini, Anthropic); Data Processing Agreements with explicit no-training clauses required from all LLM sub-processors; customer notification of sub-processor changes | CTO | Active - Mitigated | CC9.2 |
| R-AI-005 | Supply chain attack through compromised Python dependencies injecting malicious code into customer environments | AI Security | Medium (2) | High (3) | 6 - HIGH | All dependencies pinned to specific versions in requirements files; Software Bill of Materials (SBOM) maintained; automated vulnerability scanning via Dependabot; dependency review before version upgrades | CTO | Active - Mitigated | CC6.8, CC8.1 |
| R-AI-006 | Model output unreliable or hallucinated, causing customers to act on incorrect information from the AI agent | AI Reliability | Medium (2) | Medium (2) | 4 - MEDIUM | Goal tracking system verifies every agent step against the original objective; confidence scoring on all outputs; multi-step verification pipeline; customer documentation on output validation best practices | CTO | Active - Mitigated | PI-1.3 |
| R-AI-007 | Agent enters infinite loop or exhausts compute resources, causing service degradation or excessive costs for customers | AI Reliability | Medium (2) | Medium (2) | 4 - MEDIUM | State hashing detects repeated states; loop prevention triggers automatic termination; configurable maximum iteration limits; drift detection identifies off-goal execution | CTO | Active - Mitigated | CC7.1, CC7.2 |
| R-AI-008 | Unauthorized tool execution where the AI agent invokes tools or APIs that the tenant has not explicitly permitted | AI Security | Low (1) | High (3) | 3 - MEDIUM | ToolPolicy enforces explicit allowlisting; default_deny prevents any tool execution without explicit tenant authorization; per-tenant tool configuration via TenantConfig; audit logging of all tool invocations | CTO | Active - Mitigated | CC5.1, CC6.1 |
| R-AI-009 | Brain-inspired learning system leaks user behavioral patterns, enabling reconstruction of personal data from weight deltas | AI Privacy | Low (1) | High (3) | 3 - MEDIUM | Per-tenant weight isolation prevents cross-tenant pattern leakage; per-user weight deltas support GDPR Art. 17 erasure; weight deletion API removes individual user influence; configurable opt-in for all learning features | CTO | Active - Mitigated | CC6.7, GDPR Art.17 |
| R-AI-010 | License key bypass or forgery allows unauthorized use of enterprise SDK features without a valid license | AI Security | Low (1) | Medium (2) | 2 - LOW | Ed25519 cryptographic signature validation; hardware-bound license keys; license validation at SDK initialization; tamper detection on license files | CTO | Active - Mitigated | CC6.1 |

### 3.2 Operational Risks

| Risk ID | Risk | Category | Likelihood | Impact | Risk Score | Mitigation | Owner | Status | CC Mapping |
|---------|------|----------|------------|--------|------------|------------|-------|--------|------------|
| R-OPS-001 | Key employee departure results in critical knowledge loss for the corteX engine or brain-inspired architecture | Operational | Medium (2) | High (3) | 6 - HIGH | Comprehensive documentation (97 pages MkDocs); code comments and docstrings on all modules; cross-training within the team; bus factor mitigation through pair programming; documented architecture decisions in repository | CEO | Active - Mitigated | CC1.4, CC1.7 |
| R-OPS-002 | Service outage of critical development infrastructure (GitHub, CI/CD, cloud provider) halts SDK development or delivery | Operational | Medium (2) | Medium (2) | 4 - MEDIUM | Local Git mirrors of all repositories; CI/CD pipeline can be rebuilt from configuration as code; SDK distributed via PyPI (independent of Questo infrastructure); on-prem customer deployments unaffected by Questo outages | CTO | Active - Mitigated | CC7.5, CC9.1 |
| R-OPS-003 | Supply chain disruption to critical development tools or cloud services affects SDK availability | Operational | Low (1) | Medium (2) | 2 - LOW | Multi-provider strategy for LLM capabilities (OpenAI, Gemini, Anthropic); SDK operates independently once installed; no runtime dependency on Questo servers; PyPI package independently hosted | CTO | Active - Accepted | CC9.1 |
| R-OPS-004 | Accidental data deletion or corruption of source code, documentation, or customer configuration templates | Operational | Low (1) | High (3) | 3 - MEDIUM | Git version control with full history; GitHub repository backups; branch protection rules preventing force-push to main; automated CI/CD ensures all changes are tested before merge | CTO | Active - Mitigated | CC7.5, CC8.1 |
| R-OPS-005 | SDK release contains breaking changes that disrupt customer production environments | Operational | Medium (2) | High (3) | 6 - HIGH | Semantic versioning strictly enforced; comprehensive test suite (3,355+ unit tests); CI/CD pipeline blocks releases with failing tests; CHANGELOG documents all changes; rollback procedures documented for PyPI releases | CTO | Active - Mitigated | CC8.1 |
| R-OPS-006 | Inadequate incident response delays breach containment, increasing exposure window and regulatory liability | Operational | Medium (2) | High (3) | 6 - HIGH | Documented Incident Response Plan (POL-004); defined severity levels (P0-P3) with response time SLAs; annual tabletop exercises; GDPR-compliant 72-hour notification procedures; communication templates pre-prepared | CTO | Active - Mitigated | CC7.4 |
| R-OPS-007 | Regulatory changes (EU AI Act, GDPR amendments) require significant SDK modifications without sufficient lead time | Operational | Medium (2) | Medium (2) | 4 - MEDIUM | Active monitoring of EU AI Act timeline (high-risk requirements effective Aug 2026); ComplianceEngine framework designed for extensible compliance checks; modular architecture supports rapid feature addition; legal counsel engaged for regulatory tracking | CEO | Active - Mitigated | CC3.4 |

### 3.3 Fraud Risks

| Risk ID | Risk | Category | Likelihood | Impact | Risk Score | Mitigation | Owner | Status | CC Mapping |
|---------|------|----------|------------|--------|------------|------------|-------|--------|------------|
| R-FRD-001 | Insider threat where an employee or contractor exfiltrates source code, customer data, or cryptographic keys | Fraud | Low (1) | Critical (4) | 4 - MEDIUM | GitHub access controls with least privilege; branch protection and required PR reviews; audit logging on all repository access; NDA and confidentiality agreements signed by all personnel; background checks on all employees; code review requirements on all changes | CEO | Active - Mitigated | CC1.1, CC3.3, CC6.3 |
| R-FRD-002 | Social engineering attack (phishing, pretexting) targets employees to gain access to development systems or customer data | Fraud | Medium (2) | High (3) | 6 - HIGH | MFA required on all systems (GitHub, email, cloud); security awareness training with phishing simulation; email filtering and link scanning; incident reporting procedures documented and trained; clean desk policy to prevent physical social engineering | CTO | Active - Mitigated | CC1.5, CC3.3, CC6.2 |
| R-FRD-003 | Financial fraud through manipulation of licensing, billing, or vendor payment processes | Fraud | Low (1) | Medium (2) | 2 - LOW | Ed25519 license validation prevents license forgery; separation of duties for financial transactions; vendor payment approval requires dual authorization; regular financial reconciliation; contractual terms documented and version-controlled | CEO | Active - Mitigated | CC3.3, CC5.1 |

### 3.4 Third-Party Risks

| Risk ID | Risk | Category | Likelihood | Impact | Risk Score | Mitigation | Owner | Status | CC Mapping |
|---------|------|----------|------------|--------|------------|------------|-------|--------|------------|
| R-VND-001 | Vendor data breach at an LLM provider (OpenAI, Google, Anthropic) exposes customer conversation data processed through their APIs | Third-Party | Low (1) | Critical (4) | 4 - MEDIUM | DPAs with explicit breach notification clauses (24-hour notification requirement); enterprise API tiers with enhanced security; on-prem deployment option eliminates LLM provider dependency; vendor SOC 2 reports reviewed annually; customer-configurable LLM provider selection | CTO | Active - Mitigated | CC9.2 |
| R-VND-002 | LLM API provider experiences extended outage, preventing customer AI agents from functioning | Third-Party | Medium (2) | Medium (2) | 4 - MEDIUM | Provider-agnostic architecture supports failover between LLM providers; configurable fallback provider chains; on-prem model support eliminates cloud dependency; graceful degradation documented in SDK; customer documentation covers multi-provider configuration | CTO | Active - Mitigated | CC9.1, CC9.2 |
| R-VND-003 | GitHub experiences a security breach exposing corteX source code or CI/CD secrets | Third-Party | Low (1) | High (3) | 3 - MEDIUM | GitHub SOC 2 report reviewed annually; repository secrets encrypted at rest; minimal secrets stored in CI/CD; branch protection rules enforce review requirements; local repository mirrors maintained; GitHub Advanced Security features enabled | CTO | Active - Mitigated | CC9.2 |

### 3.5 Compliance and Legal Risks

| Risk ID | Risk | Category | Likelihood | Impact | Risk Score | Mitigation | Owner | Status | CC Mapping |
|---------|------|----------|------------|--------|------------|------------|-------|--------|------------|
| R-CMP-001 | GDPR non-compliance due to inadequate data subject rights implementation results in supervisory authority enforcement action | Compliance | Medium (2) | Critical (4) | 8 - HIGH | DSAR API endpoints planned for data export, rectification, erasure, and restriction; ROPA maintained and reviewed semi-annually; DPA template covers all Article 28 requirements; breach notification procedures meet 72-hour requirement; PII detection in processing pipeline | CEO | Active - Mitigating | CC3.4, GDPR Arts. 15-22 |
| R-CMP-002 | EU AI Act non-compliance when customer deploys corteX in high-risk domain without adequate conformity assessment | Compliance | Medium (2) | High (3) | 6 - HIGH | SDK documentation covers AI Act obligations for customers; ComplianceEngine supports framework-specific pre-checks; DPIA template provided for customer use; high-risk deployment guidance in enterprise documentation; timeline tracking for Aug 2026 enforcement date | CEO | Active - Mitigating | CC3.4 |
| R-CMP-003 | Cross-border data transfer violation when customer data is processed in jurisdiction without adequate protection mechanism | Compliance | Low (1) | High (3) | 3 - MEDIUM | On-prem deployment eliminates transfer concerns; EU region selection available for cloud LLM providers (Google Vertex); EU-US Data Privacy Framework (DPF) plus SCCs as backup for US providers; per-tenant data residency configuration; Transfer Impact Assessments documented per provider | CEO | Active - Mitigated | CC9.2, GDPR Arts. 44-49 |

---

## 4. Risk Heat Map

```
                    IMPACT
            Low(1)  Medium(2)  High(3)  Critical(4)
           +--------+----------+---------+-----------+
High(3)    |        |          |         |           |
           | MEDIUM |   HIGH   |CRITICAL | CRITICAL  |
           +--------+----------+---------+-----------+
Medium(2)  |        | R-AI-006 | R-AI-001| R-AI-003  |
           |  LOW   | R-AI-007 | R-AI-005| R-CMP-001 |
           |        | R-OPS-002| R-OPS-001|          |
           |        | R-OPS-007| R-OPS-005|          |
           |        | R-VND-002| R-OPS-006|          |
           |        |          | R-FRD-002|          |
           |        |          | R-CMP-002|          |
           +--------+----------+---------+-----------+
Low(1)     |        | R-AI-010 | R-AI-002| R-FRD-001 |
           |  LOW   | R-OPS-003| R-AI-004| R-VND-001 |
           |        | R-FRD-003| R-AI-008|           |
           |        |          | R-AI-009|           |
           |        |          | R-OPS-004|          |
           |        |          | R-VND-003|          |
           |        |          | R-CMP-003|          |
           +--------+----------+---------+-----------+
  L
  I
  K
  E
  L
  I
  H
  O
  O
  D
```

---

## 5. Risk Summary by Level

| Risk Level | Count | Risk IDs |
|------------|-------|----------|
| CRITICAL | 0 | None |
| HIGH | 7 | R-AI-001, R-AI-003, R-AI-005, R-OPS-001, R-OPS-005, R-OPS-006, R-FRD-002, R-CMP-001, R-CMP-002 |
| MEDIUM | 14 | R-AI-002, R-AI-004, R-AI-006, R-AI-007, R-AI-008, R-AI-009, R-OPS-002, R-OPS-004, R-OPS-007, R-FRD-001, R-VND-001, R-VND-002, R-VND-003, R-CMP-003 |
| LOW | 3 | R-AI-010, R-OPS-003, R-FRD-003 |

---

## 6. Review and Update Schedule

| Activity | Frequency | Responsible | Next Due |
|----------|-----------|-------------|----------|
| Full risk register review | Semi-annually | CTO | 2026-08-16 |
| AI-specific risk review | Quarterly | CTO | 2026-05-16 |
| Third-party vendor risk review | Annually | CTO | 2027-02-16 |
| Fraud risk assessment | Annually | CEO | 2027-02-16 |
| New risk identification session | Quarterly | CTO + Engineering | 2026-05-16 |
| Post-incident risk update | After every P0/P1 incident | CTO | As needed |
| Regulatory change impact review | As regulations are published | CEO | As needed |

---

## 7. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO, Questo Ltd | Initial risk register with 24 entries across 5 categories |
