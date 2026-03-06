# Information Security Policy

**Document ID**: POL-001
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO
**Classification**: INTERNAL

---

## 1. Purpose

This Information Security Policy establishes the organizational approach to information security at Questo Ltd. It defines the security objectives, principles, and governance structure that protect the confidentiality, integrity, and availability of all information assets associated with the corteX AI Agent SDK platform and Questo Ltd's business operations.

This policy serves as the master security policy from which all other security policies derive their authority. It addresses the unique security challenges inherent in operating a multi-tenant, enterprise-grade AI agent SDK, including prompt injection protection, cross-tenant data isolation, and Bring-Your-Own-Key (BYOK) model management.

**SOC 2 Mapping**: CC1.1-CC1.5 (Control Environment), CC2.1-CC2.3 (Communication and Information), CC5.1-CC5.3 (Control Activities)

## 2. Scope

This policy applies to:

- **Personnel**: All employees, contractors, consultants, temporary staff, and any third parties with access to Questo Ltd information systems or data.
- **Systems**: All information systems, including development environments, production infrastructure, cloud services, communication platforms, and third-party services used in the operation of the corteX platform.
- **Data**: All data created, received, stored, transmitted, or processed by Questo Ltd, including but not limited to source code, customer data, API keys, model configurations, and business records.
- **Locations**: All locations from which Questo Ltd business is conducted, including offices, remote work locations, and cloud environments.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **corteX** | The enterprise AI Agent SDK developed and distributed by Questo Ltd |
| **BYOK** | Bring-Your-Own-Key model allowing tenants to provide their own LLM API keys |
| **Tenant** | An enterprise customer organization using the corteX SDK |
| **KeyVault** | The corteX module (`corteX/security/vault.py`) responsible for per-tenant API key isolation and protection |
| **SafetyPolicy** | The corteX module enforcing input/output safety checks, including prompt injection protection |
| **TenantConfig** | Configuration object defining per-tenant security settings, safety levels, and access controls |
| **Information Asset** | Any data, system, software, hardware, or intellectual property owned or managed by Questo Ltd |
| **Classified Information** | Information categorized as INTERNAL, CONFIDENTIAL, or RESTRICTED per POL-008 |
| **Multi-Tenant Isolation** | Architectural guarantee that data and state from one tenant cannot be accessed by another |

## 4. Roles and Responsibilities

### 4.1 Chief Executive Officer (CEO)

- Approves this policy and all subordinate security policies
- Provides organizational commitment to information security through resource allocation
- Receives quarterly security posture reports from the CTO
- Serves as final authority for risk acceptance decisions exceeding CTO threshold

### 4.2 Chief Technology Officer (CTO)

- Owns this policy and is responsible for its implementation and maintenance
- Chairs quarterly security review meetings
- Approves risk treatment plans and technology security controls
- Ensures security is integrated into the corteX development lifecycle
- Reports security posture to the CEO quarterly

### 4.3 Security Lead

- Executes day-to-day security operations and monitoring
- Manages access reviews, vulnerability scanning, and incident response coordination
- Maintains the risk register and tracks remediation efforts
- Conducts security awareness training and onboarding

### 4.4 Engineering Team Members

- Comply with all security policies and procedures
- Report suspected security incidents immediately via the process defined in POL-004
- Complete mandatory security awareness training within 30 days of hire and annually thereafter
- Follow secure coding practices defined in the SDLC Policy (POL-018)

### 4.5 All Personnel

- Acknowledge and comply with this policy and all related policies
- Protect credentials and never share account access
- Report suspected security incidents or policy violations promptly
- Complete required security awareness training on schedule

## 5. Policy Statements

### 5.1 Security Objectives

Questo Ltd commits to the following security objectives for the corteX platform:

1. **Confidentiality**: Protect tenant data, API keys, and proprietary information from unauthorized disclosure. Enforce strict multi-tenant isolation so that no tenant can access another tenant's data, configurations, or operational state.
2. **Integrity**: Ensure the accuracy and completeness of AI agent processing through goal tracking, input/output validation, and loop prevention. Maintain the integrity of the corteX codebase through rigorous change management.
3. **Availability**: Maintain the availability of the corteX SDK distribution, documentation, and supporting services. Support on-premises deployments that operate independently of Questo infrastructure.
4. **Privacy**: Process personal data in compliance with GDPR and applicable data protection regulations. Minimize data collection and enforce purpose limitation.

### 5.2 Security Principles

All security decisions at Questo Ltd are governed by these principles:

- **Defense in Depth**: Multiple layers of security controls protect information assets. No single control failure should result in a security breach.
- **Least Privilege**: Access to systems and data is limited to the minimum necessary for job functions (detailed in POL-002).
- **Security by Default**: The corteX SDK ships with secure defaults. SafetyPolicy enforcement is enabled by default. Tool execution requires explicit allowlisting via ToolPolicy.
- **On-Premises First**: The corteX SDK must operate with zero required external dependencies. All security controls must function in air-gapped environments.
- **Tenant Isolation as Architecture**: Multi-tenant data isolation is enforced at the architectural level through session-scoped memory, tenant-scoped KeyVault instances, and tenant-filtered audit logs. This is not a configuration option; it is a structural guarantee.

### 5.3 Information Asset Classification

All information assets are classified into four levels per POL-008:

| Level | Description | Examples |
|-------|-------------|----------|
| **RESTRICTED** | Highest sensitivity; compromise causes severe damage | Customer API keys, Ed25519 signing keys, encryption keys |
| **CONFIDENTIAL** | Business-sensitive; compromise causes significant damage | Customer prompts, source code, architecture documents, employee data |
| **INTERNAL** | Not for public release; compromise causes moderate damage | Internal procedures, meeting notes, non-sensitive configurations |
| **PUBLIC** | Approved for public consumption | SDK documentation, public API specifications, marketing materials |

These levels map directly to the `DataLevel` enum enforced programmatically by `corteX/security/classification.py`.

### 5.4 AI-Specific Security Controls

The corteX platform implements the following AI-specific security measures:

1. **Prompt Injection Protection**: `SafetyPolicy` includes pattern-matching detection for known prompt injection techniques (`_INJECTION_PATTERNS`). Input validation occurs before any prompt reaches the LLM provider.
2. **Output Safety Enforcement**: `SafetyPolicy.check_output()` validates all model outputs against safety criteria before returning results to the calling application.
3. **Multi-Tenant Data Isolation**: Each tenant session operates in an isolated memory space. Working memory, short-term memory, long-term memory, and episodic memory are all scoped to the tenant. Cross-tenant queries are architecturally impossible.
4. **BYOK Model**: Tenants provide their own LLM API keys, which are stored in tenant-scoped KeyVault instances. Questo Ltd does not hold or have access to tenant API keys in production deployments.
5. **Tool Execution Control**: The `ToolPolicy` system enforces allowlisting for all tool executions. Agents cannot call external tools or APIs unless the tenant explicitly enables them. Default posture is deny-all.
6. **Loop Prevention**: The brain-inspired engine uses state hashing and drift detection to prevent infinite agent loops and resource exhaustion.
7. **Goal Tracking**: `GoalTracker` verifies every agent step against the original goal, providing processing integrity assurance.

### 5.5 Human Resources Security

- All employees and contractors undergo background checks prior to receiving access to information systems (CC1-04).
- Employment agreements include confidentiality obligations and security responsibilities.
- All personnel complete security awareness training within 30 days of hire and annually thereafter (CC1-05).
- Annual performance reviews include evaluation of security responsibility adherence (CC1-06).
- Upon termination or role change, access is revoked within 24 hours per the deprovisioning process in POL-002.
- Contractor engagements require signed NDAs and security acknowledgements (CC1-08).

### 5.6 Physical and Environmental Security

- Cloud infrastructure physical security is inherited from cloud service providers (verified through their SOC 2 reports).
- Office facilities employ appropriate access controls proportionate to the sensitivity of work conducted.
- Full disk encryption is required on all company-issued and personal devices used for Questo business (POL-020).
- Clean desk practices are enforced to prevent unauthorized viewing of classified information.

### 5.7 Communications and Operations Management

- All internal security communications use designated secure channels.
- Security-relevant SDK changes are explicitly flagged in the CHANGELOG (CC2-07).
- Customer notification procedures are defined for security incidents that may affect SDK users (POL-004).
- A public trust and security page is maintained on the corteX documentation site (CC2-04).

### 5.8 Access Control

Detailed access control requirements are defined in POL-002. Key principles:

- Multi-factor authentication (MFA) is required on all systems without exception.
- Role-Based Access Control (RBAC) governs access to all systems and data.
- Quarterly access reviews are conducted with manager sign-off.
- Privileged access is logged and monitored.

### 5.9 System Development and Maintenance

- The corteX SDK follows secure development lifecycle practices (POL-018).
- All code changes require peer review through GitHub Pull Requests.
- A CI/CD pipeline with 8,082+ automated tests must pass before any merge (POL-003).
- Dependencies are scanned for known vulnerabilities.
- Secret scanning is enabled on all repositories.
- The SDK enforces a maximum of 300 lines per file, type hints on all public functions, async by default for I/O, and logging over print statements.

### 5.10 Compliance and Legal Requirements

- Questo Ltd complies with GDPR for processing data of EU residents, including the 72-hour breach notification requirement (Art. 33-34).
- Data processing agreements (DPAs) are maintained with all sub-processors (LLM providers, cloud providers).
- The corteX SDK supports customer compliance by providing data classification enforcement, audit logging, and data erasure capabilities.
- Regular compliance reviews are conducted as part of the annual risk assessment process (POL-005).

## 6. Procedures

### 6.1 Policy Distribution and Acknowledgement

1. This policy is distributed to all employees and contractors upon hire and whenever substantive updates are made.
2. All personnel must sign an acknowledgement confirming they have read and understood this policy within 14 days of distribution.
3. Acknowledgement records are maintained by the Security Lead and retained for the duration of employment plus three years.

### 6.2 Security Awareness Training

1. New hires complete security awareness training within 30 days of start date.
2. All personnel complete annual refresher training covering: information classification, incident reporting, phishing awareness, secure coding practices, and AI-specific security risks (prompt injection, data leakage).
3. Training completion records are maintained with dates and topics covered.
4. The CTO reviews and approves training content annually.

### 6.3 Quarterly Security Review

1. The CTO chairs a quarterly security review meeting.
2. The agenda includes: review of security incidents, risk register updates, access review results, vulnerability scan findings, policy compliance status, and upcoming security initiatives.
3. Meeting minutes are documented and retained as evidence of ongoing security governance.
4. Action items are tracked to completion with assigned owners and due dates.

### 6.4 Annual Policy Review

1. This policy and all subordinate policies are reviewed at least annually by the policy owner (CTO).
2. Reviews assess the continued adequacy of the policy in light of changes to: the threat landscape, business operations, regulatory requirements, and technology environment.
3. Substantive changes require CEO approval before publication.
4. The review date, reviewer, and any changes made are recorded in the Revision History.

### 6.5 Exception Management

All exceptions to this policy follow the process defined in Section 7.

## 7. Exceptions

Exceptions to this policy may be granted only under the following conditions:

1. **Request**: The requesting party submits a written exception request to the CTO, including: the specific policy provision requiring an exception, the business justification, the proposed compensating controls, and the requested duration.
2. **Review**: The CTO evaluates the risk posed by the exception, considering the compensating controls and the potential impact of a security incident.
3. **Approval**: Exceptions affecting RESTRICTED or CONFIDENTIAL data require CEO approval. Exceptions affecting INTERNAL data may be approved by the CTO alone.
4. **Duration**: Exceptions are granted for a maximum of 12 months and must be renewed through the same process.
5. **Documentation**: All exceptions are documented in the Exception Register, including the approval, compensating controls, and expiration date.
6. **Review**: Active exceptions are reviewed during each quarterly security review meeting.

## 8. Enforcement

- Violations of this policy may result in disciplinary action, up to and including termination of employment or contract.
- Violations involving intentional data theft, sabotage, or fraud will be referred to law enforcement.
- The Security Lead is responsible for investigating reported policy violations.
- Enforcement actions are documented and reported to the CTO.
- Repeated unintentional violations will result in mandatory remedial training before continued system access.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-002 | Access Control Policy |
| POL-003 | Change Management Policy |
| POL-004 | Incident Response Plan |
| POL-005 | Risk Assessment and Mitigation Policy |
| POL-006 | Business Continuity Plan |
| POL-007 | Disaster Recovery Plan |
| POL-008 | Data Classification Policy |
| POL-009 | Vendor Management Policy |
| POL-010 | Acceptable Use Policy |
| POL-011 | Encryption Policy |
| POL-012 | Password Policy |
| POL-013 | Code of Conduct |
| POL-017 | Logging and Monitoring Policy |
| POL-018 | Software Development Lifecycle Policy |
| POL-020 | Workstation Security Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
