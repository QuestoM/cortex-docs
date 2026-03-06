# Workstation Security Policy

**Document ID**: POL-020
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Workstation Security Policy defines the security requirements for all computing devices (laptops, desktops, phones, tablets) used to access Questo Ltd systems or data. As a remote-first organization where all corteX AI Agent SDK development occurs on distributed endpoints, workstation security is a critical layer in the defense-in-depth strategy.

This policy ensures that every device used for Questo Ltd work is configured with appropriate security controls, including disk encryption, screen lock, software updates, and endpoint protection, to protect the corteX source code, customer data, and business information stored or accessible from these devices.

**SOC 2 Mapping**: CC6.1 (Logical Access), CC6.4 (Physical Access to Assets), CC6.8 (Malware Prevention)

## 2. Scope

This policy applies to:

- **Devices**: All laptops, desktops, phones, and tablets that access Questo Ltd systems or store Questo Ltd data, whether company-issued or personal (BYOD).
- **Personnel**: All employees, contractors, consultants, and third parties who use computing devices for Questo Ltd work.
- **Operating Systems**: All operating systems used on in-scope devices, including Windows, macOS, Linux, iOS, and Android.
- **Data**: All Questo Ltd data stored on or accessible from in-scope devices, including source code, business documents, credentials, and communications.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **FDE** | Full Disk Encryption; encryption of the entire disk volume so data is unreadable without the decryption key |
| **EDR** | Endpoint Detection and Response; advanced endpoint security software that detects, investigates, and responds to threats |
| **BYOD** | Bring-Your-Own-Device; personal devices used for company work |
| **MDM** | Mobile Device Management; centralized management of mobile device security policies |
| **Patch** | A software update that fixes a bug or vulnerability |
| **Screen Lock** | An automatic lock that requires authentication to regain access after a period of inactivity |
| **Remote Wipe** | The ability to erase all data on a device remotely, typically used when a device is lost or stolen |
| **Endpoint Protection** | Software that protects a device from malware, including antivirus, anti-malware, and EDR capabilities |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and approves the approved hardware and software configurations
- Approves BYOD requests and associated security requirements
- Reviews workstation security compliance quarterly

### 4.2 Security Lead

- Defines and maintains the approved workstation security baseline configuration
- Verifies endpoint compliance during onboarding and quarterly reviews
- Manages remote wipe capabilities and executes wipe commands for lost or stolen devices
- Coordinates endpoint security incident response
- Evaluates and recommends endpoint protection solutions

### 4.3 All Personnel

- Configure their devices in compliance with this policy before accessing Questo Ltd systems
- Install required security software and keep it active and updated
- Apply operating system and application updates promptly per Section 5.3
- Report lost or stolen devices immediately per POL-004 and POL-015
- Do not disable, circumvent, or uninstall required security controls

## 5. Policy Statements

### 5.1 Full Disk Encryption

Full disk encryption is **mandatory** on all devices that access Questo Ltd systems or store Questo Ltd data. There are no exceptions.

| Operating System | Required FDE | Minimum Standard |
|-----------------|-------------|-----------------|
| **Windows** | BitLocker | AES-256, TPM-backed |
| **macOS** | FileVault 2 | AES-256-XTS |
| **Linux** | LUKS | AES-256-XTS |
| **iOS** | Native encryption (enabled by default with passcode) | AES-256 hardware encryption |
| **Android** | Native encryption (enabled by default on modern devices) | AES-256 file-based encryption |

- FDE must be enabled before any Questo Ltd data is stored on the device or before the device is used to access Questo Ltd systems.
- Recovery keys for FDE must be stored securely. For company-issued devices, recovery keys are escrowed with the Security Lead. For BYOD, the user is responsible for recovery key backup.
- FDE status is verified during onboarding and during quarterly compliance checks.

### 5.2 Screen Lock and Authentication

1. **Automatic Screen Lock**: Devices must automatically lock after a maximum of 5 minutes of inactivity.
2. **Lock Authentication**: Unlocking the device must require strong authentication: a password meeting POL-012 requirements, biometric authentication (fingerprint, face recognition), or a minimum 6-digit PIN.
3. **Failed Attempts**: After 10 consecutive failed unlock attempts, the device must either lock for a minimum of 30 minutes or trigger a remote wipe (configurable based on data sensitivity).
4. **Manual Lock**: Personnel must manually lock their device (Windows: Win+L, macOS: Ctrl+Cmd+Q, Linux: Super+L) whenever stepping away, regardless of the automatic lock timer.

### 5.3 Software Updates and Patching

1. **Operating System Updates**: Critical and security updates must be applied within 7 days of release. All other updates must be applied within 30 days.
2. **Application Updates**: Applications used for Questo Ltd work (browsers, development tools, communication platforms) must be kept on supported versions. Security updates must be applied within 14 days.
3. **Automatic Updates**: Automatic updates should be enabled where possible. If automatic updates are disabled for development stability reasons, the user is responsible for manual updates within the required timeframes.
4. **End-of-Life Software**: Operating systems and applications that have reached end-of-life (no longer receiving security updates) must not be used for Questo Ltd work. Devices running EOL software must be upgraded before continued use.
5. **Development Tools**: Python interpreters, pip, Git, and IDE software must be maintained on currently supported versions.

### 5.4 Endpoint Protection

1. **Required Protection**: All Windows and macOS devices must run active endpoint protection software. Linux devices must run endpoint protection if available for the distribution, or implement compensating controls (see Section 5.4.2).
2. **Real-Time Scanning**: Endpoint protection must include real-time file and process scanning.
3. **Definition Updates**: Malware definitions must be updated automatically, at minimum daily.
4. **No Disabling**: Users must not disable, pause, or uninstall endpoint protection software. If a temporary exception is needed (e.g., for a legitimate development tool that triggers a false positive), the user must obtain Security Lead approval and re-enable protection immediately after.

#### 5.4.1 Recommended Endpoint Protection Software

The Security Lead maintains a list of approved endpoint protection solutions. At minimum, the following built-in protections must be active:

| Operating System | Minimum Protection |
|-----------------|-------------------|
| **Windows** | Microsoft Defender (real-time protection, cloud-delivered protection, tamper protection enabled) |
| **macOS** | XProtect + Gatekeeper (enabled) + recommended third-party EDR |
| **Linux** | ClamAV or equivalent + rootkit detection; recommended third-party EDR |
| **iOS** | Managed via MDM with jailbreak detection |
| **Android** | Google Play Protect + device encryption verified |

#### 5.4.2 Linux Compensating Controls

If a third-party EDR agent is not available for a Linux distribution, the following compensating controls must be implemented:

- ClamAV with daily definition updates and weekly full system scans
- Rootkit detection tool (rkhunter or chkrootkit) with monthly scans
- Host-based firewall (iptables/nftables) configured to deny all inbound by default
- Automatic security updates enabled via the distribution's package manager

### 5.5 Firewall and Network Security

1. **Host Firewall**: The operating system firewall must be enabled on all devices.
2. **Inbound Connections**: All unsolicited inbound connections must be blocked by default. Exceptions for development servers (e.g., local test servers) are permitted on loopback (localhost) only.
3. **Bluetooth**: Bluetooth should be disabled when not in active use to reduce the attack surface.
4. **AirDrop/Nearby Share**: File sharing services must be set to "Contacts Only" or disabled.

### 5.6 Data Storage and Handling on Workstations

1. **Minimize Local Storage**: Confidential and Restricted data should be stored in backed-up cloud systems, not exclusively on local devices. Local copies should be minimized.
2. **No Unencrypted Removable Media**: Confidential or Restricted data must not be stored on unencrypted USB drives, external hard drives, or SD cards.
3. **Credential Storage**: Credentials, API keys, and tokens must be stored in the operating system's credential manager or an approved password manager per POL-012. Plaintext credential files are prohibited.
4. **Source Code**: The corteX source code repository is cloned locally for development. FDE protects the local clone. Source code must not be copied to personal cloud storage, personal email, or unauthorized devices.
5. **Temporary Files**: Development temporary files, build artifacts, and cached data should be cleaned periodically. Sensitive data must not remain in temporary directories indefinitely.

### 5.7 BYOD Requirements

Personnel using personal devices for Questo Ltd work must meet all requirements in this policy. Additionally:

1. **Approval**: BYOD usage must be approved by the CTO.
2. **Compliance Verification**: The Security Lead verifies BYOD compliance during onboarding and quarterly reviews.
3. **Remote Wipe Consent**: BYOD users must consent to remote wipe of the device in case of loss, theft, or termination. Where containerization is available, only the work container is wiped.
4. **Separation**: Where technically feasible, work data should be separated from personal data using work profiles, containers, or separate partitions.
5. **Departure**: Upon termination, all Questo Ltd data must be removed from BYOD devices. The employee confirms deletion in writing.

### 5.8 Device Disposal

1. **Data Destruction**: Before disposal, donation, or return (for leased devices), all data must be securely erased using a full cryptographic wipe (erase the FDE keys, rendering all data unrecoverable).
2. **Verification**: Data destruction is verified by attempting to boot the device and confirming no data is accessible.
3. **Documentation**: Disposal is documented with: device identifier (serial number), disposal date, disposal method, and executor name.
4. **Records**: Disposal records are retained for 3 years.

## 6. Procedures

### 6.1 New Device Setup Procedure

1. The Security Lead provides the Workstation Security Baseline Checklist to the new user.
2. The user configures the device per the checklist:
   - Enable Full Disk Encryption (Section 5.1)
   - Set screen lock timeout to 5 minutes maximum (Section 5.2)
   - Install and activate endpoint protection (Section 5.4)
   - Enable the host firewall (Section 5.5)
   - Enable automatic OS and application updates (Section 5.3)
   - Configure credential storage in an approved password manager (Section 5.6)
   - Enroll in VPN per POL-016
3. The user confirms completion of the checklist to the Security Lead.
4. The Security Lead verifies key settings (FDE enabled, endpoint protection active, firewall on) remotely or via screenshot evidence.
5. The setup is documented in the device inventory.

### 6.2 Quarterly Workstation Compliance Review

1. The Security Lead requests a compliance self-assessment from all personnel each quarter.
2. The self-assessment covers: FDE status, screen lock timeout, endpoint protection status, OS version and patch status, firewall status, and any security software exceptions.
3. The Security Lead reviews self-assessments and follows up on any non-compliant items.
4. Non-compliant devices must be remediated within 5 business days. Access may be restricted for devices that remain non-compliant.
5. Results are documented and included in the quarterly security review report to the CTO.

### 6.3 Lost or Stolen Device Procedure

1. The user reports the loss or theft immediately to the Security Lead via the fastest available channel.
2. The Security Lead initiates remote wipe within 1 hour of receiving the report.
3. The Security Lead revokes the device's VPN certificate, access tokens, and any cached credentials.
4. A security incident is opened per POL-004 to assess data exposure risk.
5. If the device contained RESTRICTED data, the incident is escalated per the RESTRICTED data breach procedures in POL-004.
6. A replacement device is provisioned per the New Device Setup Procedure (Section 6.1).

## 7. Exceptions

1. Exceptions to this policy require written approval from the CTO.
2. Exception requests must include: the specific policy provision, device details, business justification, compensating controls, and requested duration (maximum 90 days).
3. No exceptions are permitted for: Full Disk Encryption (Section 5.1), the screen lock requirement (Section 5.2), or the requirement to report lost or stolen devices.
4. Temporary endpoint protection exceptions (e.g., false positive blocking legitimate development tools) may be approved by the Security Lead for a maximum of 24 hours.
5. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Accessing Questo Ltd systems from a device that does not meet the requirements of this policy is a policy violation.
- Disabling Full Disk Encryption or endpoint protection is a serious violation resulting in immediate access suspension pending remediation.
- Failure to report a lost or stolen device promptly (within 4 hours) is a policy violation subject to disciplinary action.
- Repeated non-compliance identified during quarterly reviews will result in mandatory remedial training and may result in access restrictions.
- The Security Lead monitors compliance through quarterly reviews and device inventory audits.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-004 | Incident Response Plan |
| POL-011 | Encryption Policy |
| POL-012 | Password Policy |
| POL-015 | Physical Security Policy |
| POL-016 | Remote Access Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
