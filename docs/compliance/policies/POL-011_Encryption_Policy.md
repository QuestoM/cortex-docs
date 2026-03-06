# Encryption Policy

**Document ID**: POL-011
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Encryption Policy defines the cryptographic standards, key management practices, and encryption requirements for protecting data at rest, in transit, and during processing at Questo Ltd. It establishes the approved algorithms, minimum key lengths, key lifecycle management procedures, and implementation requirements that apply to the corteX AI Agent SDK and all Questo operations.

This policy covers the specific cryptographic implementations within the corteX SDK, including the Fernet encryption (AES-128-CBC + HMAC-SHA256, with PBKDF2-derived 256-bit master key) in KeyVault (`corteX/security/vault.py`), Ed25519 digital signatures for license validation, TLS requirements for all network communications, and PBKDF2 key derivation. It ensures that all cryptographic controls meet industry standards and SOC 2 requirements.

**SOC 2 Mapping**: CC6.1 (Logical Access), CC6.6 (Encryption), CC6.7 (Transmission Security)

## 2. Scope

This policy applies to:

- **Data at Rest**: All stored data classified as INTERNAL or higher per POL-008, including database contents, file system data, backups, API keys in KeyVault, and encryption keys.
- **Data in Transit**: All data transmitted over networks, including API communications with LLM providers, internal service communications, web traffic, and email.
- **Data in Processing**: Cryptographic operations within the corteX SDK, including key derivation, encryption/decryption, and digital signature operations.
- **Key Management**: All cryptographic keys used by Questo Ltd or within the corteX SDK, including symmetric keys, asymmetric key pairs, API keys, and signing keys.
- **Systems**: All Questo information systems, development environments, production infrastructure, and the corteX SDK itself.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Fernet** | A symmetric encryption specification using AES-128-CBC with HMAC-SHA256 for authentication. The `cryptography` library implementation uses AES-128-CBC, but KeyVault derives 256-bit keys via PBKDF2 for the key derivation layer. |
| **AES-256** | Advanced Encryption Standard with 256-bit key length |
| **PBKDF2** | Password-Based Key Derivation Function 2; used to derive encryption keys from passwords or identifiers |
| **Ed25519** | An elliptic curve digital signature algorithm providing 128-bit security |
| **TLS** | Transport Layer Security; protocol for encrypting data in transit |
| **HMAC-SHA256** | Hash-based Message Authentication Code using SHA-256; provides data integrity and authenticity |
| **Key Rotation** | The process of replacing a cryptographic key with a new key |
| **Cryptographic Erasure** | Destroying encrypted data by destroying the encryption key |
| **HSM** | Hardware Security Module; a physical device for secure key storage and cryptographic operations |
| **Key Derivation** | The process of generating cryptographic keys from a source value (password, identifier, or master key) |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and approves all cryptographic standards and implementations
- Reviews and approves changes to cryptographic algorithms or key lengths
- Ensures cryptographic controls are implemented correctly in the corteX SDK
- Approves key management procedures and key custodian assignments

### 4.2 Security Lead (Key Custodian)

- Manages the lifecycle of all operational encryption keys
- Executes key rotation procedures per the defined schedule
- Maintains the key inventory with current status and next rotation dates
- Ensures encryption is correctly implemented across all systems
- Monitors for deprecated algorithms and recommends updates

### 4.3 Engineering Team

- Implements cryptographic controls in the corteX SDK per this policy
- Uses only approved cryptographic libraries and algorithms
- Never implements custom cryptographic algorithms
- Reports any cryptographic weaknesses or concerns to the Security Lead

### 4.4 All Personnel

- Use only approved encryption tools for protecting Questo data
- Never circumvent or disable encryption controls
- Report suspected key compromises immediately per POL-004

## 5. Policy Statements

### 5.1 Approved Cryptographic Algorithms

Only the following algorithms are approved for use in Questo operations and the corteX SDK:

#### 5.1.1 Symmetric Encryption

| Algorithm | Key Length | Use Case | Implementation |
|-----------|-----------|----------|---------------|
| **Fernet (AES-128-CBC + HMAC-SHA256)** | 256-bit master key (128-bit AES + 128-bit HMAC) | Data at rest (KeyVault) | `cryptography` library Fernet with PBKDF2-derived key |
| **AES-256-GCM** | 256-bit | Data at rest (alternative to CBC) | `cryptography` library AESGCM |
| **ChaCha20-Poly1305** | 256-bit | Data at rest (alternative for non-AES-NI hardware) | `cryptography` library |

#### 5.1.2 Asymmetric Encryption and Signatures

| Algorithm | Key Length | Use Case | Implementation |
|-----------|-----------|----------|---------------|
| **Ed25519** | 256-bit (128-bit security) | License validation, code signing | `cryptography` library Ed25519 |
| **RSA** | 4096-bit minimum | Legacy systems only (new implementations use Ed25519) | `cryptography` library RSA |
| **ECDSA P-256** | 256-bit | TLS certificates (when Ed25519 is not supported) | TLS library |

#### 5.1.3 Hashing and Key Derivation

| Algorithm | Use Case | Parameters |
|-----------|----------|-----------|
| **PBKDF2-HMAC-SHA256** | Key derivation from tenant_id in KeyVault | 600,000 iterations minimum (per OWASP 2024 guidance) |
| **SHA-256** | Data integrity, non-security hashing | N/A |
| **SHA-512** | Extended hash requirements | N/A |
| **HMAC-SHA256** | Message authentication (included in Fernet) | N/A |

#### 5.1.4 Prohibited Algorithms

The following algorithms are explicitly prohibited and must not be used in any Questo system or corteX code:

- MD5 (for any security purpose)
- SHA-1 (for any security purpose)
- DES and 3DES
- RC4
- RSA with key length below 2048 bits
- Any custom or proprietary cryptographic algorithm
- TLS 1.0 and TLS 1.1

### 5.2 Encryption at Rest

#### 5.2.1 corteX KeyVault Encryption

The `KeyVault` module (`corteX/security/vault.py`) provides encryption at rest for tenant API keys:

1. **Algorithm**: Fernet (AES-128-CBC + HMAC-SHA256) from the `cryptography` library.
2. **Key Derivation**: PBKDF2-HMAC-SHA256 with 600,000 iterations, using the tenant_id combined with a static salt prefix (`corteX.security.vault.v1:`) to derive a unique encryption key per tenant.
3. **Key Isolation**: Each tenant's KeyVault instance derives its own encryption key. No shared encryption keys exist across tenants.
4. **Memory-Only Storage**: Encrypted keys are stored in memory only. Keys never touch disk or log output.
5. **Atomic Rotation**: When an API key is rotated, the old key material is zeroed before the new encrypted value is stored.
6. **Leak Detection**: `KeyVault.detect_leak()` scans all outputs for accidental API key exposure and sanitizes them.
7. **Maximum Key Length**: A 4,096-byte limit (`_MAX_KEY_LENGTH`) prevents abuse of the storage mechanism.

#### 5.2.2 Data Storage Encryption

| Data Type | Encryption Requirement | Minimum Standard |
|-----------|----------------------|-----------------|
| RESTRICTED data | Mandatory | AES-256 |
| CONFIDENTIAL data | Mandatory | AES-256 |
| INTERNAL data | Required when feasible | AES-256 preferred; AES-128 acceptable |
| PUBLIC data | Not required | N/A |
| Backups | Mandatory (all classifications) | AES-256 |
| Laptop full disk encryption | Mandatory (all company devices) | AES-256 (BitLocker, FileVault, LUKS) |
| Cloud storage | Mandatory | Provider-managed encryption + customer-managed keys for RESTRICTED |

#### 5.2.3 Database Encryption

- All databases storing CONFIDENTIAL or RESTRICTED data must use Transparent Data Encryption (TDE) or equivalent.
- Column-level encryption is required for individual RESTRICTED fields (e.g., API keys) when stored in databases.
- Database backup encryption follows the same standards as the source data.

### 5.3 Encryption in Transit

#### 5.3.1 TLS Requirements

| Use Case | Minimum Version | Preferred Version | Certificate Requirement |
|----------|----------------|-------------------|----------------------|
| corteX documentation site | TLS 1.2 | TLS 1.3 | Valid CA-signed certificate |
| corteX server API (optional) | TLS 1.3 | TLS 1.3 | Valid CA-signed certificate |
| LLM provider API calls | TLS 1.2 | TLS 1.3 | Provider-managed (verified) |
| Internal service communication | TLS 1.2 | TLS 1.3 | Valid CA-signed or internal CA |
| Email (in transit) | TLS 1.2 | TLS 1.3 | Provider-managed |
| VPN | TLS 1.2 or IPsec | TLS 1.3 or IKEv2 | Per VPN configuration |

#### 5.3.2 TLS Configuration Standards

1. **Cipher Suites**: Only cipher suites providing forward secrecy are permitted. ECDHE-based key exchange is preferred.
2. **Certificate Management**: TLS certificates are renewed automatically via Let's Encrypt or equivalent. Certificates expiring within 30 days trigger an alert.
3. **Certificate Pinning**: Not required for general use. May be implemented for specific high-security integrations at CTO discretion.
4. **HSTS**: HTTP Strict Transport Security is enabled on all Questo web services with a minimum max-age of 1 year.

#### 5.3.3 LLM Provider API Communication

All communication with LLM providers (OpenAI, Google Gemini, Anthropic) must use TLS. The corteX SDK enforces HTTPS for all provider API endpoints. The `corteX/core/` LLM provider modules do not support HTTP (unencrypted) connections.

### 5.4 Ed25519 License Validation

The corteX SDK uses Ed25519 digital signatures for license validation:

1. **Signing Key**: A 256-bit Ed25519 private key used to sign license files. This key is classified as RESTRICTED.
2. **Verification Key**: The corresponding Ed25519 public key, embedded in the SDK for offline license verification. This key is PUBLIC.
3. **Key Storage**: The private signing key is stored in an encrypted backup with split custody (Section 5.5.3). It is loaded into memory only during license generation.
4. **Verification Process**: The SDK verifies license authenticity by checking the Ed25519 signature against the embedded public key. This operates entirely offline with no network dependency.
5. **Key Rotation**: If the signing key is compromised, a new key pair is generated and a new SDK version is released with the updated public key. Existing licenses remain valid until their expiration.

### 5.5 Key Management

#### 5.5.1 Key Lifecycle

| Phase | Procedure |
|-------|-----------|
| **Generation** | Keys are generated using cryptographically secure random number generators from the `cryptography` library. Never use `random` module for key material. |
| **Distribution** | Keys are distributed through encrypted channels only. Never transmit keys in plaintext. |
| **Storage** | Keys are stored encrypted, with access limited to authorized key custodians. RESTRICTED classification applies to all signing and encryption master keys. |
| **Use** | Keys are used only for their designated purpose. Encryption keys are not used for signing and vice versa. |
| **Rotation** | Keys are rotated per the schedule in Section 5.5.2 or immediately upon suspected compromise. |
| **Revocation** | Compromised keys are revoked immediately. Dependent systems are updated to reject the compromised key. |
| **Destruction** | Retired keys are securely destroyed through cryptographic erasure or secure deletion. Destruction is documented. |

#### 5.5.2 Key Rotation Schedule

| Key Type | Rotation Frequency | Trigger for Immediate Rotation |
|----------|-------------------|-------------------------------|
| LLM provider API keys (Questo-held) | Every 90 days | Suspected compromise, employee departure with access |
| Cloud provider credentials | Every 90 days | Suspected compromise |
| GitHub Actions secrets | Every 90 days | Suspected compromise |
| PyPI publishing tokens | Every 180 days | Suspected compromise |
| Ed25519 license signing key | On compromise only | Suspected compromise |
| PBKDF2 salt prefix (KeyVault) | On major version only | Cryptographic weakness discovered |
| TLS certificates | Automated renewal (90 days for Let's Encrypt) | Compromise, algorithm deprecation |

#### 5.5.3 Split Custody for RESTRICTED Keys

The Ed25519 license signing key and any future master encryption keys are managed under split custody:

1. The key is encrypted and split into two shares using a threshold scheme.
2. Each share is stored in a separate secure location, accessible to a different designated custodian.
3. Both custodians must participate to reconstruct the key.
4. Custodian assignments are documented and reviewed during quarterly access reviews.
5. If a custodian departs, their share is re-generated with a new custodian immediately.

### 5.6 Cryptographic Library Standards

1. **Approved Library**: The `cryptography` Python library is the approved cryptographic library for the corteX SDK. It provides audited, well-maintained implementations of all approved algorithms.
2. **No Custom Cryptography**: Custom implementations of cryptographic algorithms are strictly prohibited. All cryptographic operations must use the approved library.
3. **Library Updates**: The `cryptography` library is updated promptly when security patches are released. It is classified as a Critical dependency per POL-009.
4. **Version Pinning**: The minimum acceptable version of the `cryptography` library is maintained in the project's dependency specification and updated per the change management process (POL-003).

## 6. Procedures

### 6.1 Key Generation Procedure

1. Determine the key type and algorithm per the approved list (Section 5.1).
2. Generate the key using the `cryptography` library's secure random generation.
3. Record the key in the key inventory with: key identifier, algorithm, purpose, generation date, custodian, and next rotation date.
4. Store the key per the classification requirements (Section 5.5.1).
5. For RESTRICTED keys: implement split custody per Section 5.5.3.

### 6.2 Key Rotation Procedure

1. Generate a new key per Section 6.1.
2. Update all systems using the old key to use the new key.
3. For KeyVault tenant keys: rotation is automatic through the `KeyVault.rotate()` method, which atomically replaces the old encrypted value.
4. Verify that the new key is operational by testing encryption/decryption or signing/verification.
5. Securely destroy the old key material.
6. Update the key inventory with the rotation date and new next-rotation date.
7. Document the rotation in the change log.

### 6.3 Key Compromise Response Procedure

1. Report the suspected compromise immediately per POL-004.
2. Revoke the compromised key immediately.
3. Generate a replacement key per Section 6.1.
4. Assess the impact of the compromise: what data may have been exposed, what operations may have been forged.
5. For LLM API keys: notify the LLM provider and rotate the key on their platform. For BYOK keys: notify the affected tenant immediately.
6. For the Ed25519 signing key: generate a new key pair, publish a new SDK version with the updated public key, and issue a security advisory.
7. Document the incident and response per POL-004 procedures.

### 6.4 Annual Cryptographic Review

1. The CTO reviews all cryptographic implementations annually.
2. The review assesses: algorithm currency (are any algorithms approaching deprecation?), key length adequacy, PBKDF2 iteration count (adjust upward as hardware improves), key rotation compliance, and industry guidance updates.
3. Findings are documented and acted upon through the change management process (POL-003).
4. The PBKDF2 iteration count in KeyVault (`_PBKDF2_ITERATIONS`) is reviewed against current OWASP guidance and adjusted upward as needed.

## 7. Exceptions

1. Use of TLS 1.2 (instead of TLS 1.3) is permitted when connecting to external services that do not yet support TLS 1.3, provided the cipher suite uses forward secrecy.
2. Use of RSA (instead of Ed25519) is permitted for compatibility with systems that do not support Ed25519, with a minimum key length of 4096 bits.
3. All other exceptions to approved algorithms or key management procedures require CTO approval with documented justification and compensating controls.
4. No exceptions are permitted for: the prohibition on custom cryptography, the split custody requirement for RESTRICTED keys, or the immediate rotation requirement upon suspected compromise.
5. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Use of prohibited cryptographic algorithms in the corteX SDK or Questo systems is a serious policy violation. Code review (POL-003) must catch algorithm violations before merge.
- Failure to rotate keys per the defined schedule is a policy violation.
- Storing encryption keys or API keys in source code, log files, or unencrypted storage is a serious policy violation.
- Implementing custom cryptographic algorithms is grounds for immediate code revert and disciplinary action.
- The Security Lead monitors key rotation compliance and reports overdue rotations to the CTO.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-004 | Incident Response Plan |
| POL-007 | Disaster Recovery Plan |
| POL-008 | Data Classification Policy |
| POL-009 | Vendor Management Policy |
| POL-012 | Password Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
