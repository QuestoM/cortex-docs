# Access Control Policy

**Document ID**: POL-002
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO
**Classification**: INTERNAL

---

## 1. Purpose

This Access Control Policy defines how access to Questo Ltd information systems, source code, data, and infrastructure is granted, modified, reviewed, and revoked. It ensures that access is restricted to authorized personnel based on the principles of least privilege and need-to-know, protecting the corteX AI Agent SDK platform and all associated data assets from unauthorized access.

**SOC 2 Mapping**: CC5.1-CC5.2 (Control Activities), CC6.1-CC6.8 (Logical and Physical Access Controls)

## 2. Scope

This policy applies to:

- **Personnel**: All employees, contractors, consultants, and third parties who require access to any Questo Ltd information system.
- **Systems**: All systems used in the development, distribution, and support of the corteX SDK, including but not limited to: GitHub repositories, cloud infrastructure, CI/CD pipelines, communication platforms, email, PyPI, documentation hosting, and internal tools.
- **Data**: All data categories as defined in POL-008 (Data Classification Policy), from PUBLIC through RESTRICTED.
- **API Keys and Credentials**: All service accounts, API keys, SSH keys, tokens, and other authentication credentials used within Questo Ltd operations.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Least Privilege** | The principle of granting only the minimum level of access necessary to perform a job function |
| **Need-to-Know** | Access is granted only when the individual has a demonstrated business need for the specific information |
| **RBAC** | Role-Based Access Control; access permissions are assigned to roles, and users are assigned to roles |
| **MFA** | Multi-Factor Authentication; requires two or more independent authentication factors |
| **Privileged Access** | Administrative or elevated access that permits system configuration, user management, or access to RESTRICTED data |
| **Service Account** | A non-human account used by applications, CI/CD pipelines, or automated processes |
| **Access Review** | A periodic audit of user access rights to verify continued appropriateness |
| **Deprovisioning** | The process of revoking all access rights for a departing user |
| **KeyVault** | The corteX security module (`corteX/security/vault.py`) providing per-tenant API key isolation |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and ensures its implementation across all systems
- Approves access to RESTRICTED data and systems
- Chairs quarterly access review meetings
- Approves creation of new privileged roles

### 4.2 Security Lead

- Executes user provisioning and deprovisioning procedures
- Conducts quarterly access reviews and prepares review reports
- Monitors privileged access usage and investigates anomalies
- Maintains the access control matrix and RBAC definitions

### 4.3 Team Leads / Managers

- Approve access requests for their direct reports
- Participate in quarterly access reviews by confirming the continued need for each team member's access
- Notify the Security Lead within 4 hours when a team member departs or changes roles

### 4.4 All Personnel

- Request access only for legitimate business needs
- Never share credentials, tokens, or access keys with others
- Report suspected unauthorized access immediately per POL-004
- Lock workstations when unattended (automatic lock within 5 minutes maximum)

## 5. Policy Statements

### 5.1 Access Control Principles

1. **Least Privilege**: Every user, service account, and automated process is granted only the minimum access rights necessary to perform its designated function. No exceptions are granted by default.
2. **Need-to-Know**: Access to classified information (INTERNAL, CONFIDENTIAL, RESTRICTED) requires a documented business need in addition to role-based authorization.
3. **Default Deny**: Systems are configured to deny access by default. Access is granted only through explicit provisioning.
4. **Segregation of Duties**: No single individual has unchecked control over critical processes. Code authors cannot approve their own pull requests. Infrastructure changes require separate review.
5. **Individual Accountability**: Shared accounts are prohibited. Every access action must be traceable to an individual user. Service accounts are assigned a human owner responsible for their use.

### 5.2 Authentication Requirements

#### 5.2.1 Multi-Factor Authentication

MFA is **mandatory** on all Questo Ltd systems without exception. This includes:

| System | MFA Method | Enforcement |
|--------|-----------|-------------|
| GitHub (source code) | TOTP or hardware key | Organization-level enforcement |
| Cloud infrastructure (Azure/AWS/GCP) | TOTP or hardware key | Account-level enforcement |
| Email and communication platforms | TOTP or hardware key | Domain-level enforcement |
| PyPI (package publishing) | TOTP or hardware key | Account-level enforcement |
| CI/CD pipeline administration | TOTP or hardware key | Platform-level enforcement |
| VPN access | TOTP or hardware key | Gateway-level enforcement |

MFA bypass is not permitted. If a system does not support MFA natively, a compensating control (such as VPN with MFA as a prerequisite) must be implemented and documented.

#### 5.2.2 Password Requirements

Per POL-012, all passwords must meet:

- Minimum 12 characters
- Complexity requirements (uppercase, lowercase, number, special character)
- No reuse of the last 12 passwords
- Account lockout after 5 failed attempts (30-minute lockout or manual unlock)

#### 5.2.3 Session Management

- Idle sessions must timeout after a maximum of 30 minutes for web applications
- Workstation screen locks must activate within 5 minutes of inactivity
- Administrative sessions have a maximum duration of 8 hours regardless of activity

### 5.3 Role-Based Access Control (RBAC) Definitions

#### 5.3.1 Organizational Roles

| Role | GitHub Access | Cloud Access | PyPI Access | Data Access |
|------|-------------|-------------|-------------|------------|
| **CEO** | Read all repos | Billing only | None | CONFIDENTIAL |
| **CTO** | Admin all repos | Admin | Publish (with approval) | RESTRICTED |
| **Security Lead** | Write (security repos) | Security tools admin | None | RESTRICTED |
| **Senior Engineer** | Write (assigned repos) | Read + deploy | None | CONFIDENTIAL |
| **Engineer** | Write (assigned repos) | Read only | None | CONFIDENTIAL |
| **Contractor** | Write (assigned repos only) | None | None | INTERNAL |
| **Read-Only Auditor** | Read (specified repos) | Read logs only | None | CONFIDENTIAL (audit scope) |

#### 5.3.2 corteX SDK Tenant Roles

Within the corteX SDK itself, tenant-level RBAC is enforced through `TenantConfig`:

| Tenant Role | Capabilities |
|-------------|-------------|
| **Tenant Admin** | Configure TenantConfig, set SafetyLevel, manage ToolPolicy, manage API keys |
| **Tenant Developer** | Use SDK API, execute agents within configured policies |
| **Tenant Read-Only** | View agent outputs, access logs, no execution permissions |

`SafetyLevel.LOCKED` prevents tenant users from overriding safety configurations, enforcing organizational security policy at the SDK level.

### 5.4 User Provisioning

1. **Request**: The hiring manager submits an access request specifying the required role, systems, and business justification.
2. **Approval**: The CTO approves all access requests. Requests for RESTRICTED data access require documented justification reviewed by the CTO.
3. **Implementation**: The Security Lead provisions access within 2 business days of approval, following the principle of least privilege.
4. **Verification**: The new user verifies access functionality and confirms MFA enrollment on all provisioned systems.
5. **Documentation**: The provisioning action, approver, date, and systems are recorded in the access management log.

### 5.5 User Deprovisioning

1. **Notification**: The departing user's manager notifies the Security Lead within 4 hours of confirmed departure (voluntary or involuntary).
2. **Immediate Revocation**: For involuntary departures or departures involving personnel with privileged access, revocation begins immediately upon notification.
3. **Standard Revocation**: For voluntary departures, all access is revoked by end of the last working day and no later than 24 hours after departure.
4. **Checklist Execution**: The Security Lead executes the deprovisioning checklist:
   - Disable GitHub organization membership
   - Revoke cloud infrastructure access
   - Disable email and communication platform accounts
   - Revoke VPN certificates
   - Rotate any shared secrets the user had access to
   - Recover company-issued devices
   - Confirm full disk wipe of returned devices
5. **Verification**: The Security Lead verifies revocation across all systems and documents completion.
6. **Documentation**: The deprovisioning action, date, systems affected, and verifier are recorded in the access management log.

### 5.6 Access Reviews

1. **Frequency**: Access reviews are conducted quarterly (March, June, September, December).
2. **Scope**: Every active user account across all systems (GitHub, cloud, email, PyPI, CI/CD, VPN).
3. **Process**:
   - The Security Lead generates the current access list for all systems.
   - Each manager reviews access for their direct reports and confirms: (a) the user is still active, (b) the assigned role is still appropriate, (c) no excessive permissions exist.
   - The CTO reviews privileged accounts and service accounts.
   - Identified discrepancies are remediated within 5 business days.
4. **Sign-off**: Each reviewer provides written sign-off confirming the review is complete and accurate.
5. **Documentation**: Review records, including the access lists, reviewer comments, remediation actions, and sign-offs, are retained for 3 years.

### 5.7 Service Account Management

1. Every service account must have a designated human owner documented in the service account registry.
2. Service account credentials must be stored in a secrets management system, never in source code or configuration files.
3. Service account permissions follow the same least privilege principle as human accounts.
4. Service account credentials are rotated at least every 90 days.
5. Service accounts are included in quarterly access reviews.
6. Unused service accounts are disabled within 30 days of becoming inactive.

### 5.8 API Key Management

#### 5.8.1 Questo Operational API Keys

- API keys for LLM providers (OpenAI, Google Gemini, Anthropic) used in development and testing are stored in environment variables or a secrets manager, never in source code.
- GitHub secret scanning is enabled on all repositories to detect accidental key commits.
- API keys are rotated immediately if any suspected exposure occurs and at least every 90 days.

#### 5.8.2 corteX SDK KeyVault (Tenant API Keys)

- Tenant API keys are stored in per-tenant KeyVault instances (`corteX/security/vault.py`).
- The KeyVault enforces tenant-scoped isolation: no tenant can access another tenant's keys.
- `KeyVault.detect_leak()` monitors all outputs for accidental API key exposure and sanitizes them automatically.
- In on-premises deployments, the customer is responsible for securing the KeyVault storage backend.

### 5.9 GitHub Repository Access Controls

1. All repositories are private by default.
2. Branch protection is enabled on `main` with the following rules:
   - No direct pushes to `main`
   - At least one approved pull request review required
   - All CI/CD status checks must pass (8,082+ automated tests)
   - Force pushes are disabled
   - Branch deletion is restricted
3. Organization-level MFA enforcement is enabled.
4. Deploy keys are scoped to specific repositories and have read-only access unless write is explicitly justified.
5. GitHub Actions secrets are scoped to the minimum required repositories and environments.

### 5.10 Cloud Infrastructure Access Controls

1. Cloud console access requires MFA (see Section 5.2.1).
2. Production and development environments are segregated with separate access controls.
3. Administrative access to production is limited to the CTO and Security Lead.
4. All cloud administrative actions are logged and monitored.
5. Infrastructure-as-Code (IaC) changes follow the same PR review process as application code (POL-003).

### 5.11 PyPI Publishing Access Controls

1. PyPI publishing credentials are held by the CTO only.
2. Publishing requires MFA on the PyPI account.
3. All SDK releases follow the release process defined in POL-003, including version verification and CHANGELOG review.
4. Automated publishing through CI/CD uses API tokens scoped to the `cortex-ai` project only.
5. Publishing actions are logged and verified against the approved release record.

### 5.12 Privileged Access Management

1. Privileged access is limited to the CTO and Security Lead for production systems.
2. All privileged actions are logged with timestamp, user identity, and action details.
3. Privileged access sessions are reviewed during quarterly access reviews.
4. Emergency privileged access may be granted per the exception process in Section 7, with post-incident review required within 48 hours.

## 6. Procedures

### 6.1 New User Provisioning Procedure

1. Manager completes the Access Request Form specifying: user name, role, required systems, start date, and business justification.
2. CTO reviews and approves the request.
3. Security Lead creates accounts on approved systems per the role definition in Section 5.3.
4. Security Lead sends onboarding instructions to the new user, including MFA enrollment requirements.
5. New user completes MFA enrollment on all provisioned systems within 48 hours.
6. Security Lead verifies MFA enrollment and documents provisioning in the access log.

### 6.2 Access Modification Procedure

1. Manager submits an Access Modification Request specifying: user name, current role, requested changes, and justification.
2. CTO approves modifications that involve role changes or additional system access.
3. Security Lead implements changes within 2 business days.
4. Changes are documented in the access log with the approval reference.

### 6.3 Quarterly Access Review Procedure

1. Security Lead exports current user access lists from all systems by the first business day of the review month.
2. Access lists are distributed to managers by the 3rd business day.
3. Managers complete their reviews and return signed attestations by the 10th business day.
4. Security Lead compiles findings and remediates discrepancies by the 15th business day.
5. CTO reviews the aggregated report and signs off by the 20th business day.
6. All documentation is filed in the access review archive.

### 6.4 Emergency Access Procedure

1. In the event of a security incident requiring emergency access, the requesting party contacts the CTO directly.
2. The CTO may grant temporary elevated access verbally, with written documentation to follow within 24 hours.
3. Emergency access is limited to the minimum required and is automatically revoked within 24 hours unless explicitly renewed.
4. All emergency access events are reviewed during the next quarterly access review and documented in the incident record (POL-004).

## 7. Exceptions

1. Exceptions to this policy require written approval from the CTO.
2. Exceptions involving access to RESTRICTED data require CEO approval.
3. Exception requests must include: the specific policy provision, business justification, compensating controls, and requested duration (maximum 90 days for access control exceptions).
4. Active exceptions are reviewed during quarterly access reviews.
5. All exceptions are documented in the Exception Register with approval, compensating controls, and expiration date.
6. No exceptions are permitted for: MFA requirements (Section 5.2.1), the prohibition on shared accounts (Section 5.1), or the 24-hour deprovisioning requirement (Section 5.5).

## 8. Enforcement

- Violations of this policy may result in immediate suspension of access pending investigation.
- Sharing credentials or disabling MFA is considered a serious violation resulting in disciplinary action up to and including termination.
- Unauthorized access attempts are logged, investigated per POL-004, and may be referred to law enforcement.
- The Security Lead monitors for policy violations through access logs, failed authentication alerts, and anomaly detection.
- Repeated violations result in mandatory remedial training and may result in permanent access restrictions.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-003 | Change Management Policy |
| POL-004 | Incident Response Plan |
| POL-005 | Risk Assessment and Mitigation Policy |
| POL-008 | Data Classification Policy |
| POL-011 | Encryption Policy |
| POL-012 | Password Policy |
| POL-016 | Remote Access Policy |
| POL-018 | Software Development Lifecycle Policy |
| POL-020 | Workstation Security Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
