# Acceptable Use Policy

**Document ID**: POL-010
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Acceptable Use Policy defines the permitted and prohibited uses of Questo Ltd information systems, communication platforms, development tools, and the corteX AI Agent SDK platform by all personnel. It establishes behavioral expectations that protect Questo's information assets, maintain system security, and ensure compliance with legal and regulatory obligations.

This policy addresses the unique context of working with AI agent systems, including responsibilities when interacting with LLM providers, handling customer data processed by AI agents, and developing an SDK that will be deployed in enterprise environments.

**SOC 2 Mapping**: CC1.4 (Ethical Values), CC6.1-CC6.2 (Logical Access)

## 2. Scope

This policy applies to:

- **Personnel**: All employees, contractors, consultants, and temporary staff of Questo Ltd.
- **Systems**: All Questo-owned or Questo-managed information systems, including development environments, cloud infrastructure, communication platforms, email, source code repositories, and the corteX SDK development and testing environments.
- **Devices**: All devices used to access Questo systems, whether company-issued or personal (BYOD).
- **Accounts**: All accounts provisioned for Questo business, including GitHub, cloud providers, LLM provider accounts, email, and messaging platforms.
- **Data**: All data accessed, created, processed, or transmitted using Questo systems.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Acceptable Use** | Use of Questo systems that is consistent with business purposes and this policy |
| **Prohibited Use** | Use of Questo systems that violates this policy, regardless of intent |
| **Questo Systems** | All hardware, software, networks, cloud services, and accounts owned, leased, or managed by Questo Ltd |
| **Personal Use** | Use of Questo systems for non-business purposes |
| **LLM Interaction** | Any communication with a large language model through an API or interface |
| **Prompt** | Input text or instructions sent to an LLM provider |
| **Shadow IT** | Use of unauthorized applications, services, or systems for Questo business |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and ensures its communication to all personnel
- Approves the list of authorized tools and services
- Reviews policy violations and determines appropriate response
- Updates this policy to reflect changes in technology and business practices

### 4.2 Security Lead

- Monitors for policy violations through system logs and periodic audits
- Investigates reported policy violations
- Maintains the list of approved tools, platforms, and services
- Conducts acceptable use training as part of security awareness

### 4.3 Team Leads / Managers

- Ensure their team members acknowledge and understand this policy
- Report suspected violations to the Security Lead
- Model acceptable use behavior

### 4.4 All Personnel

- Read, understand, and comply with this policy
- Report suspected violations by others
- Ask the Security Lead for guidance when uncertain about acceptable use

## 5. Policy Statements

### 5.1 General Acceptable Use

#### 5.1.1 Permitted Uses

- Using Questo systems for authorized business purposes
- Reasonable incidental personal use that does not interfere with job duties, consume significant resources, or violate any other provision of this policy
- Using approved tools and platforms for development, testing, communication, and collaboration
- Accessing LLM provider APIs for authorized development and testing of the corteX SDK

#### 5.1.2 Prohibited Uses

The following uses of Questo systems are prohibited:

1. **Security Violations**:
   - Attempting to access systems, accounts, or data without authorization
   - Sharing credentials, API keys, or access tokens with unauthorized persons
   - Disabling or circumventing security controls (MFA, antivirus, firewalls, encryption)
   - Installing unauthorized software that may compromise system security
   - Connecting to Questo systems from networks known to be compromised

2. **Data Handling Violations**:
   - Transmitting CONFIDENTIAL or RESTRICTED data through unapproved channels
   - Storing customer API keys, credentials, or RESTRICTED data on personal devices
   - Sending production customer data to external LLM providers for personal experimentation
   - Copying source code or proprietary data to personal accounts or storage
   - Exfiltrating any Questo data for non-business purposes

3. **LLM-Specific Prohibitions**:
   - Sending RESTRICTED data (API keys, credentials, signing keys) to any LLM provider
   - Sending CONFIDENTIAL customer data to LLM providers without authorized business purpose and appropriate data handling agreement in place
   - Using LLM providers to generate code that circumvents corteX security controls
   - Using personal LLM accounts to process Questo business data
   - Storing prompts containing sensitive data in LLM conversation histories

4. **Legal and Ethical Violations**:
   - Using Questo systems for any illegal activity
   - Accessing, distributing, or storing inappropriate, offensive, or discriminatory content
   - Using Questo systems to harass, threaten, or discriminate against any person
   - Violating intellectual property rights, including using pirated software
   - Using Questo systems for personal financial gain beyond authorized compensation

5. **Operational Violations**:
   - Using unauthorized cloud services or SaaS applications for Questo business (shadow IT)
   - Making changes to production systems without following change management (POL-003)
   - Exhausting LLM provider API quotas through personal experimentation
   - Intentionally causing system outages or performance degradation

### 5.2 LLM Provider Usage Guidelines

Given that the corteX SDK integrates with multiple LLM providers, the following guidelines apply to all interactions with LLM APIs:

1. **Authorized Accounts Only**: Use only Questo-provisioned API keys and accounts for business-related LLM interactions. Never use personal API keys for Questo work, and never use Questo API keys for personal projects.
2. **Data Classification Awareness**: Before sending any data to an LLM provider, consider its classification level (POL-008). RESTRICTED data must never be included in prompts. CONFIDENTIAL data requires an authorized business purpose and an appropriate data handling agreement with the provider.
3. **Prompt Hygiene**: Do not include real customer API keys, passwords, or personal data in prompts. Use synthetic or redacted data for development and testing.
4. **Output Review**: Review LLM outputs for accuracy before incorporating into the corteX codebase. Do not blindly trust generated code, especially for security-critical modules.
5. **Rate Limit Awareness**: Be aware of API rate limits (currently Tier-1 on Gemini: 25 RPM, 250 RPD for Gemini 3 models). Coordinate with the team to avoid quota exhaustion.
6. **No Sensitive Conversations**: Do not discuss CONFIDENTIAL business strategy, unreleased product plans, or vulnerability details in LLM conversations that may be retained by the provider.

### 5.3 Source Code and Development

1. **Repository Rules**: All code development occurs in authorized GitHub repositories. Pushing code to personal repositories requires CTO approval.
2. **Branch Protection**: Respect branch protection rules. Do not attempt to bypass PR requirements, CI/CD checks, or review requirements.
3. **Secret Handling**: Never hardcode API keys, passwords, credentials, or encryption keys in source code. Use environment variables or the `KeyVault` module.
4. **Open-Source Contributions**: Contributions to external open-source projects using knowledge gained at Questo require CTO notification. Contributions that include Questo proprietary information are prohibited.
5. **Development Environments**: Keep development environments patched and secure. Enable full disk encryption on all devices used for development (POL-020).

### 5.4 Email and Communication Platforms

#### 5.4.1 Business Communication

1. Use Questo-approved communication platforms for all business discussions.
2. Do not use personal messaging applications (WhatsApp, Telegram, personal Signal) for business conversations containing INTERNAL or higher data.
3. Communications with customers, vendors, or the public about Questo or corteX must be accurate and authorized. Security-related communications require CTO approval.
4. Do not auto-forward Questo email to personal email accounts. Email forwarding rules that send CONFIDENTIAL or RESTRICTED data outside Questo-controlled systems are prohibited.
5. Exercise caution with email attachments and links. Report suspected phishing emails to the Security Lead immediately per POL-004 and do not click links or open attachments from unverified senders.

#### 5.4.2 Email Security

1. CONFIDENTIAL data may be sent via email only with encryption (TLS in transit is mandatory; attachment-level encryption is required for highly sensitive content).
2. RESTRICTED data must never be transmitted via email under any circumstances (POL-008).
3. Verify recipient email addresses before sending CONFIDENTIAL content. Use address auto-complete carefully to avoid misdirected emails.
4. Do not include API keys, passwords, cryptographic keys, or other secrets in email, even to internal recipients. Use the approved secrets management system or password manager's secure sharing feature.

### 5.5 Social Media

1. **Official Accounts**: Only personnel explicitly authorized by the CEO may post to official Questo social media accounts (LinkedIn, Twitter/X, blog).
2. **Personal Accounts**: When discussing Questo or corteX on personal social media, personnel must:
   - Clearly state that views are their own and not official Questo positions.
   - Not disclose INTERNAL, CONFIDENTIAL, or RESTRICTED information.
   - Not share screenshots of internal systems, code, conversations, or dashboards.
   - Not disclose unreleased features, roadmap items, or financial information.
   - Not discuss security vulnerabilities, active incidents, or internal investigation details.
3. **Competitive Commentary**: Do not make disparaging comments about competitors, their products, or their employees on social media.
4. **Customer References**: Do not disclose customer names, use cases, or data on social media without explicit written customer consent and CEO approval.
5. **Pre-Approval**: Blog posts, articles, or conference presentations about corteX, AI agent technology, or Questo business require CTO review before publication or delivery.

### 5.6 Software Installation and Shadow IT

1. **Authorized Software Only**: Only software approved by the CTO or Security Lead may be installed on Questo-managed systems or used for Questo business purposes. The Security Lead maintains the authorized software list.
2. **Development Tools**: Development tools and libraries included in the corteX project's `pyproject.toml`, `requirements.txt`, or documented setup procedures are pre-approved for development use.
3. **New Tool Requests**: Requests for new software or SaaS services are submitted to the Security Lead for evaluation. Tools that process CONFIDENTIAL or higher data require CTO approval and vendor assessment per POL-009.
4. **Browser Extensions**: Browser extensions must be approved by the Security Lead before installation. Extensions with broad permissions (access to all sites, read all page content) are prohibited unless specifically approved for a business purpose.
5. **Shadow IT Prohibition**: Using unapproved cloud services, SaaS applications, AI tools, or development platforms for Questo business is prohibited. Examples of shadow IT include using personal Dropbox for work files, unapproved project management tools, or unapproved AI assistants for processing Questo data.
6. **Open-Source Software**: Use of open-source software must comply with license requirements. Software with copyleft licenses (GPL, AGPL) requires CTO review before inclusion in the corteX SDK or use in Questo infrastructure.
7. **Mobile Applications**: Business-related mobile applications must be downloaded from official app stores only. Side-loading applications on devices used for Questo business is prohibited.

### 5.7 Monitoring and Privacy

1. **Monitoring Disclosure**: Questo Ltd monitors activity on its information systems to protect its assets, ensure policy compliance, detect security incidents, and maintain system performance. By using Questo systems, all personnel consent to this monitoring.
2. **Scope of Monitoring**: Monitoring may include, but is not limited to: email content and metadata, messaging platform content, internet browsing activity, file access and modification patterns, login and authentication events, application usage, network traffic, and source code repository activity.
3. **No Expectation of Privacy**: Personnel should have no expectation of privacy when using Questo information systems, including for permitted personal use (Section 5.1.2).
4. **Legal Compliance**: Monitoring is conducted in compliance with applicable privacy laws and employment regulations. Monitoring data is classified as CONFIDENTIAL and handled accordingly per POL-008.
5. **Audit Trails**: System logs and audit trails are maintained per POL-017 (Logging and Monitoring Policy) and retained for a minimum of 1 year.
6. **Access to Monitoring Data**: Access to monitoring data and investigation results is restricted to the Security Lead, CTO, and personnel authorized for specific investigations per POL-002.
7. **Consequences**: Evidence of policy violations discovered through monitoring is handled per Section 8 (Enforcement) and POL-001 Section 8.

### 5.8 Personal Devices (BYOD)

Personnel using personal devices to access Questo systems must:

1. Enable full disk encryption
2. Enable device screen lock with a maximum 5-minute timeout
3. Keep the operating system and software updated with security patches
4. Enable MFA on all Questo-related accounts accessed from the device
5. Not store RESTRICTED data on personal devices
6. Submit the device for remote wipe capability if accessing CONFIDENTIAL or higher data
7. Report lost or stolen devices immediately to the Security Lead

### 5.9 Internet and Network Use

1. Internet access is provided primarily for business purposes. Limited personal browsing is permitted per Section 5.1.2.
2. Downloading software from untrusted sources is prohibited. Software downloads are restricted to the authorized software list (Section 5.6).
3. Visiting websites known to host malicious content is prohibited.
4. Using anonymization tools (Tor, anonymous VPNs) to bypass Questo security controls or monitoring is prohibited.
5. VPN is required when accessing Questo systems from public or untrusted networks (coffee shops, airports, co-working spaces).
6. Connecting unauthorized network devices (personal routers, wireless access points) to Questo networks is prohibited.
7. Network modifications, including DNS settings changes, require CTO approval.

### 5.10 Remote Work

1. All personnel working remotely must ensure a secure physical environment that prevents unauthorized viewing or access to Questo data.
2. Remote connections to Questo systems must use VPN or equivalent encrypted connections with MFA authentication.
3. Company and personal devices used for Questo work must be physically secured when unattended (locked room, locked vehicle trunk). Devices must never be left unattended in public spaces.
4. Printed documents containing CONFIDENTIAL or RESTRICTED data must be secured and properly disposed of (cross-cut shredding) when no longer needed.
5. Take precautions to prevent shoulder surfing when working with classified data in semi-public environments.

## 6. Procedures

### 6.1 Policy Acknowledgement Procedure

1. All personnel receive a copy of this policy upon hire or engagement.
2. Personnel sign an acknowledgement form within 14 days of receipt.
3. Acknowledgement records are maintained by the Security Lead for the duration of employment plus 3 years.
4. The policy is redistributed and re-acknowledged annually as part of security awareness training.

### 6.2 Authorized Tools and Services

1. The Security Lead maintains a list of approved tools, platforms, and services for Questo business.
2. Requests for new tools or services are submitted to the Security Lead for evaluation.
3. The CTO approves new tools that process CONFIDENTIAL or higher data.
4. The approved tools list is reviewed annually and updated as needed.

### 6.3 Reporting Violations

1. Personnel who observe or suspect a policy violation report it to the Security Lead or their manager.
2. Reports may be made anonymously through the designated reporting channel.
3. The Security Lead investigates reported violations and escalates per POL-001 enforcement provisions.
4. Retaliation against good-faith reporters is prohibited and is itself a serious policy violation.

## 7. Exceptions

1. Exceptions to this policy require written approval from the CTO.
2. Exceptions to LLM data handling restrictions (Section 5.2) require both CTO and Security Lead approval.
3. Exceptions are documented with: the specific provision, justification, compensating controls, and maximum duration (90 days, renewable).
4. No exceptions are permitted for: sharing credentials, disabling MFA, or knowingly processing RESTRICTED data in violation of its handling requirements.
5. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Violations of this policy may result in disciplinary action, up to and including termination of employment or contract.
- First-time minor violations (e.g., incidental personal use exceeding reasonable limits) result in a documented warning.
- Serious violations (e.g., data exfiltration, credential sharing, sending RESTRICTED data to LLM providers) result in immediate access suspension pending investigation and may result in termination.
- Violations involving illegal activity are referred to law enforcement.
- The Security Lead tracks violations and reports patterns to the CTO during quarterly security reviews.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-004 | Incident Response Plan |
| POL-008 | Data Classification Policy |
| POL-009 | Vendor Management Policy |
| POL-011 | Encryption Policy |
| POL-012 | Password Policy |
| POL-013 | Code of Conduct |
| POL-017 | Logging and Monitoring Policy |
| POL-020 | Workstation Security Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
