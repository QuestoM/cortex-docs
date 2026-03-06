# Remote Access Policy

**Document ID**: POL-016
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Remote Access Policy defines the requirements for secure remote access to Questo Ltd systems, data, and infrastructure. As a remote-first organization where all engineering work on the corteX AI Agent SDK occurs from distributed locations, secure remote access is fundamental to Questo Ltd's operational security posture.

This policy establishes requirements for VPN usage, multi-factor authentication, endpoint security, and acceptable remote access practices to protect the corteX codebase, customer data, and Questo Ltd business operations.

**SOC 2 Mapping**: CC6.1 (Logical Access), CC6.2 (Access Credentials), CC6.6 (System Boundaries), CC6.7 (Data Transmission)

## 2. Scope

This policy applies to:

- **Personnel**: All employees, contractors, consultants, and any authorized third parties who access Questo Ltd systems remotely.
- **Systems**: All Questo Ltd systems accessible remotely, including: GitHub repositories, cloud infrastructure consoles, CI/CD pipelines, communication platforms, documentation hosting, PyPI publishing, and development environments.
- **Networks**: All networks from which remote access originates, including home networks, co-working spaces, mobile networks, and hotel networks.
- **Devices**: All devices used to access Questo Ltd systems, whether company-issued or personal (BYOD).

## 3. Definitions

| Term | Definition |
|------|-----------|
| **VPN** | Virtual Private Network; an encrypted tunnel used to secure network traffic between a remote device and Questo Ltd infrastructure |
| **MFA** | Multi-Factor Authentication; authentication requiring two or more independent factors |
| **SSO** | Single Sign-On; a centralized authentication mechanism that provides access to multiple systems |
| **BYOD** | Bring-Your-Own-Device; the use of personal devices for company work |
| **Endpoint** | A computing device (laptop, phone, tablet) used to access Questo Ltd systems |
| **Split Tunneling** | A VPN configuration where some traffic goes through the VPN and some goes directly to the internet |
| **Trusted Network** | A network that meets Questo Ltd security requirements (WPA3/WPA2 with strong password, no shared access with untrusted parties) |
| **Untrusted Network** | Any network that does not meet trusted network criteria, including public Wi-Fi, hotel networks, and co-working space networks |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and ensures its implementation
- Approves the VPN solution and configuration standards
- Reviews remote access logs quarterly for anomalies
- Approves remote access for third parties and auditors

### 4.2 Security Lead

- Manages VPN infrastructure and user provisioning
- Monitors remote access logs for unauthorized access attempts and anomalies
- Provisions and revokes VPN certificates and credentials
- Conducts quarterly review of active remote access accounts
- Responds to remote access security incidents

### 4.3 All Personnel

- Use the VPN when connecting from untrusted networks
- Maintain MFA on all remotely accessible systems per POL-002
- Ensure endpoints meet the security requirements defined in POL-020 before connecting
- Report suspected unauthorized remote access immediately per POL-004
- Never share VPN credentials or MFA tokens with any other person

## 5. Policy Statements

### 5.1 Remote Access Principles

1. **MFA Without Exception**: Multi-factor authentication is required on all remotely accessible systems. There are no exceptions to this requirement. Systems that do not support MFA natively must be accessed through a gateway that enforces MFA.
2. **Encryption in Transit**: All remote access connections must use encryption. Minimum standard is TLS 1.2 for HTTPS connections and AES-256 for VPN tunnels.
3. **Endpoint Compliance**: Devices used for remote access must meet the requirements defined in POL-020 (Workstation Security Policy), including full disk encryption, current operating system patches, and active endpoint protection.
4. **Individual Accountability**: Shared remote access accounts are prohibited. Every remote connection must be traceable to an individual user.
5. **Least Privilege**: Remote access is granted only to the systems and data required for the user's job function, consistent with POL-002.

### 5.2 VPN Requirements

#### 5.2.1 When VPN Is Required

VPN usage is mandatory when:

- Connecting from any untrusted network (public Wi-Fi, hotel networks, co-working spaces, mobile hotspots in public locations)
- Accessing cloud infrastructure management consoles
- Performing administrative operations on any Questo Ltd system

VPN usage is strongly recommended but not mandatory when:

- Connecting from a trusted home network to systems that already enforce MFA and TLS (e.g., GitHub, Google Workspace)

#### 5.2.2 VPN Configuration Standards

- **Protocol**: WireGuard or IPsec IKEv2. Legacy protocols (PPTP, L2TP without IPsec) are prohibited.
- **Encryption**: AES-256-GCM minimum for data encryption; RSA-2048 or Ed25519 for key exchange.
- **Authentication**: Certificate-based authentication combined with MFA. Username/password-only VPN authentication is prohibited.
- **Split Tunneling**: Split tunneling is disabled by default. The CTO may approve split tunneling configurations on a case-by-case basis with documented justification and compensating controls.
- **Session Timeout**: VPN sessions timeout after 12 hours of inactivity, requiring re-authentication.
- **Concurrent Sessions**: Each user may have a maximum of 2 concurrent VPN sessions (e.g., laptop and phone).

#### 5.2.3 VPN Credential Management

- VPN certificates are issued individually per user and per device.
- Certificates have a maximum validity of 12 months and must be renewed before expiration.
- Certificates are revoked immediately upon employee departure (within the 24-hour deprovisioning window per POL-002) or upon device loss or compromise.
- The Security Lead maintains a registry of issued VPN certificates with: user, device, issuance date, expiration date, and status.

### 5.3 Remote Access to Specific Systems

| System | Access Method | MFA Required | VPN Required | Additional Controls |
|--------|-------------|-------------|-------------|-------------------|
| GitHub (source code) | HTTPS + SSH | Yes (org-enforced) | Recommended | Branch protection, IP allow-list if available |
| Cloud Console (AWS/Azure/GCP) | HTTPS | Yes (account-enforced) | Yes | Session recording for admin actions |
| CI/CD Pipeline (GitHub Actions) | HTTPS | Yes (via GitHub) | No (SaaS) | Secrets scoped per environment |
| Communication Platforms | HTTPS | Yes | Recommended | DLP controls on file sharing |
| PyPI Publishing | HTTPS | Yes (account-enforced) | Yes | CTO approval required per POL-002 |
| Documentation Hosting Admin | HTTPS | Yes | Recommended | Read-only public access |

### 5.4 Third-Party Remote Access

- Third-party remote access (auditors, consultants, vendors) requires written approval from the CTO.
- Third-party access is provisioned with a defined scope and expiration date (maximum 90 days, renewable with re-approval).
- Third-party endpoints must meet Questo Ltd security requirements or access must be through a managed virtual desktop.
- All third-party remote access is logged and monitored.
- Third-party access is revoked within 24 hours of engagement completion.

### 5.5 Remote Access Monitoring

- All VPN connections are logged with: user identity, source IP, connection time, disconnection time, and data volume.
- Failed authentication attempts are logged and trigger alerts after 5 consecutive failures.
- Connections from unusual geographic locations trigger an alert for Security Lead review.
- Remote access logs are retained for 12 months per POL-017 (Logging and Monitoring Policy).
- Quarterly access log reviews are conducted to identify anomalies, inactive accounts, and policy compliance.

### 5.6 Prohibited Remote Access Practices

The following practices are strictly prohibited:

1. Sharing VPN credentials, certificates, or MFA tokens with any other person.
2. Using personal VPN services (consumer VPN providers) as a substitute for the company VPN.
3. Connecting to Questo Ltd systems from publicly shared or kiosk computers.
4. Storing Questo Ltd credentials in unencrypted files, browser autofill on shared devices, or sticky notes.
5. Disabling endpoint protection software while connected to Questo Ltd systems.
6. Using remote desktop tools (RDP, VNC, TeamViewer) to access Questo Ltd systems unless approved by the CTO.

## 6. Procedures

### 6.1 Remote Access Provisioning

1. The employee's manager submits a remote access request as part of the onboarding process or via an access modification request per POL-002.
2. The Security Lead verifies that the user's endpoint meets POL-020 requirements.
3. The Security Lead issues VPN credentials (certificate + MFA enrollment) and provides configuration instructions.
4. The user confirms successful VPN connectivity and MFA functionality.
5. The provisioning is documented in the access management log.

### 6.2 Remote Access Deprovisioning

1. Upon employee departure notification, VPN certificates are revoked within the deprovisioning timeline defined in POL-002 (immediately for involuntary departures, within 24 hours for voluntary departures).
2. The Security Lead verifies revocation by confirming the certificate appears on the revocation list and connection attempts are rejected.
3. Revocation is documented in the access management log.

### 6.3 Quarterly Remote Access Review

1. The Security Lead generates a report of all active VPN users, certificates, and connection logs.
2. The report is cross-referenced against the current employee roster and authorized third-party access list.
3. Inactive accounts (no VPN connection in 90 days) are flagged for review and disabled if no business justification is provided.
4. Findings are documented and presented to the CTO as part of the quarterly access review.

## 7. Exceptions

1. Exceptions to this policy require written approval from the CTO.
2. Exception requests must include: the specific policy provision, business justification, compensating controls, and requested duration (maximum 90 days).
3. No exceptions are permitted for: MFA requirements, the prohibition on shared accounts, or the prohibition on accessing systems from public kiosk computers.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Violations of this policy may result in immediate suspension of remote access pending investigation.
- Sharing VPN credentials or MFA tokens is a serious violation resulting in disciplinary action up to and including termination.
- Repeated failure to use VPN when required will result in mandatory remedial training and may result in access restrictions.
- The Security Lead monitors compliance through VPN connection logs and periodic endpoint compliance checks.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-004 | Incident Response Plan |
| POL-011 | Encryption Policy |
| POL-012 | Password Policy |
| POL-015 | Physical Security Policy |
| POL-017 | Logging and Monitoring Policy |
| POL-020 | Workstation Security Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
