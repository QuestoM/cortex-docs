# Physical Security Policy

**Document ID**: POL-015
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Physical Security Policy defines the physical and environmental security controls that protect Questo Ltd information assets, personnel, and equipment. As a remote-first startup with no company-owned data centers, this policy addresses the unique physical security profile of a distributed engineering organization that delivers the corteX AI Agent SDK as a customer-deployed, on-premises product.

Physical security controls focus on: securing personnel workstations in remote environments, inheriting physical security from cloud service providers, and defining requirements for the rare occasions when personnel work from shared or co-working spaces.

**SOC 2 Mapping**: CC6.4 (Physical Access), CC6.5 (Physical Access Restrictions)

## 2. Scope

This policy applies to:

- **Personnel**: All employees, contractors, and consultants working on behalf of Questo Ltd, regardless of location.
- **Locations**: All locations from which Questo Ltd business is conducted, including home offices, co-working spaces, travel locations, and any temporary office facilities.
- **Equipment**: All company-issued and personal devices used for Questo Ltd business, including laptops, phones, tablets, external storage, and peripherals.
- **Cloud Infrastructure**: Physical security of cloud-hosted infrastructure is inherited from cloud service providers and validated through their SOC 2 reports.
- **Customer On-Premises Deployments**: Physical security of customer-deployed corteX instances is the responsibility of the customer. Questo Ltd provides security guidance in the SDK deployment documentation.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Remote-First** | An organizational model where remote work is the default; there is no required physical office |
| **Workspace** | Any physical location where an employee or contractor performs Questo Ltd work |
| **Company-Issued Device** | A laptop, phone, or other device owned and provided by Questo Ltd |
| **BYOD** | Bring-Your-Own-Device; use of personal devices for company work, subject to security requirements |
| **Co-Working Space** | A shared commercial workspace used on an ad-hoc or subscription basis |
| **Clean Desk** | The practice of ensuring no confidential information is visible or accessible on a desk when unattended |
| **CSP** | Cloud Service Provider (e.g., AWS, Azure, GCP) providing infrastructure hosting |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and ensures its implementation
- Approves the selection of cloud service providers based on physical security posture
- Reviews CSP SOC 2 reports annually to validate inherited physical controls
- Approves exceptions for non-standard work locations

### 4.2 Security Lead

- Verifies that CSP physical security controls meet Questo Ltd requirements
- Maintains records of CSP SOC 2 reports and physical security certifications
- Conducts annual review of remote workspace security practices
- Investigates physical security incidents (device theft, unauthorized access)

### 4.3 All Personnel

- Secure their workspace to prevent unauthorized viewing of or access to confidential information
- Lock workstations when stepping away (automatic lock within 5 minutes per POL-002)
- Store company devices securely when not in use
- Report lost or stolen devices immediately per POL-004
- Follow clean desk practices when working with confidential information

## 5. Policy Statements

### 5.1 Remote-First Physical Security Model

Questo Ltd operates as a remote-first organization. The following principles govern the physical security approach:

1. **No Company Data Center**: Questo Ltd does not own or operate physical data centers. All infrastructure is cloud-hosted or customer-deployed on-premises.
2. **Inherited Controls**: Physical security of cloud infrastructure is inherited from CSPs. Questo validates these controls annually through CSP SOC 2 Type II reports.
3. **Endpoint-Centric Security**: Physical security focuses on protecting endpoints (laptops, phones) since these are the primary physical attack surface. Detailed endpoint requirements are defined in POL-020 (Workstation Security Policy).
4. **Personnel Responsibility**: Each team member is responsible for the physical security of their workspace and devices.

### 5.2 Cloud Service Provider Physical Security

Questo Ltd relies on CSPs for the physical security of hosted infrastructure. The following controls apply:

1. **CSP Selection Criteria**: CSPs must hold a current SOC 2 Type II report with no material exceptions related to physical security controls. Additionally, they must demonstrate: physical access controls (biometric, badge, mantrap), environmental controls (fire suppression, climate control, flood protection), redundant power and cooling, 24/7 physical monitoring (CCTV, security personnel), and visitor management procedures.
2. **Annual Validation**: The Security Lead reviews CSP SOC 2 reports annually. Any material exceptions or complementary user entity controls (CUECs) are documented and addressed.
3. **Subprocessor Review**: If the CSP subcontracts physical hosting, the subprocessor's physical security posture is included in the review per POL-009 (Vendor Management Policy).

### 5.3 Remote Workspace Requirements

All personnel working remotely must maintain a workspace that meets the following minimum requirements:

1. **Privacy**: The workspace should be in a private area where screens are not visible to unauthorized persons (family members, visitors, passersby). If a fully private space is not available, a privacy screen on the laptop is required when working with CONFIDENTIAL or RESTRICTED information.
2. **Network Security**: Home networks must use WPA3 or WPA2 encryption with a strong password. Public Wi-Fi must not be used for accessing Questo systems without an active VPN connection per POL-016 (Remote Access Policy).
3. **Device Storage**: When not in use, company devices must be stored in a secure location (locked drawer, closet, or room). Devices must not be left in vehicles, common areas, or other locations where theft risk is elevated.
4. **Clean Desk**: Printed confidential materials (if any) must be secured when not in active use and shredded when no longer needed. Whiteboards or notes containing confidential information must be erased after use.
5. **Visitor Awareness**: Personnel must ensure that visitors to their home or workspace cannot view screens displaying confidential information or overhear confidential conversations.

### 5.4 Co-Working Space Requirements

When personnel work from co-working spaces:

1. **Privacy Screens**: Privacy screen filters are mandatory on all devices when working in shared spaces.
2. **Device Supervision**: Devices must never be left unattended and unlocked in shared spaces. If stepping away, the device must be locked and either taken or secured.
3. **Confidential Conversations**: Phone calls or meetings involving confidential information must be conducted in private rooms, not open areas.
4. **Network**: Co-working space networks are treated as untrusted. VPN connection is mandatory per POL-016.
5. **Printing**: Confidential documents must not be printed on shared printers in co-working spaces.

### 5.5 Travel Security

When traveling with company devices:

1. **Carry-On Only**: Devices must be carried in hand luggage, never checked baggage.
2. **Screen Lock**: Devices must be shut down or locked when passing through security checkpoints.
3. **Public Spaces**: Work on confidential information in public spaces (airports, cafes, trains) requires a privacy screen and heightened awareness of surroundings.
4. **Hotel Security**: Devices must be stored in the hotel room safe when not in use. If no safe is available, devices must not be left unattended in the room.
5. **Border Crossings**: Personnel must be aware that devices may be subject to inspection at international borders. If traveling to high-risk regions, consult the CTO before traveling with devices containing source code or RESTRICTED data.

### 5.6 Device Lifecycle Physical Security

1. **Provisioning**: New devices are configured with full disk encryption before any confidential data is stored (POL-020).
2. **Maintenance**: Device repairs must be performed by authorized service providers. Full disk encryption ensures data is protected during service.
3. **Disposal**: Devices reaching end-of-life undergo certified data destruction (cryptographic wipe or physical destruction). The Security Lead maintains disposal records with device serial numbers and destruction dates.
4. **Lost or Stolen Devices**: Lost or stolen devices are reported immediately to the Security Lead per POL-004. Remote wipe is initiated within 1 hour of report. The incident is investigated and documented.

### 5.7 Customer On-Premises Deployments

- The corteX SDK is designed for customer-deployed, on-premises operation. Physical security of the customer's deployment environment is the customer's responsibility.
- Questo Ltd provides deployment security guidance in the SDK documentation, including recommendations for: server room access controls, network segmentation, and hardware security module (HSM) integration for key management.
- Questo Ltd does not have physical or logical access to customer on-premises environments.

## 6. Procedures

### 6.1 New Employee Physical Security Setup

1. Upon hire, the Security Lead provides the employee with the Remote Workspace Security Checklist covering: network configuration, device storage, workspace privacy, and clean desk requirements.
2. The employee confirms compliance with the checklist within 14 days of start date.
3. If a privacy screen is needed, it is provided by Questo Ltd or reimbursed.

### 6.2 Annual CSP Physical Security Review

1. The Security Lead obtains current SOC 2 Type II reports from all CSPs by March 31 each year.
2. The Security Lead reviews the reports for physical security sections, noting any material exceptions or CUECs.
3. Findings are documented and presented to the CTO. Material exceptions trigger a risk assessment per POL-005.
4. Review records are retained for 3 years.

### 6.3 Lost or Stolen Device Procedure

1. The employee reports the loss or theft immediately to the Security Lead via the fastest available channel.
2. The Security Lead initiates remote wipe within 1 hour.
3. The Security Lead revokes the device's VPN certificate and any stored access tokens.
4. A security incident is opened per POL-004 to assess data exposure risk.
5. A replacement device is provisioned per the standard provisioning process.

## 7. Exceptions

1. Exceptions to this policy require written approval from the CTO.
2. Exception requests must include: the specific policy provision, business justification, compensating controls, and requested duration (maximum 6 months).
3. No exceptions are permitted for: full disk encryption on devices, the requirement to report lost or stolen devices, or the prohibition on leaving devices unattended and unlocked in public spaces.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Violations of this policy, including failure to report lost or stolen devices, may result in disciplinary action up to and including termination.
- Repeated failure to comply with workspace security requirements will result in mandatory remedial training and may result in temporary suspension of remote work privileges.
- The Security Lead monitors compliance through annual workspace security self-assessments.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-004 | Incident Response Plan |
| POL-009 | Vendor Management Policy |
| POL-011 | Encryption Policy |
| POL-016 | Remote Access Policy |
| POL-020 | Workstation Security Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
