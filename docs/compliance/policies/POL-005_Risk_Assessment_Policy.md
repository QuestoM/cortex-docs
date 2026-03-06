# Risk Assessment and Mitigation Policy

**Document ID**: POL-005
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO
**Classification**: INTERNAL

---

## 1. Purpose

This Risk Assessment and Mitigation Policy establishes the methodology for identifying, analyzing, evaluating, treating, and monitoring risks to Questo Ltd and the corteX AI Agent SDK platform. It defines the risk scoring framework, risk register maintenance procedures, AI-specific risk categories, fraud risk assessment requirements, and third-party risk assessment processes. This policy ensures that Questo Ltd proactively manages risks to information security, business operations, and customer trust.

**SOC 2 Mapping**: CC3.1-CC3.4 (Risk Assessment), CC9.1 (Risk Mitigation)

## 2. Scope

This policy applies to:

- **Risk Categories**: All risks to the confidentiality, integrity, and availability of Questo Ltd information assets, business operations, and the corteX platform. This includes: technology risks, security risks, operational risks, compliance risks, AI-specific risks, vendor risks, and fraud risks.
- **Assets**: All information assets as defined in POL-001, including source code, customer data, API keys, infrastructure, intellectual property, and personnel.
- **Personnel**: All employees and contractors are responsible for identifying and reporting risks. The CTO and Security Lead are responsible for the formal risk assessment process.
- **Third Parties**: Vendors, sub-processors, LLM providers, and other third parties whose services or products affect the security posture of the corteX platform.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Risk** | The potential for loss or harm arising from a threat exploiting a vulnerability |
| **Threat** | Any circumstance or event with the potential to adversely impact an asset |
| **Vulnerability** | A weakness that can be exploited by a threat to cause harm |
| **Likelihood** | The estimated probability that a risk event will occur within the assessment period |
| **Impact** | The estimated severity of harm if a risk event occurs |
| **Risk Score** | The combination of likelihood and impact, expressed as Low/Medium/High/Critical |
| **Risk Appetite** | The level of risk that Questo Ltd is willing to accept in pursuit of its objectives |
| **Risk Treatment** | The action taken in response to a risk: mitigate, accept, transfer, or avoid |
| **Risk Register** | The authoritative record of identified risks, their assessments, treatments, and status |
| **Residual Risk** | The level of risk remaining after risk treatment controls are applied |
| **Inherent Risk** | The level of risk before any controls or mitigations are applied |
| **Fraud Risk** | Risks related to intentional acts of deception for personal gain or to cause harm |
| **AI-Specific Risk** | Risks unique to AI systems, including prompt injection, hallucination, data leakage, and model manipulation |

## 4. Roles and Responsibilities

### 4.1 CTO (Risk Owner)

- Owns this policy and serves as the senior risk officer for Questo Ltd
- Chairs the annual formal risk assessment and quarterly risk review meetings
- Approves risk treatment plans and risk acceptance decisions up to the defined threshold
- Escalates risks exceeding the CTO acceptance threshold to the CEO
- Ensures adequate resources are allocated for risk mitigation activities

### 4.2 CEO

- Approves risk acceptance decisions for Critical-rated risks
- Reviews the risk posture summary at least quarterly
- Sets the organizational risk appetite in consultation with the CTO
- Approves this policy and any substantive changes

### 4.3 Security Lead

- Executes the day-to-day risk management process
- Maintains the risk register, ensuring it is current and complete
- Conducts risk identification activities, including vulnerability assessments and threat analysis
- Prepares risk assessment reports for CTO review
- Coordinates vendor risk assessments
- Tracks risk treatment action items to completion

### 4.4 Engineering Team

- Identifies and reports risks discovered during development, code review, and operations
- Implements risk mitigation controls as assigned
- Participates in risk assessment workshops when requested
- Ensures that code changes are evaluated for risk impact per POL-003

### 4.5 All Personnel

- Report identified or suspected risks through the Security Lead
- Comply with risk mitigation controls relevant to their role
- Participate in risk awareness training as part of the security awareness program (POL-001)

## 5. Policy Statements

### 5.1 Risk Assessment Methodology

Questo Ltd uses a qualitative risk assessment methodology based on a likelihood-by-impact matrix. This approach provides a consistent, repeatable framework for evaluating and prioritizing risks.

#### 5.1.1 Likelihood Scale

| Rating | Description | Frequency Estimate |
|--------|-------------|-------------------|
| **High** | The threat is expected to materialize within the assessment period. Known active exploitation or strong preconditions exist. | More than once per year |
| **Medium** | The threat could reasonably materialize. Conditions exist that make exploitation plausible. | Once every 1-3 years |
| **Low** | The threat is unlikely to materialize. Significant barriers to exploitation exist. | Less than once every 3 years |

#### 5.1.2 Impact Scale

| Rating | Description | Criteria |
|--------|-------------|----------|
| **Critical** | Existential threat to Questo Ltd or catastrophic harm to customers. | Broad customer data breach, complete loss of source code, regulatory action resulting in inability to operate, irreparable reputational damage |
| **High** | Significant harm to Questo Ltd operations or customer trust. | Data breach affecting a limited number of customers, extended service outage (>24h), significant financial loss, major reputational damage |
| **Medium** | Moderate disruption to operations or limited harm to customers. | Minor data exposure, short service disruption (<4h), moderate financial impact, limited reputational impact |
| **Low** | Minimal impact, easily recoverable. | No data exposure, negligible operational impact, minimal financial impact, no reputational impact |

#### 5.1.3 Risk Scoring Matrix

| | Impact: Low | Impact: Medium | Impact: High | Impact: Critical |
|---|------------|---------------|-------------|-----------------|
| **Likelihood: High** | Medium | High | Critical | Critical |
| **Likelihood: Medium** | Low | Medium | High | Critical |
| **Likelihood: Low** | Low | Low | Medium | High |

#### 5.1.4 Risk Rating Definitions

| Rating | Description | Required Action | Timeframe |
|--------|-------------|-----------------|-----------|
| **Critical** | Unacceptable risk requiring immediate attention | Immediate mitigation or escalation. CEO must approve any acceptance. | Begin mitigation within 24 hours |
| **High** | Significant risk requiring prompt treatment | Develop and begin executing a treatment plan. CTO approval required for acceptance. | Treatment plan within 5 business days |
| **Medium** | Moderate risk requiring planned treatment | Include in the treatment plan with assigned owner and target date. | Treatment plan within 30 days |
| **Low** | Acceptable risk within risk appetite | Monitor during regular risk reviews. Document acceptance rationale. | Review at next quarterly assessment |

### 5.2 Risk Identification Process

Risk identification is a continuous activity supported by structured assessment events:

1. **Annual Formal Assessment**: A comprehensive risk assessment conducted annually, facilitated by the CTO and documented in the risk register. This assessment covers all risk categories in scope.
2. **Quarterly Risk Reviews**: The CTO chairs a quarterly review of the risk register, assessing changes in risk levels, progress on treatment plans, and newly identified risks.
3. **Change-Driven Assessment**: Significant changes to the corteX platform, infrastructure, or business operations trigger a risk reassessment per POL-003. The change author includes a risk impact analysis in the Pull Request.
4. **Continuous Identification**: All personnel are expected to identify and report risks as they are discovered. The Security Lead triages reported risks and adds them to the register.
5. **Threat Intelligence**: The Security Lead monitors relevant threat intelligence sources, including vulnerability databases (NVD), security advisories (GitHub, dependency tools), AI security research, and industry publications.

### 5.3 Risk Treatment

For each identified risk, one of four treatment options is selected:

| Treatment | Description | When Applied |
|-----------|-------------|-------------|
| **Mitigate** | Implement controls to reduce likelihood or impact | The risk is above the acceptable threshold and can be reduced to an acceptable level through controls |
| **Accept** | Acknowledge the risk and monitor it | The risk is within the risk appetite, or the cost of mitigation exceeds the potential impact |
| **Transfer** | Shift the risk to a third party | The risk can be effectively managed through insurance, contractual arrangements, or outsourcing |
| **Avoid** | Eliminate the risk by removing the risk source | The risk is unacceptable and cannot be adequately mitigated |

#### 5.3.1 Risk Acceptance Criteria

| Risk Rating | Acceptance Authority |
|------------|---------------------|
| Critical | CEO only, with documented justification and compensating controls |
| High | CTO, with documented justification and compensating controls |
| Medium | CTO or Security Lead, with documented rationale |
| Low | Security Lead, documented in the risk register |

All risk acceptance decisions are documented in the risk register with: the approver, date, justification, compensating controls (if any), and the next review date.

### 5.4 Risk Register Maintenance

The risk register is the authoritative living document for all identified risks. It is maintained by the Security Lead and reviewed by the CTO quarterly.

Each risk entry includes:

| Field | Description |
|-------|-------------|
| Risk ID | Unique identifier (format: R-[CATEGORY]-NNN) |
| Risk Title | Concise description of the risk |
| Risk Category | Technology, Security, Operational, Compliance, AI-Specific, Vendor, Fraud |
| Threat Description | What could go wrong and how |
| Affected Assets | Systems, data, or processes affected |
| Inherent Likelihood | Rating before controls |
| Inherent Impact | Rating before controls |
| Inherent Risk Score | Calculated from inherent likelihood and impact |
| Current Controls | Existing controls that reduce the risk |
| Residual Likelihood | Rating after controls |
| Residual Impact | Rating after controls |
| Residual Risk Score | Calculated from residual likelihood and impact |
| Treatment Decision | Mitigate, Accept, Transfer, or Avoid |
| Treatment Plan | Specific actions to further reduce the risk |
| Treatment Owner | Person responsible for executing the treatment plan |
| Target Date | Date by which the treatment plan should be complete |
| Status | Open, In Progress, Mitigated, Accepted, Closed |
| Last Reviewed | Date of most recent review |
| CC Mapping | Relevant SOC 2 Common Criteria |

### 5.5 AI-Specific Risk Categories

Given the nature of the corteX platform, the following AI-specific risk categories are assessed in addition to standard information security risks:

| Risk ID | Risk | Inherent Rating | Current Controls | Residual Rating |
|---------|------|-----------------|-----------------|-----------------|
| R-AI-001 | Prompt injection bypasses SafetyPolicy | High | `SafetyPolicy._INJECTION_PATTERNS` pattern matching, input validation | Medium |
| R-AI-002 | Cross-tenant data leakage through shared agent state | High | Session isolation, tenant_id enforcement at every data layer, isolated memory subsystems | Medium |
| R-AI-003 | API keys exposed in agent outputs or logs | Critical | `KeyVault.detect_leak()` automatic detection, log sanitization | High |
| R-AI-004 | LLM provider trains on customer data | Medium | Enterprise API tiers with no-training guarantees, DPA with providers | Low |
| R-AI-005 | Supply chain attack via malicious Python dependency | High | Dependency pinning, vulnerability scanning, SBOM generation | Medium |
| R-AI-006 | Model output unreliable or hallucinated | Medium | `GoalTracker` verification, confidence scoring, multi-step validation | Low |
| R-AI-007 | Agent infinite loop causing resource exhaustion | Medium | State hashing, loop prevention, configurable step limits | Low |
| R-AI-008 | Unauthorized tool execution by agent | High | `ToolPolicy` allowlisting with `default_deny` posture | Low |
| R-AI-009 | Brain-inspired learning leaks user behavioral patterns | Medium | Per-tenant weight isolation, weight delta erasure support | Low |
| R-AI-010 | License key bypass or forgery | Medium | Ed25519 cryptographic validation | Low |

These AI-specific risks are reviewed during every quarterly risk review and updated as the threat landscape evolves.

### 5.6 Fraud Risk Assessment

Questo Ltd assesses fraud risks annually as part of the formal risk assessment. The fraud risk assessment considers:

#### 5.6.1 Internal Fraud Risks

| Risk | Description | Controls |
|------|-------------|----------|
| Intellectual property theft | Employee or contractor copies source code or proprietary algorithms | GitHub access controls, DLP, repository monitoring, NDA enforcement, exit procedures (POL-002) |
| Unauthorized credential use | Personnel use credentials beyond their authorized scope | RBAC enforcement, access logging, quarterly access reviews (POL-002) |
| Financial fraud | Unauthorized financial transactions or expense manipulation | Segregation of duties, approval workflows, financial audit |
| Data manipulation | Intentional corruption of customer data or system records | Change management controls (POL-003), audit logging, integrity monitoring |

#### 5.6.2 External Fraud Risks

| Risk | Description | Controls |
|------|-------------|----------|
| License fraud | Unauthorized use of the corteX SDK without valid license | Ed25519 license validation, usage monitoring |
| Social engineering | Phishing or pretexting targeting Questo personnel | Security awareness training (POL-001), MFA (POL-002), incident reporting (POL-004) |
| Account takeover | External attacker gains access through stolen credentials | MFA on all systems (POL-002), monitoring for anomalous access patterns |
| Supply chain manipulation | Adversary compromises a dependency or build tool | Dependency scanning, reproducible builds, SBOM verification |

#### 5.6.3 Management Override Controls

- Segregation of duties ensures no single person controls end-to-end processes (POL-002).
- Code changes require independent peer review (POL-003).
- Financial transactions require dual approval.
- Access to production systems requires the CTO and is fully logged.
- Quarterly access reviews provide independent verification of access appropriateness.

### 5.7 Third-Party Risk Assessment

All third-party vendors and service providers whose services or products affect the security of the corteX platform are subject to risk assessment.

#### 5.7.1 Vendor Classification

| Classification | Description | Assessment Frequency | Requirements |
|----------------|-------------|---------------------|-------------|
| **Critical** | Vendors with direct access to customer data or core infrastructure | Annually | SOC 2 report, DPA, security questionnaire, contract review |
| **High** | Vendors providing significant services but without direct data access | Annually | SOC 2 report or equivalent, DPA if applicable, contract review |
| **Medium** | Vendors providing limited services with minimal security impact | Every 2 years | Security questionnaire, contract review |
| **Low** | Vendors with no data access and minimal operational dependency | Every 3 years | Contract review |

#### 5.7.2 Critical Vendor Assessments

The following vendors are classified as Critical or High and are assessed annually:

| Vendor | Service | Classification | Key Assessment Areas |
|--------|---------|---------------|---------------------|
| OpenAI | LLM Provider | Critical | SOC 2 report, DPA with no-training clause, data residency, incident notification |
| Google (Gemini/Vertex) | LLM Provider + Cloud | Critical | SOC 2 report, CDPA, EU data residency options, incident notification |
| Anthropic | LLM Provider | Critical | SOC 2 report, DPA with no-training clause, data residency, incident notification |
| GitHub | Source code hosting, CI/CD | Critical | SOC 2 report, access controls, secret scanning, incident notification |
| Cloud Provider | Infrastructure hosting | Critical | SOC 2 report, shared responsibility model, encryption, incident notification |
| PyPI | Package distribution | High | Publishing security, MFA requirements, package integrity verification |

#### 5.7.3 Vendor Assessment Procedure

1. The Security Lead maintains the vendor inventory with current classifications.
2. For Critical and High vendors, the Security Lead requests and reviews the vendor's SOC 2 report (or equivalent) annually.
3. The Security Lead evaluates whether the vendor's controls adequately address the risks specific to Questo's use of their services.
4. Findings are documented in the vendor assessment record, including: vendor name, classification, assessment date, documents reviewed, identified risks, and recommendations.
5. Material vendor risk findings are added to the risk register.
6. The CTO reviews vendor assessments and approves continued use or required remediation actions.

#### 5.7.4 Sub-Processor Management

For GDPR compliance, all sub-processors (vendors that process personal data on behalf of Questo or Questo's customers) are documented and managed:

- A current list of sub-processors is maintained and made available to customers.
- Data Processing Agreements (DPAs) are in place with all sub-processors.
- DPAs include: processing scope, data types, security requirements, breach notification obligations, audit rights, and data return/deletion obligations.
- Changes to sub-processors are communicated to affected customers per contractual requirements.

### 5.8 Change-Driven Risk Reassessment

Significant changes to the corteX platform trigger a risk reassessment:

1. The change author identifies risks in the Pull Request description per POL-003.
2. For significant changes (as defined in POL-003 Section 5.1), the CTO or Security Lead reviews the risk analysis.
3. New risks identified through the change process are added to the risk register.
4. Existing risks affected by the change are re-evaluated.
5. The change may be blocked if it introduces unacceptable risk without adequate mitigation.

### 5.9 Annual Risk Review Process

The annual formal risk assessment follows this process:

1. **Preparation** (2 weeks before): The Security Lead distributes the current risk register and assessment templates to all participants.
2. **Risk Identification Workshop**: The CTO facilitates a workshop with the Security Lead, senior engineers, and relevant stakeholders to identify new risks and review existing risks.
3. **Risk Analysis**: Each identified risk is scored using the methodology in Section 5.1. Inherent and residual risk scores are calculated.
4. **Risk Treatment Planning**: For each risk above the acceptable threshold, a treatment plan is developed with specific actions, owners, and target dates.
5. **Risk Acceptance Review**: Risks proposed for acceptance are reviewed by the appropriate authority per Section 5.3.1.
6. **Report**: The Security Lead prepares the Annual Risk Assessment Report, summarizing: the risk landscape, changes since the last assessment, top risks, treatment plans, and accepted risks.
7. **Approval**: The CTO reviews and approves the report. Critical risks are escalated to the CEO.
8. **Distribution**: The report is shared with the CEO and relevant stakeholders.
9. **Tracking**: Treatment plan action items are entered into the tracking system and monitored during quarterly risk reviews.

## 6. Procedures

### 6.1 Risk Identification Procedure

1. Identify the threat source and threat event.
2. Identify the vulnerability that the threat could exploit.
3. Identify the asset(s) at risk.
4. Determine the potential impact to confidentiality, integrity, and availability.
5. Record the risk in the risk register with a unique Risk ID.
6. Notify the Security Lead for prioritization and initial assessment.

### 6.2 Risk Scoring Procedure

1. Assess the inherent likelihood (before controls) using the scale in Section 5.1.1.
2. Assess the inherent impact using the scale in Section 5.1.2.
3. Calculate the inherent risk score using the matrix in Section 5.1.3.
4. Document existing controls that address the risk.
5. Assess the residual likelihood and impact (after controls).
6. Calculate the residual risk score.
7. Compare the residual risk to the acceptance criteria in Section 5.3.1.
8. Determine the appropriate treatment (mitigate, accept, transfer, avoid).

### 6.3 Quarterly Risk Review Procedure

1. The Security Lead distributes the current risk register to the CTO 5 business days before the review.
2. The CTO chairs the quarterly risk review meeting.
3. The meeting covers: review of risk register changes, progress on treatment plans, newly identified risks, changes to the threat landscape, and vendor risk updates.
4. Meeting minutes document: attendees, decisions made, risks added or modified, and action items.
5. The risk register is updated to reflect review outcomes.
6. Action items are assigned owners and target dates.

### 6.4 Vendor Risk Assessment Procedure

1. The Security Lead identifies vendors requiring assessment per the schedule in Section 5.7.1.
2. For Critical/High vendors: request SOC 2 report, DPA, and complete the vendor security questionnaire.
3. Review the vendor's controls against Questo's security requirements.
4. Document findings in the vendor assessment record.
5. Identify any vendor risks and add them to the risk register.
6. Present findings to the CTO for review and approval.
7. Communicate any required remediation actions to the vendor.
8. File the assessment record and schedule the next assessment.

## 7. Exceptions

1. Exceptions to the annual risk assessment schedule require CEO approval and are limited to extraordinary circumstances (e.g., force majeure events).
2. The quarterly risk review frequency may not be reduced below quarterly.
3. Critical-rated risks may not remain in an "unmitigated" status for more than 30 days without CEO-approved acceptance.
4. Third-party risk assessments for Critical vendors may not be deferred beyond 14 months from the previous assessment.
5. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Failure to report known risks or vulnerabilities is a policy violation subject to disciplinary action.
- Risk treatment plan owners who miss their target dates without documented justification and an approved extension face escalation to the CTO.
- Circumventing risk management processes (e.g., deploying changes without required risk assessment) is a serious policy violation per POL-003 enforcement provisions.
- The CTO reviews risk management compliance as part of the quarterly risk review.
- The annual risk assessment is a required activity; failure to complete it constitutes a control deficiency for SOC 2 purposes.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-003 | Change Management Policy |
| POL-004 | Incident Response Plan |
| POL-006 | Business Continuity Plan |
| POL-007 | Disaster Recovery Plan |
| POL-008 | Data Classification Policy |
| POL-009 | Vendor Management Policy |
| POL-017 | Logging and Monitoring Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
