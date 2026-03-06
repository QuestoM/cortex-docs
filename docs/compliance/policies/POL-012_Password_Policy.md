# Password Policy

**Document ID**: POL-012
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Password Policy defines the requirements for creating, managing, protecting, and rotating passwords and other authentication credentials used to access Questo Ltd information systems. It establishes minimum complexity standards, multi-factor authentication requirements, and credential management practices that protect the corteX AI Agent SDK platform and all associated systems from unauthorized access through credential compromise.

This policy applies to all human-user passwords and is complementary to the API key management requirements defined in POL-002 and POL-011 for non-human credentials.

**SOC 2 Mapping**: CC6.1 (Logical Access), CC6.2 (Authentication), CC6.3 (Access Enforcement)

## 2. Scope

This policy applies to:

- **Passwords**: All passwords used to authenticate to Questo Ltd systems, including GitHub, cloud infrastructure, email, communication platforms, CI/CD tools, PyPI, and VPN.
- **Passphrases**: SSH key passphrases, GPG key passphrases, and encryption passphrases.
- **PINs**: Device unlock PINs where applicable.
- **MFA**: Multi-factor authentication tokens, recovery codes, and hardware security keys.
- **Personnel**: All employees, contractors, and third parties with access to Questo systems.
- **Systems**: All systems requiring authentication that are used for Questo business.

This policy does not cover API keys, service account tokens, or cryptographic keys, which are governed by POL-002 (service accounts), POL-011 (encryption keys), and the KeyVault implementation in `corteX/security/vault.py`.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Password** | A secret string of characters used to verify the identity of a user |
| **Passphrase** | A longer password consisting of multiple words, typically used for key protection |
| **MFA** | Multi-Factor Authentication; requires two or more independent factors (something you know, something you have, something you are) |
| **TOTP** | Time-based One-Time Password; a standard MFA method generating time-limited codes |
| **Hardware Security Key** | A physical device (e.g., YubiKey) used as an authentication factor |
| **Password Manager** | An approved application for securely generating, storing, and retrieving passwords |
| **Credential Stuffing** | An attack using stolen username/password pairs from other breaches |
| **Brute Force** | An attack systematically trying all possible password combinations |
| **Account Lockout** | Temporarily disabling an account after repeated failed authentication attempts |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and ensures system configurations enforce password requirements
- Approves the list of approved password managers and MFA methods
- Reviews password-related security incidents and adjusts policy as needed

### 4.2 Security Lead

- Configures password policies on all systems to enforce the requirements in this policy
- Monitors for credential-related security events (brute force attempts, credential leaks)
- Conducts periodic audits of password policy enforcement across systems
- Manages the onboarding process for MFA enrollment

### 4.3 All Personnel

- Create and maintain passwords that meet the requirements of this policy
- Enroll in MFA on all Questo systems within 48 hours of account provisioning
- Use an approved password manager for credential storage
- Report suspected credential compromise immediately per POL-004
- Never share passwords or MFA credentials with anyone

## 5. Policy Statements

### 5.1 Password Complexity Requirements

All passwords for Questo systems must meet the following minimum requirements:

| Requirement | Standard |
|-------------|----------|
| **Minimum length** | 12 characters |
| **Recommended length** | 16+ characters (passphrases encouraged) |
| **Character classes** | At least 3 of 4: uppercase letter, lowercase letter, number, special character |
| **Dictionary words** | Must not be a common dictionary word, name, or predictable pattern |
| **Personal information** | Must not contain the user's name, email, username, or company name |
| **Breached passwords** | Must not appear in known breached password databases (checked at creation where system supports it) |
| **Sequential/repeated characters** | Must not contain 3 or more sequential or repeated characters (e.g., "aaa", "123", "abc") |

#### 5.1.1 Passphrase Alternative

Passphrases of 20 or more characters consisting of 4 or more unrelated words are an acceptable alternative to complex passwords. Passphrases provide superior security through length while being easier to remember. Example format: "correct horse battery staple" (but never use published examples).

### 5.2 Multi-Factor Authentication

#### 5.2.1 MFA Requirement

MFA is **mandatory** on all Questo systems without exception. This requirement is non-negotiable and no exceptions are granted per POL-002 Section 7.

#### 5.2.2 Approved MFA Methods

| Method | Security Level | Approved For |
|--------|---------------|-------------|
| **Hardware security key** (FIDO2/WebAuthn) | Highest | All systems (preferred method) |
| **TOTP authenticator app** | High | All systems |
| **Push notification** (from approved authenticator) | High | Systems that support it |
| **SMS-based OTP** | Acceptable only as fallback | Legacy systems only; not preferred due to SIM swap risk |

#### 5.2.3 MFA Enrollment

1. New personnel must enroll in MFA on all provisioned systems within 48 hours of account creation.
2. At least two MFA methods should be registered where supported (primary and backup).
3. MFA recovery codes must be generated and stored in the approved password manager.
4. The Security Lead verifies MFA enrollment as part of the provisioning process (POL-002).

#### 5.2.4 MFA Recovery

1. If an MFA device is lost or unavailable, the user contacts the Security Lead for identity verification and MFA reset.
2. Identity verification requires confirmation from the user's manager and at least one additional verification step (video call, known contact number).
3. A new MFA device must be enrolled immediately after recovery.
4. MFA recovery events are logged and reviewed during quarterly access reviews.

### 5.3 Password Management

#### 5.3.1 Password Manager Requirement

All personnel must use an approved password manager for storing and generating passwords for Questo systems. The password manager:

1. Must use strong encryption (AES-256) for the password vault.
2. Must support MFA for vault access.
3. Must support secure password generation meeting the complexity requirements in Section 5.1.
4. Must be approved by the CTO and maintained on the approved tools list.

**The master password for the password manager must be a strong passphrase of at least 20 characters that is memorized, not written down or stored digitally outside the user's memory.**

#### 5.3.2 Prohibited Password Practices

The following practices are strictly prohibited:

1. Writing passwords on paper, sticky notes, whiteboards, or physical media
2. Storing passwords in plaintext files, spreadsheets, emails, or chat messages
3. Sharing passwords with any other person, including managers and IT support
4. Using the same password across multiple systems (each system must have a unique password)
5. Using a Questo password for personal accounts or vice versa
6. Storing passwords in web browser built-in password managers (use the approved password manager instead)
7. Transmitting passwords via email, chat, or any unencrypted channel
8. Using password hints that reveal the password

### 5.4 Password Rotation

| Account Type | Rotation Frequency | Trigger for Immediate Rotation |
|-------------|-------------------|-------------------------------|
| Standard user accounts | Every 365 days | Suspected compromise, security incident |
| Privileged accounts (admin) | Every 180 days | Suspected compromise, security incident, personnel change |
| Password manager master password | Every 365 days | Suspected compromise |
| System/service passwords | Every 90 days | Suspected compromise, personnel departure with access |

**Note**: Per NIST SP 800-63B (Digital Identity Guidelines, Section 5.1.1.2) guidance, mandatory periodic rotation is less effective than strong initial passwords combined with breach detection and compromised-credential checks. Questo follows NIST SP 800-63B by emphasizing password strength, breach-list checking at creation, and compromise-driven rotation. The rotation schedule above represents maximum intervals; event-triggered rotation (suspected compromise, personnel departure) is the primary protection mechanism.

#### 5.4.1 Rotation Rules

1. New passwords must not match any of the previous 12 passwords for the same account.
2. Password changes must use the system's password change mechanism (not an administrator reset, except for recovery).
3. Temporary passwords issued by administrators must be changed on first use and expire within 24 hours if not used.
4. Password rotation compliance is monitored by the Security Lead.

### 5.5 Account Lockout

| Parameter | Standard |
|-----------|----------|
| **Failed attempts before lockout** | 5 consecutive failed attempts |
| **Lockout duration** | 30 minutes (automatic unlock) or manual unlock by Security Lead |
| **Failed attempt counter reset** | After successful authentication |
| **Notification** | Security Lead is alerted after 3 failed attempts on privileged accounts |

Account lockout events are logged and reviewed. Patterns of lockout events across multiple accounts may indicate a credential stuffing or brute force attack and trigger the incident response process (POL-004).

### 5.6 Session Management

| Parameter | Standard |
|-----------|----------|
| **Web session idle timeout** | 30 minutes maximum |
| **Administrative session idle timeout** | 15 minutes maximum |
| **Administrative session absolute timeout** | 8 hours |
| **Workstation screen lock** | 5 minutes of inactivity (POL-020) |
| **Session tokens** | Must be invalidated on logout |
| **Concurrent sessions** | Limited to 3 per user per application |

### 5.7 Service Account Credentials

Service accounts (non-human accounts used by CI/CD pipelines, automated processes, and system integrations) have unique credential requirements:

1. **No Interactive Login**: Service accounts must not support interactive login. They must be restricted to API-based or token-based authentication only.
2. **Unique Credentials**: Each service account must have a unique credential. Sharing credentials between service accounts or between service and human accounts is prohibited.
3. **Credential Storage**: Service account credentials must be stored in a secrets management system (GitHub Actions Secrets, cloud provider secrets manager). They must never be stored in source code, configuration files committed to version control, or unencrypted files.
4. **Rotation Schedule**: Service account credentials are rotated at least every 90 days and immediately upon suspected compromise or departure of personnel with access to the credential.
5. **Scope Limitation**: Service account credentials must be scoped to the minimum permissions required. For example, PyPI publishing tokens are scoped to the `cortex-ai` project only.
6. **Ownership**: Every service account must have a designated human owner who is accountable for its credential management and documented in the service account registry.
7. **Review**: Service accounts and their credentials are included in the quarterly access review per POL-002.
8. **Decommissioning**: Unused service accounts are disabled within 30 days. Credentials for decommissioned service accounts are revoked and destroyed.

### 5.8 API Key Management and KeyVault

API keys used for LLM provider access and other integrations require specific credential handling beyond standard password practices:

#### 5.8.1 Questo Operational API Keys

1. API keys for LLM providers (OpenAI, Google Gemini, Anthropic) used in development and testing are classified as RESTRICTED per POL-008.
2. API keys are stored in environment variables or approved secrets management systems, never in source code.
3. GitHub secret scanning is enabled on all repositories to detect accidental key commits per POL-002.
4. API keys are rotated at least every 90 days and immediately if any suspected exposure occurs.
5. Access to API keys is limited to personnel with a documented business need per POL-002.

#### 5.8.2 corteX SDK KeyVault (Tenant API Keys)

The `KeyVault` module (`corteX/security/vault.py`) provides enterprise-grade credential management for tenant API keys within the corteX SDK:

1. **Encrypted Storage**: Tenant API keys are encrypted using Fernet (AES-128-CBC + HMAC-SHA256) with a PBKDF2-derived 256-bit master key (600,000 iterations) per POL-011.
2. **Per-Tenant Isolation**: Each tenant's KeyVault derives its own encryption key from the tenant_id. No cross-tenant key access is possible.
3. **Memory-Only**: Encrypted keys reside in memory only. Keys never touch disk, log output, or environment variables.
4. **Atomic Rotation**: `KeyVault.rotate()` atomically replaces old key material, zeroing the old encrypted value before storing the new one.
5. **Leak Detection**: `KeyVault.detect_leak()` monitors all agent outputs for accidental API key exposure and triggers alerts per POL-004.
6. **No Fallback**: KeyVault raises `ValueError` if a key is not found. It does not fall back to environment variables or any other source, preventing credential leakage through misconfiguration.
7. **GDPR Erasure**: `KeyVault.destroy()` supports GDPR right-to-erasure by securely erasing all tenant keys.

#### 5.8.3 Password Hashing for Stored Credentials

Any system that stores user passwords must use the following requirements:

1. Passwords must be stored using an approved adaptive hashing algorithm: bcrypt, scrypt, or Argon2id.
2. Plaintext password storage is strictly prohibited.
3. The cost factor for the hashing algorithm must follow current OWASP guidance (bcrypt work factor 12+, Argon2id with minimum 19 MiB memory, 2 iterations).
4. Password hash values are classified as CONFIDENTIAL per POL-008.
5. Hash comparison must use constant-time comparison functions to prevent timing attacks.

### 5.9 Credential Monitoring

1. The Security Lead monitors publicly disclosed credential breaches for Questo email domains and usernames.
2. If Questo credentials are found in a breach disclosure, affected users are required to change their passwords immediately.
3. GitHub secret scanning monitors all repositories for accidentally committed credentials.
4. Alerts from credential monitoring are treated as P1 incidents per POL-004.

## 6. Procedures

### 6.1 Password Creation Procedure

1. Use the approved password manager's password generator to create a new password meeting the requirements in Section 5.1.
2. Alternatively, create a passphrase of 4+ unrelated words totaling 20+ characters.
3. Store the password in the approved password manager.
4. Do not reuse passwords across systems.
5. Enroll MFA on the system if not already enrolled.

### 6.2 MFA Enrollment Procedure

1. Access the MFA settings page for the target system.
2. Select the preferred MFA method (hardware key preferred, then TOTP).
3. Complete the enrollment process per the system's instructions.
4. Generate and securely store recovery codes in the password manager.
5. Enroll a backup MFA method where supported.
6. Notify the Security Lead to confirm enrollment verification.

### 6.3 Password Compromise Response Procedure

1. If a password is suspected of being compromised, change it immediately on the affected system.
2. If the same or similar password was used on other systems (a policy violation), change those passwords immediately as well.
3. Report the suspected compromise to the Security Lead per POL-004.
4. Review account activity logs for the affected account for unauthorized access.
5. If unauthorized access is confirmed, the Security Lead escalates per the incident response plan.

### 6.4 Password Policy Audit Procedure

1. The Security Lead audits password policy enforcement quarterly on all systems.
2. The audit verifies: minimum length enforcement, complexity requirements, lockout policy configuration, MFA enforcement, and rotation compliance.
3. Systems not meeting policy requirements are reported to the CTO for remediation.
4. Audit results are documented and retained for 3 years.

## 7. Exceptions

1. No exceptions are granted for the MFA requirement (Section 5.2.1).
2. No exceptions are granted for the prohibition on password sharing (Section 5.3.2).
3. Systems that do not support the minimum 12-character password length must have a compensating control (MFA is mandatory, plus IP restriction where available) documented and approved by the CTO.
4. The 365-day rotation maximum may be extended to 730 days for standard accounts if the system enforces MFA and continuous credential monitoring is in place, with CTO approval.
5. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Sharing passwords is a serious policy violation resulting in disciplinary action up to and including termination.
- Failure to enroll in MFA within the required timeframe results in account suspension until enrollment is complete.
- Storing passwords in plaintext (outside the approved password manager) is a policy violation.
- Failure to change a password after a confirmed or suspected compromise is a serious policy violation.
- The Security Lead monitors compliance and reports violations to the CTO.
- Repeated violations result in mandatory remedial training and may result in access restrictions.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-004 | Incident Response Plan |
| POL-008 | Data Classification Policy |
| POL-009 | Vendor Management Policy |
| POL-010 | Acceptable Use Policy |
| POL-011 | Encryption Policy |
| POL-020 | Workstation Security Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
