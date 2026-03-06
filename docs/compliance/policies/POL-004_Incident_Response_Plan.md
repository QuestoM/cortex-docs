# Incident Response Plan

**Document ID**: POL-004
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO
**Classification**: INTERNAL

---

## 1. Purpose

This Incident Response Plan (IRP) establishes the procedures for detecting, responding to, containing, eradicating, and recovering from security incidents affecting Questo Ltd and the corteX AI Agent SDK platform. It defines incident severity levels, response team roles, communication protocols, and post-incident review processes. This plan addresses corteX-specific incident scenarios including cross-tenant data leakage, API key exposure, and prompt injection bypasses, as well as regulatory obligations under the GDPR 72-hour breach notification requirement.

**SOC 2 Mapping**: CC7.3 (Event Evaluation), CC7.4 (Incident Response), CC7.5 (Recovery)

## 2. Scope

This plan applies to:

- **Incidents**: All security incidents, suspected security incidents, and security events affecting Questo Ltd information systems, data, personnel, or customers.
- **Systems**: All systems within Questo Ltd's operational environment, including development infrastructure, production environments, cloud services, communication platforms, and the corteX SDK itself.
- **Data**: All data categories from PUBLIC through RESTRICTED, with heightened procedures for incidents involving CONFIDENTIAL or RESTRICTED data.
- **Personnel**: All employees, contractors, and third parties who may discover, report, or respond to security incidents.
- **Customers**: Incidents that may affect corteX SDK users, their data, or their operations.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Security Event** | An observable occurrence in a system or network that may indicate a security concern |
| **Security Incident** | A confirmed or strongly suspected security event that has compromised or threatens the confidentiality, integrity, or availability of information or systems |
| **Data Breach** | A security incident resulting in the unauthorized access to, disclosure of, or loss of personal data or classified information |
| **Incident Response Team (IRT)** | The designated group responsible for managing and coordinating the response to security incidents |
| **Incident Commander (IC)** | The individual directing the overall incident response effort |
| **Containment** | Actions taken to limit the scope and impact of an incident |
| **Eradication** | Actions taken to eliminate the root cause of an incident |
| **Recovery** | Actions taken to restore systems and operations to their normal state |
| **Post-Incident Review (PIR)** | A structured analysis conducted after incident resolution to identify lessons learned and improvements |
| **Cross-Tenant Leakage** | A security incident where data from one corteX tenant becomes accessible to another tenant |
| **Tabletop Exercise** | A discussion-based exercise where the IRT walks through a simulated incident scenario |

## 4. Roles and Responsibilities

### 4.1 Incident Response Team (IRT)

| Role | Primary | Backup | Responsibilities |
|------|---------|--------|-----------------|
| **Incident Commander** | CTO | Security Lead | Overall incident direction, decision authority, stakeholder communication |
| **Technical Lead** | Security Lead | Senior Engineer | Technical investigation, containment execution, evidence collection |
| **Communications Lead** | CEO | CTO | Customer notification, regulatory notification, public communications |
| **Engineering Support** | Senior Engineer(s) | Engineer(s) | Technical remediation, system recovery, forensic analysis |

### 4.2 All Personnel

- Report suspected security incidents immediately through the reporting channels defined in Section 6.1.
- Preserve evidence by not modifying or deleting potentially relevant data, logs, or systems.
- Cooperate fully with the IRT during investigations.
- Maintain confidentiality about incident details until communications are authorized by the Incident Commander.

### 4.3 External Parties

- **Legal Counsel**: Consulted for incidents with regulatory, legal, or contractual implications.
- **Forensic Specialists**: Engaged for P0 incidents requiring independent forensic analysis.
- **Law Enforcement**: Engaged when criminal activity is suspected, at the direction of the CEO and legal counsel.
- **Data Protection Authority**: Notified within 72 hours for GDPR-qualifying breaches (Section 5.8).

## 5. Policy Statements

### 5.1 Incident Severity Classification

All security incidents are classified using the following severity levels:

| Severity | Description | Response Time | Update Frequency | Examples |
|----------|-------------|---------------|------------------|---------|
| **P0 - Critical** | Active data breach, system compromise, or cross-tenant data leakage. Immediate threat to customer data or Questo operations. | **Immediate** (within 15 minutes of identification) | Every 30 minutes until contained | Cross-tenant data leak, customer API key exposure at scale, active attacker in systems, ransomware |
| **P1 - High** | Potential data exposure, significant unpatched vulnerability, or partial system compromise. No confirmed data exfiltration but credible threat. | **1 hour** | Every 2 hours until contained | Critical SDK vulnerability, dependency with active CVE, single API key exposure, unauthorized access detected |
| **P2 - Medium** | Security policy violation, minor vulnerability, or suspicious activity without confirmed compromise. | **4 hours** | Daily until resolved | Failed brute-force attempts, configuration error exposing non-sensitive data, phishing email received but not acted upon |
| **P3 - Low** | Informational event, potential hardening opportunity, or policy clarification needed. | **Next business day** | As needed | Minor policy non-compliance, security improvement suggestion, low-severity vulnerability in non-critical component |

Severity may be escalated or de-escalated as new information becomes available. The Incident Commander makes escalation decisions.

### 5.2 Incident Detection Sources

Incidents may be detected through the following channels:

1. **Automated Monitoring**: CI/CD pipeline alerts, log analysis alerts, secret scanning alerts, vulnerability scanning alerts, and anomaly detection systems.
2. **Internal Reporting**: Employee observation and reporting through the designated channels.
3. **External Reporting**: Customer reports, security researcher disclosures, vendor notifications, and public vulnerability databases (NVD, GitHub Advisories).
4. **Audit Findings**: Discoveries during access reviews, penetration tests, or internal audits.
5. **corteX SDK Alerts**: `KeyVault.detect_leak()` automatic detection of API key exposure, `SafetyPolicy` alerts for prompt injection attempts, and `AuditConfig` monitoring events.

### 5.3 Incident Response Phases

Every incident follows these six phases:

#### Phase 1: Detection and Reporting

- The individual who detects the event reports it immediately through the designated channel (Section 6.1).
- The Security Lead performs initial triage to determine if the event constitutes a security incident.
- If confirmed or strongly suspected, the Security Lead assigns a severity level and activates the IRT.

#### Phase 2: Assessment and Triage

- The Incident Commander confirms the severity classification.
- The Technical Lead gathers initial facts: what happened, when, what systems and data are affected, and what is the current scope.
- The Incident Commander determines the response posture: full IRT activation (P0/P1) or standard handling (P2/P3).
- An incident ticket is created with a unique incident ID, initial classification, and known facts.

#### Phase 3: Containment

Containment actions depend on the incident type and severity:

| Incident Type | Immediate Containment | Short-Term Containment |
|--------------|----------------------|----------------------|
| Cross-tenant data leakage | Isolate affected tenant sessions, halt affected agent processes | Disable cross-tenant pathways, deploy isolation patch |
| API key exposure | Rotate exposed keys immediately, revoke compromised credentials | Audit all key usage for the exposure window, notify affected tenants |
| Prompt injection bypass | Enable maximum SafetyLevel (LOCKED), block identified injection patterns | Deploy pattern update to SafetyPolicy, notify affected tenants |
| Unauthorized system access | Disable compromised account, revoke all sessions | Reset all credentials for affected systems, review access logs |
| Malicious dependency | Pin to last known-good version, block update | Remove malicious dependency, audit for impact, publish advisory |
| LLM provider breach | Rotate all API keys for affected provider, assess data exposure | Review DPA obligations, assess tenant impact, notify tenants |

Evidence must be preserved before any containment action that modifies system state. The Technical Lead ensures forensic copies are taken where appropriate.

#### Phase 4: Eradication

- The Technical Lead identifies and eliminates the root cause of the incident.
- For code vulnerabilities: a fix is developed, reviewed, and deployed through the emergency change process (POL-003 Section 5.7).
- For compromised accounts: credentials are reset, MFA is re-enrolled, and the account is audited for unauthorized actions.
- For configuration errors: the configuration is corrected and validated.
- The Technical Lead verifies that the eradication is complete and the threat vector is eliminated.

#### Phase 5: Recovery

- Systems are restored to normal operations in a controlled manner.
- The Technical Lead verifies system integrity before restoring full service.
- Monitoring is enhanced for the affected systems for a minimum of 30 days post-incident.
- Tenants are notified of service restoration and any actions they need to take (e.g., key rotation).
- The Incident Commander formally declares the incident resolved.

#### Phase 6: Post-Incident Review

- A Post-Incident Review (PIR) meeting is held within 5 business days of incident resolution for P0/P1 incidents and within 10 business days for P2 incidents.
- The PIR documents: timeline of events, root cause analysis, effectiveness of the response, lessons learned, and recommended improvements.
- Action items from the PIR are assigned owners and tracked to completion.
- The PIR report is retained for a minimum of 3 years.
- P3 incidents are reviewed in aggregate during the monthly security review.

### 5.4 Communication Plan

#### 5.4.1 Internal Communication

| Severity | Notify Immediately | Notify Within 1 Hour | Notify Within 24 Hours |
|----------|-------------------|---------------------|----------------------|
| P0 | CTO, CEO, Security Lead | All IRT members | All employees |
| P1 | CTO, Security Lead | All IRT members | Relevant team leads |
| P2 | Security Lead | CTO | Relevant team leads |
| P3 | Security Lead | -- | -- |

Internal communications use the designated secure channel. Incident details are not discussed outside the IRT until the Incident Commander authorizes broader communication.

#### 5.4.2 Customer Communication

- Affected customers are notified within **24 hours** of confirming an incident that impacts their data or service.
- Customer notifications include: description of the incident, data affected, actions Questo has taken, actions the customer should take, and a point of contact for questions.
- The Communications Lead (CEO) approves all customer-facing notifications.
- Follow-up communications are sent when significant new information becomes available and upon incident resolution.

#### 5.4.3 Regulatory Communication

- GDPR breach notifications are handled per Section 5.8.
- Other regulatory notifications are coordinated with legal counsel.
- Regulatory communications are approved by the CEO.

### 5.5 Evidence Preservation

For all P0 and P1 incidents, the following evidence is preserved:

1. System logs from affected systems (preserved before any remediation)
2. Network traffic captures (if available)
3. Affected file system snapshots or forensic images
4. Screenshots of relevant system states
5. Email, chat, and communication records related to the incident
6. Git history and deployment records for the time window
7. Access logs for affected systems and accounts

Evidence is stored in a restricted-access location, labeled with the incident ID, and retained for a minimum of 3 years. Chain of custody is maintained for evidence that may be used in legal proceedings.

### 5.6 corteX-Specific Incident Scenarios

#### Scenario 1: Cross-Tenant Data Leakage

**Indicators**: Tenant data appearing in another tenant's session, memory isolation failure, cross-tenant query succeeding.

**Response**:
1. Immediately halt all affected agent sessions.
2. Isolate the affected tenant environments.
3. Identify the isolation failure point (session management, memory subsystem, or KeyVault).
4. Assess the scope: which tenants are affected, what data was exposed, and for how long.
5. Notify all affected tenants within 24 hours with specific data exposure details.
6. Deploy a fix through the emergency change process (POL-003).
7. Conduct a full architectural review of tenant isolation controls.
8. Engage an external security firm for independent verification of the fix.

**Severity**: P0 (Critical)

#### Scenario 2: Customer API Key Exposure

**Indicators**: `KeyVault.detect_leak()` alert, API key found in logs, API key committed to repository, customer report of unauthorized API usage.

**Response**:
1. Immediately rotate the exposed API key (if Questo has the ability; for BYOK, notify the customer immediately to rotate).
2. Identify the exposure vector (log output, code commit, memory leak, etc.).
3. Audit all API calls made with the exposed key during the exposure window.
4. Notify the affected tenant with: the key that was exposed, the exposure timeframe, the exposure vector, and instructions for rotation.
5. Patch the exposure vector and deploy through emergency change process.
6. Review and strengthen `detect_leak()` patterns if the exposure was not automatically detected.

**Severity**: P0 if broad exposure; P1 if limited and quickly contained.

#### Scenario 3: Prompt Injection Bypass

**Indicators**: SafetyPolicy logs showing blocked attempts followed by successful execution of prohibited actions, unexpected agent behavior reported by tenant, output containing content that violates configured safety policies.

**Response**:
1. Set affected tenants to `SafetyLevel.LOCKED` to prevent further exploitation.
2. Identify the injection technique that bypassed `SafetyPolicy._INJECTION_PATTERNS`.
3. Develop and deploy updated detection patterns.
4. Audit agent outputs from the exploitation window for data leakage or unauthorized actions.
5. Notify affected tenants with: the nature of the bypass, actions taken, and any data exposure.
6. Publish a security advisory for the SDK if the vulnerability affects all deployments.

**Severity**: P1 (High); escalate to P0 if data exfiltration confirmed.

#### Scenario 4: Malicious SDK Dependency

**Indicators**: Dependency scanning alert, unusual behavior in a dependency update, external vulnerability disclosure for a dependency.

**Response**:
1. Pin the corteX SDK to a known-good version of the dependency.
2. Assess whether the malicious code was included in any published SDK versions.
3. If a published version is affected: yank the affected version on PyPI, publish a patched version, and issue a security advisory.
4. Audit the malicious dependency for: data exfiltration capabilities, backdoor access, and supply chain propagation.
5. Notify all SDK users through the advisory channel.
6. Review the dependency vetting process and strengthen controls.

**Severity**: P1 (High); escalate to P0 if active exploitation or data exfiltration confirmed.

#### Scenario 5: LLM Provider Breach

**Indicators**: Provider public disclosure, anomalous API behavior, customer reports of unexpected outputs.

**Response**:
1. Assess the scope of the breach relative to corteX: were customer prompts, API keys, or outputs exposed?
2. Rotate all Questo-held API keys for the affected provider.
3. Notify tenants using the affected provider (for BYOK deployments) and recommend key rotation.
4. Review DPA obligations with the provider.
5. Assess whether alternative LLM providers should be recommended to affected tenants.
6. Document the incident for the risk register (POL-005) and vendor assessment records (POL-009).

**Severity**: P1 (High); escalate based on confirmed data exposure.

#### Scenario 6: Unauthorized SDK License Usage

**Indicators**: License validation failure logs, Ed25519 signature verification failures, usage telemetry anomalies.

**Response**:
1. Identify the unauthorized usage through license validation logs.
2. Assess whether the unauthorized use creates security risk (e.g., running an unpatched version).
3. Contact the organization through appropriate channels (legal counsel engagement).
4. Review Ed25519 license validation logic for potential bypass vulnerabilities.
5. Document for legal and business follow-up.

**Severity**: P2 (Medium) unless security bypass is involved, then P1.

### 5.7 Tabletop Exercise Requirements

1. **Frequency**: At least one tabletop exercise is conducted annually.
2. **Participants**: All IRT members must participate.
3. **Scenarios**: Exercises rotate through the corteX-specific scenarios defined in Section 5.6, covering at least two scenarios per exercise.
4. **Format**: The facilitator presents a scenario with escalating injects. Participants walk through their response actions, decisions, and communications.
5. **Documentation**: Exercise date, participants, scenario(s), key decisions, identified gaps, and improvement actions are documented.
6. **Follow-Up**: Improvement actions are assigned owners and tracked to completion within 60 days.
7. **Scheduling**: The next tabletop exercise is scheduled at the conclusion of each exercise.

### 5.8 GDPR Breach Notification

When an incident involves personal data of EU residents, the following requirements apply per GDPR Articles 33 and 34:

#### 5.8.1 Notification to Supervisory Authority (Art. 33)

- Within **72 hours** of becoming aware of a personal data breach, Questo Ltd must notify the relevant supervisory authority, unless the breach is unlikely to result in a risk to the rights and freedoms of individuals.
- The notification must include:
  - Nature of the personal data breach, including categories and approximate number of data subjects and records affected
  - Name and contact details of the Data Protection Officer or contact point
  - Likely consequences of the breach
  - Measures taken or proposed to address the breach, including mitigation measures
- If full information is not available within 72 hours, information may be provided in phases without undue delay.
- The decision to notify or not notify is documented with the rationale.

#### 5.8.2 Communication to Data Subjects (Art. 34)

- If the breach is likely to result in a **high risk** to the rights and freedoms of individuals, affected data subjects are notified without undue delay.
- The communication describes the nature of the breach in clear, plain language and includes the same information as the supervisory authority notification.
- Direct communication is preferred (email, in-app notification). Public communication is used only when direct communication would involve disproportionate effort.

#### 5.8.3 Breach Assessment Criteria

The Incident Commander, in consultation with legal counsel, assesses whether the breach requires notification based on:

- Type of personal data affected (financial, health, identity, credentials)
- Volume of data subjects affected
- Sensitivity of the data
- Whether the data was encrypted or pseudonymized
- Whether the breach has been fully contained
- Likelihood that the data has been accessed by unauthorized parties

## 6. Procedures

### 6.1 Incident Reporting Procedure

Any person who discovers or suspects a security incident must immediately:

1. **Report** the incident through one of these channels (in order of preference):
   - Dedicated security incident channel (chat/messaging platform)
   - Direct message to the Security Lead
   - Email to security@questo.io
   - Direct verbal communication to any IRT member
2. **Include** the following information in the report:
   - Date and time of discovery
   - Description of what was observed
   - Systems, accounts, or data believed to be affected
   - Any actions already taken
   - Reporter's contact information
3. **Preserve** evidence by not modifying, deleting, or shutting down affected systems unless doing so is necessary to prevent further immediate harm.

### 6.2 Incident Triage Procedure

Upon receiving an incident report, the Security Lead:

1. Acknowledges receipt within 15 minutes.
2. Performs initial assessment to determine if the event is a security incident.
3. If confirmed, assigns a severity level per Section 5.1.
4. Creates an incident ticket with a unique ID (format: INC-YYYY-NNN).
5. Notifies the Incident Commander (CTO) for P0/P1 incidents immediately.
6. Activates the IRT per the communication plan for the assigned severity.
7. Begins the Assessment and Triage phase (Section 5.3, Phase 2).

### 6.3 Post-Incident Review Procedure

1. The Incident Commander schedules the PIR meeting within 5 business days (P0/P1) or 10 business days (P2) of incident resolution.
2. All IRT members who participated in the response attend.
3. The meeting follows a blameless format focused on systemic improvement.
4. The agenda covers: incident timeline reconstruction, root cause analysis, response effectiveness assessment, lessons learned, and improvement recommendations.
5. The Technical Lead prepares the PIR report documenting all findings.
6. The Incident Commander approves the PIR report and ensures action items are entered into the tracking system.
7. Action items are reviewed during monthly security meetings until completed.

### 6.4 Incident Documentation Procedure

Every incident is documented with the following minimum information:

- Incident ID (INC-YYYY-NNN)
- Date and time of detection
- Date and time of containment
- Date and time of resolution
- Severity level (initial and final, if changed)
- Description of the incident
- Systems and data affected
- Root cause
- Actions taken (containment, eradication, recovery)
- Customer notifications sent (if any)
- Regulatory notifications sent (if any)
- PIR findings and action items
- Evidence inventory

Incident records are stored in the restricted-access incident archive and retained for a minimum of 5 years.

## 7. Exceptions

1. The severity response times defined in Section 5.1 are mandatory minimums. They may not be extended except when the Incident Commander determines that a specific incident does not pose an active threat and documents the justification.
2. The 72-hour GDPR notification window (Section 5.8) is a legal requirement and cannot be extended through internal exception processes.
3. Evidence preservation requirements (Section 5.5) may be overridden by the Incident Commander only when containment actions require immediate system modification to prevent further harm. The override decision and rationale must be documented.
4. All other exceptions to this plan require written approval from the CTO and are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Failure to report a known or suspected security incident is a serious policy violation subject to disciplinary action up to and including termination.
- Intentional destruction of evidence related to a security incident is grounds for immediate termination and referral to law enforcement.
- IRT members who fail to respond within the timeframes specified for their severity level are subject to performance review and remedial action.
- All personnel are required to cooperate with incident investigations. Obstruction of an investigation is a serious policy violation.
- The CTO reviews incident response effectiveness quarterly and addresses systemic compliance issues.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-003 | Change Management Policy |
| POL-005 | Risk Assessment and Mitigation Policy |
| POL-006 | Business Continuity Plan |
| POL-007 | Disaster Recovery Plan |
| POL-008 | Data Classification Policy |
| POL-009 | Vendor Management Policy |
| POL-017 | Logging and Monitoring Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial plan creation |
