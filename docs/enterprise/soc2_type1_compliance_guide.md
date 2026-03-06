# SOC 2 Type 1 Compliance Guide for corteX (AI Agent SDK)

> **Document Type**: Permanent Compliance Reference
> **Applicable To**: Questo / corteX - Enterprise AI Agent SDK
> **Last Updated**: 2026-02-16
> **Status**: Research Complete - Pre-Implementation

---

## Table of Contents

1. [SOC 2 Fundamentals](#1-soc-2-fundamentals)
2. [Trust Service Criteria (TSC) Deep Dive](#2-trust-service-criteria-tsc-deep-dive)
3. [Common Criteria (CC1-CC9) Detailed Controls](#3-common-criteria-cc1-cc9-detailed-controls)
4. [AI Agent SDK Specific Controls](#4-ai-agent-sdk-specific-controls)
5. [Required Policies and Documents](#5-required-policies-and-documents)
6. [Technical Controls Checklist](#6-technical-controls-checklist)
7. [The Audit Process Step-by-Step](#7-the-audit-process-step-by-step)
8. [Compliance Automation Platforms](#8-compliance-automation-platforms)
9. [2025-2026 Updates and AI Considerations](#9-2025-2026-updates-and-ai-considerations)
10. [Cost and Timeline Estimates](#10-cost-and-timeline-estimates)
11. [corteX-Specific Implementation Plan](#11-cortex-specific-implementation-plan)

---

## 1. SOC 2 Fundamentals

### 1.1 What is SOC 2?

SOC 2 (System and Organization Controls 2) is an auditing framework developed by the
**American Institute of Certified Public Accountants (AICPA)** that evaluates how service
organizations manage customer data. It is the de facto security standard for SaaS companies
selling to enterprise customers.

**Key facts:**
- SOC 2 is NOT a certification -- it is an **attestation report** issued by a CPA firm
- Only **licensed CPA firms** accredited by the AICPA can perform SOC 2 audits
- The auditor must be **completely independent** (no prior relationship with the organization)
- SOC 2 reports are **restricted-use documents** (shared only with customers under NDA)
- 66% of B2B customers now check for SOC 2 before signing contracts (2025 data)

### 1.2 Type 1 vs Type 2

| Aspect | Type 1 | Type 2 |
|--------|--------|--------|
| **What it evaluates** | Design of controls at a **point in time** | Operating effectiveness over a **period of time** |
| **Duration assessed** | Single date (snapshot) | 6-12 months of operation |
| **Timeline to complete** | 3-6 months preparation + 1-2 month audit | 6-12 months observation + 1-3 month audit |
| **Cost** | $20,000 - $60,000 total | $50,000 - $150,000+ total |
| **What auditors test** | Are controls properly designed? | Are controls operating effectively over time? |
| **Report output** | Controls listed, no test results | Controls listed WITH test results and exceptions |
| **Best for** | First-time compliance, unblocking deals | Ongoing trust, mature organizations |

**Recommended path for corteX:**
1. Start with **Type 1** to establish baseline and unblock enterprise sales
2. Immediately begin collecting evidence for **Type 2** (the clock starts from Type 1 date)
3. Complete **Type 2** within 12 months of Type 1

### 1.3 Who Issues SOC 2 Reports?

Only **licensed CPA firms** with AICPA accreditation. Common qualifications auditors hold:
- **CPA** (Certified Public Accountant) -- required
- **CISA** (Certified Information Systems Auditor) -- common
- **CISSP** (Certified Information System Security Professional) -- valuable
- **CISM** (Certified Information Security Manager) -- valuable

**Startup-friendly audit firms (2026):**
- Prescient Security
- Johanson Group LLP
- ConstellationGRC
- Insight Assurance
- Linford & Company
- A-LIGN
- Schellman

**Selection criteria:**
- Experience auditing SaaS/AI companies specifically
- Familiarity with cloud environments (AWS, GCP, Azure)
- Willingness to do a readiness assessment before formal audit
- Competitive pricing for startups (some offer startup packages)
- Familiarity with compliance automation tools (Vanta, Drata, Secureframe)

---

## 2. Trust Service Criteria (TSC) Deep Dive

### 2.1 The Five TSC Categories

| # | Criteria | Required? | Description |
|---|----------|-----------|-------------|
| 1 | **Security** (Common Criteria) | **MANDATORY** | Protection against unauthorized access (logical + physical) |
| 2 | **Availability** | Optional | System uptime and accessibility per SLAs |
| 3 | **Processing Integrity** | Optional | Data processing is complete, valid, accurate, timely |
| 4 | **Confidentiality** | Optional | Protection of data classified as confidential |
| 5 | **Privacy** | Optional | PII collection, use, retention, disclosure, disposal per notice |

### 2.2 Recommended TSC for corteX (AI Agent SDK)

| Criteria | Include? | Rationale |
|----------|----------|-----------|
| **Security** | **YES (Required)** | Foundation of all SOC 2. Covers CC1-CC9. |
| **Confidentiality** | **YES** | Customers send prompts, API keys, and proprietary data through the SDK. Must prove this is protected. |
| **Availability** | **CONDITIONAL** | Include ONLY if corteX offers a hosted/SaaS server component with uptime SLAs. Skip for pure SDK distribution. |
| **Processing Integrity** | **YES** | AI agents process customer data -- must prove accurate, complete processing. Critical for enterprise trust. |
| **Privacy** | **CONDITIONAL** | Include if corteX collects/stores PII. For a pure SDK, may not apply directly. Include if the server component handles user data. |

**Recommendation**: Start with **Security + Confidentiality + Processing Integrity** (3 of 5).
Add Availability and Privacy if/when corteX launches hosted services.

---

## 3. Common Criteria (CC1-CC9) Detailed Controls

The Security TSC contains **9 Common Criteria** groups, mapped to the **COSO Framework**
(Committee of Sponsoring Organizations of the Treadway Commission). These 9 groups contain
approximately **33 individual sub-criteria** and typically require **60-100 specific controls**
and **200-300 pieces of evidence**.

### CC1: Control Environment (COSO Principles 1-5)

**Purpose**: Does the organization value integrity, ethics, and security?

| Sub-Criteria | COSO Principle | Description |
|--------------|----------------|-------------|
| **CC1.1** | Principle 1 | Entity demonstrates commitment to integrity and ethical values |
| **CC1.2** | Principle 2 | Board/oversight body demonstrates independence from management and exercises oversight of internal controls |
| **CC1.3** | Principle 3 | Management establishes structures, reporting lines, and appropriate authority/responsibility |
| **CC1.4** | Principle 4 | Entity demonstrates commitment to attract, develop, and retain competent individuals |
| **CC1.5** | Principle 5 | Entity holds individuals accountable for their internal control responsibilities |

**Controls needed:**
- Documented organizational chart reviewed annually
- Code of Conduct in Employee Handbook, signed by all employees
- Board or advisory board with independent oversight capability
- Defined roles and responsibilities for security functions
- Background checks for all employees and contractors
- Security awareness training program (documented attendance)
- Annual performance reviews that include security responsibilities
- Whistleblower / ethics reporting procedures
- Contractor/vendor NDAs and security acknowledgements

**Evidence for auditors:**
- Signed Code of Conduct acknowledgements
- Org chart with reporting lines
- Board/advisory meeting minutes
- Background check completion records
- Training completion records with dates and topics
- Job descriptions with security responsibilities
- Employee handbook (current, dated, version-controlled)

**corteX-specific considerations:**
- As a small startup, a formal "Board of Directors" may not exist -- an Advisory Board or documented founder oversight can suffice
- Document that the CEO/CTO reviews security controls quarterly
- Ensure all contributors (including contractors) sign security acknowledgements

---

### CC2: Communication and Information (COSO Principles 13-15)

**Purpose**: Is security information properly communicated internally and externally?

| Sub-Criteria | COSO Principle | Description |
|--------------|----------------|-------------|
| **CC2.1** | Principle 13 | Entity obtains/generates and uses relevant, quality information to support internal control |
| **CC2.2** | Principle 14 | Entity internally communicates information including objectives and responsibilities for internal control |
| **CC2.3** | Principle 15 | Entity communicates with external parties regarding matters affecting internal control |

**Controls needed:**
- Internal communication channels for security policies (Slack, email, wiki)
- Security policy distribution and acknowledgement tracking
- External communication procedures (customer notifications, breach disclosure)
- System status pages or customer portals for communicating security matters
- Documented process for informing new hires about security obligations
- Regular security updates to stakeholders (monthly/quarterly)

**Evidence for auditors:**
- Distribution records for security policies
- Internal communication logs (Slack channels, email distributions)
- Customer-facing security documentation (trust page, security FAQ)
- Onboarding checklist showing security communication steps
- Meeting minutes from security review sessions

**corteX-specific considerations:**
- Publish a public Trust/Security page on the corteX website
- Include security practices in SDK documentation
- Document how security incidents would be communicated to SDK users
- CHANGELOG entries for security-relevant SDK updates

---

### CC3: Risk Assessment (COSO Principles 6-9)

**Purpose**: Does the organization identify, analyze, and manage risks?

| Sub-Criteria | COSO Principle | Description |
|--------------|----------------|-------------|
| **CC3.1** | Principle 6 | Entity specifies objectives with sufficient clarity to enable identification and assessment of risks |
| **CC3.2** | Principle 7 | Entity identifies risks to achievement of objectives and analyzes them to determine management approach |
| **CC3.3** | Principle 8 | Entity considers the potential for fraud in assessing risks |
| **CC3.4** | Principle 9 | Entity identifies and assesses changes that could significantly impact the system of internal control |

**Controls needed:**
- Formal risk assessment process (at least annual)
- Risk register documenting identified risks, likelihood, impact, and mitigations
- Fraud risk assessment procedures
- Change impact assessment process (how system changes affect risk)
- Third-party/vendor risk assessment procedures
- Documented risk acceptance criteria and thresholds
- Regular review and update of risk assessments

**Evidence for auditors:**
- Risk register (spreadsheet or GRC tool output)
- Risk assessment methodology documentation
- Meeting minutes from risk review sessions
- Change impact assessments for major system changes
- Vendor risk assessment records
- Risk treatment/mitigation plans

**corteX-specific considerations:**
- Assess risks specific to AI agent systems: prompt injection, data leakage, model poisoning
- Assess risks of SDK dependencies (supply chain attacks on Python packages)
- Document risks of customer API key handling
- Assess multi-tenant data isolation risks
- Assess risks of on-prem deployment model (customer environments are untrusted)

**Example Risk Register Entries for corteX:**

| Risk ID | Risk Description | Likelihood | Impact | Mitigation | Owner |
|---------|-----------------|------------|--------|------------|-------|
| R-001 | Customer API keys exposed in logs | Medium | Critical | Log sanitization, key masking in all outputs | CTO |
| R-002 | Cross-tenant data leakage in shared memory | Low | Critical | Tenant isolation at session/memory layer | CTO |
| R-003 | Malicious prompt injection through SDK | Medium | High | Input validation, safety enforcement layer | Lead Dev |
| R-004 | Supply chain attack via Python dependency | Medium | High | Dependency pinning, audit, SBOM generation | DevOps |
| R-005 | Unauthorized SDK license usage | Medium | Medium | Ed25519 license validation, telemetry | CTO |
| R-006 | LLM provider data retention of customer prompts | Medium | High | Provider agreements, data processing addendums | CEO |

---

### CC4: Monitoring Activities (COSO Principles 16-17)

**Purpose**: Does the organization monitor and evaluate control effectiveness?

| Sub-Criteria | COSO Principle | Description |
|--------------|----------------|-------------|
| **CC4.1** | Principle 16 | Entity selects, develops, and performs ongoing/separate evaluations to ascertain whether controls are present and functioning |
| **CC4.2** | Principle 17 | Entity evaluates and communicates internal control deficiencies in a timely manner to parties responsible for corrective action |

**Controls needed:**
- Regular internal audits or control self-assessments
- Automated monitoring of security controls (SIEM, log analysis)
- Defined process for reporting control deficiencies
- Escalation procedures for identified control gaps
- Metrics/KPIs for control effectiveness
- Periodic penetration testing or vulnerability scanning

**Evidence for auditors:**
- Internal audit reports or self-assessment records
- SIEM/monitoring dashboard screenshots
- Vulnerability scan reports
- Penetration test results
- Deficiency tracking and remediation records
- Security metrics reports

---

### CC5: Control Activities (COSO Principles 10-12)

**Purpose**: Are proper controls, processes, and technologies in place to reduce risk?

| Sub-Criteria | COSO Principle | Description |
|--------------|----------------|-------------|
| **CC5.1** | Principle 10 | Entity selects and develops control activities that contribute to mitigation of risks to acceptable levels |
| **CC5.2** | Principle 11 | Entity selects and develops general control activities over technology to support achievement of objectives |
| **CC5.3** | Principle 12 | Entity deploys control activities through policies that establish expectations and procedures that put policies into action |

**Controls needed:**
- Segregation of duties (no single person has unchecked access)
- Automated controls where possible (CI/CD gates, automated testing)
- Technology-specific controls (firewall rules, access controls, encryption)
- Documented procedures for all critical operations
- Regular review and update of control activities
- Exception handling procedures

**Evidence for auditors:**
- Policy documents with version control
- CI/CD pipeline configurations showing security gates
- Access control matrices showing segregation of duties
- Automated test results
- Procedure documentation for critical operations

---

### CC6: Logical and Physical Access Controls (CRITICAL FOR SDK)

**Purpose**: Is access to systems and data properly restricted?

This is the **most important CC category** for a software company. It has 8 sub-criteria:

| Sub-Criteria | Description | Key Controls |
|--------------|-------------|--------------|
| **CC6.1** | Logical access security software, infrastructure, and architectures | Asset inventory, access control software, network segmentation, user identification/authentication, data encryption at rest, encryption key management |
| **CC6.2** | User registration and authorization | Credential creation with owner approval, credential removal when no longer needed, periodic access reviews |
| **CC6.3** | Access modification and role-based controls | RBAC implementation, separation of incompatible functions, access removal processes, periodic role reviews |
| **CC6.4** | Physical access controls | Data center access restrictions, physical access creation/modification/removal processes, periodic physical access reviews |
| **CC6.5** | Asset disposal and data sanitization | Data identification on disposed equipment, data made unreadable, equipment removed from control after sanitization |
| **CC6.6** | External access security | Communication channel restrictions, credential protection during transmission, additional auth for external access, boundary protection (firewalls, DMZ, IDS) |
| **CC6.7** | Data transmission and removal controls | DLP procedures, encryption for data in transit, removable media protection, mobile device safeguards |
| **CC6.8** | Malware prevention and detection | Software installation restrictions, unauthorized change detection, change control for software, antivirus/anti-malware, asset scanning before network deployment |

**Evidence for auditors (CC6):**
- Access control lists (ACLs) from all systems (GitHub, cloud, servers)
- MFA configuration screenshots
- User provisioning/deprovisioning procedures and records
- Quarterly access review records (manager sign-offs)
- Network architecture diagrams with segmentation
- Encryption configurations (at rest and in transit)
- Key management procedures
- Firewall rules and configurations
- IDS/IPS configurations and alert logs
- Antivirus/anti-malware deployment records
- Asset inventory spreadsheet
- Data sanitization procedures and records

**corteX-specific CC6 controls:**
- SDK repository access: GitHub branch protection, required reviews, signed commits
- Cloud infrastructure: MFA on all admin accounts, IAM roles with least privilege
- PyPI publishing: MFA required, limited to authorized maintainers
- Customer API keys: Never logged, never stored in plaintext, masked in all outputs
- Encryption: TLS 1.3 for all API communications, AES-256 for data at rest
- License keys: Ed25519 cryptographic validation

---

### CC7: System Operations

**Purpose**: Are systems monitored and is incident response in place?

| Sub-Criteria | Description | Key Controls |
|--------------|-------------|--------------|
| **CC7.1** | Vulnerability detection and monitoring | Configuration standards/hardening, infrastructure monitoring, CIS benchmark scanning |
| **CC7.2** | Anomaly detection and analysis | System component monitoring, anomaly identification, security event classification |
| **CC7.3** | Security event evaluation | Determine if events constitute security incidents, assess impact on objectives |
| **CC7.4** | Incident response | Incident response program, role assignment, containment, eradication, communication |
| **CC7.5** | Recovery from security incidents | System rebuild/restoration, root cause analysis, post-incident improvements, management communication |

**Evidence for auditors (CC7):**
- Monitoring tool configurations and dashboards (e.g., Datadog, CloudWatch, Grafana)
- Alert configurations and escalation procedures
- Vulnerability scan reports and remediation records
- Incident response plan document
- Incident response runbooks
- Post-incident review reports (even if no real incidents, document tabletop exercises)
- Backup and recovery test records
- System hardening documentation

---

### CC8: Change Management

**Purpose**: Are system changes properly tested, approved, and documented?

| Sub-Criteria | Description | Key Controls |
|--------------|-------------|--------------|
| **CC8.1** | Changes are authorized, designed, developed, configured, documented, tested, approved, and implemented | Change request and approval process, impact analysis, version control, testing requirements, rollback procedures, audit logging |

**Evidence for auditors (CC8):**
- Change management policy document
- Git history showing PR reviews and approvals
- CI/CD pipeline configurations with required checks
- Test results for deployments
- Release notes / changelogs
- Rollback procedures documentation
- Emergency change procedures
- Change advisory board (CAB) meeting records (if applicable)

**corteX-specific CC8 controls:**
- All code changes via Pull Request with at least 1 reviewer
- CI/CD pipeline: automated tests must pass before merge
- Version-controlled releases with semantic versioning
- PyPI release process documented with approval steps
- Database migration procedures (if server component)
- SDK breaking changes documented in CHANGELOG

---

### CC9: Risk Mitigation

**Purpose**: Are business disruption and vendor risks managed?

| Sub-Criteria | Description | Key Controls |
|--------------|-------------|--------------|
| **CC9.1** | Risk mitigation for business disruptions | Business continuity plan, disaster recovery plan, incident communication procedures |
| **CC9.2** | Vendor and business partner risk management | Vendor assessment process, vendor accountability, performance monitoring, issue resolution, termination procedures |

**Evidence for auditors (CC9):**
- Business continuity plan (BCP)
- Disaster recovery plan (DRP) with RTO/RPO targets
- BCP/DRP test results
- Vendor inventory list
- Vendor risk assessments (for critical vendors)
- Vendor contracts with security requirements
- Vendor performance review records
- Data processing agreements (DPAs) with vendors

**corteX-specific CC9 controls:**

Critical vendors to assess:
| Vendor | Service | Risk Level | Assessment Needed |
|--------|---------|------------|-------------------|
| OpenAI | LLM Provider | High | DPA, data retention policy, SOC 2 report review |
| Google (Gemini) | LLM Provider | High | DPA, data retention policy, SOC 2 report review |
| GitHub | Source code hosting | High | SOC 2 report review, access controls |
| PyPI | Package distribution | Medium | Publishing security, MFA requirements |
| Cloud Provider | Infrastructure | High | SOC 2 report review, shared responsibility model |
| Any local model providers | LLM Provider | Medium | Security assessment, data handling review |

---

## 4. AI Agent SDK Specific Controls

### 4.1 Important Distinction

**SOC 2 does NOT include controls unique to AI/ML systems.** The framework is
technology-neutral. However, AI systems introduce specific risk scenarios that must be
addressed through the existing CC1-CC9 framework.

For AI-specific governance, consider complementary frameworks:
- **ISO 42001** -- AI Management System
- **EU AI Act** -- Regulatory compliance (if serving EU customers)
- **NIST AI Risk Management Framework** -- Voluntary guidance

### 4.2 AI-Specific Risk Controls (Mapped to SOC 2 CC)

| Risk Area | SOC 2 Mapping | Control Description |
|-----------|---------------|---------------------|
| **Prompt Injection** | CC6.1, CC7.2 | Input validation and sanitization on all SDK inputs; anomaly detection for malicious prompts |
| **Data Leakage via LLM** | CC6.7, CC9.2 | Customer data isolation per session; vendor DPAs with LLM providers; no training on customer data clauses |
| **API Key Exposure** | CC6.1, CC6.7 | Keys never logged, never in plaintext storage, masked in all outputs, rotation support |
| **Cross-Tenant Contamination** | CC6.1, CC6.3 | Session-level memory isolation; tenant ID enforcement at every data access layer |
| **Model Output Reliability** | CC5.1 (Processing Integrity) | Output validation, confidence scoring, goal-tracking verification |
| **Supply Chain Attacks** | CC6.8, CC8.1 | Dependency pinning, SBOM generation, vulnerability scanning of all packages |
| **On-Prem Security** | CC2.3, CC6.1 | Security hardening guide for on-prem deployments; minimum security requirements documented |
| **Training Data Poisoning** | CC3.2, CC7.1 | If applicable: data provenance tracking, integrity verification of training data |
| **Unauthorized Tool Execution** | CC5.1, CC6.1 | Tool allowlisting per tenant; explicit opt-in required for external tool access |
| **Agent Loop / Resource Exhaustion** | CC7.1, CC7.2 | Loop detection (state hashing), resource limits, automatic termination |

### 4.3 Multi-Tenant Isolation Controls

For corteX specifically, given its multi-tenant enterprise architecture:

```
REQUIRED CONTROLS:
1. Session Isolation
   - Each tenant's agent session operates in isolated memory space
   - No shared state between tenants
   - Session IDs are cryptographically random (UUID4 minimum)

2. Data Access Controls
   - Tenant ID enforced at EVERY data access layer
   - Database queries ALWAYS filtered by tenant_id
   - Memory systems (working, short-term, long-term) scoped to tenant
   - Cross-tenant queries are architecturally impossible (not just policy-prevented)

3. API Key Isolation
   - Each tenant provides their own LLM API keys (BYOK model)
   - Keys encrypted at rest (AES-256)
   - Keys never cross tenant boundaries
   - Keys purged from memory after use

4. Audit Trail
   - All operations tagged with tenant_id
   - Audit logs cannot be modified
   - Log access restricted to authorized personnel
   - Tenant-specific log views available

5. Evidence for Auditors
   - Architecture diagrams showing tenant isolation
   - Code review showing tenant_id enforcement
   - Penetration test results showing cross-tenant isolation
   - Unit tests proving isolation (already in test suite)
```

### 4.4 API Key Management Controls

```
REQUIRED CONTROLS:
1. Storage
   - API keys stored in encrypted vault (not environment variables in production)
   - Encryption: AES-256-GCM or equivalent
   - Key derivation: PBKDF2 or Argon2

2. Transmission
   - TLS 1.3 for all key transmission
   - Keys never in URL parameters
   - Keys never in log output (masked as *****)

3. Lifecycle
   - Key rotation support built into SDK
   - Key expiration handling
   - Revocation procedures documented
   - Key usage auditing

4. Access
   - Least privilege: keys only accessible by components that need them
   - No shared API keys across tenants
   - Admin access to key storage requires MFA + approval

5. Monitoring
   - Alert on unusual API key usage patterns
   - Alert on failed authentication attempts
   - Alert on key access from unexpected locations
```

---

## 5. Required Policies and Documents

### 5.1 Core Policies (Must Have BEFORE Audit)

Every policy should include: Purpose, Scope, Definitions, Roles and Responsibilities,
Policy Statements, Procedures, Exceptions, Enforcement, Review Schedule, Version History.

| # | Policy | Maps to CC | Priority | Description |
|---|--------|-----------|----------|-------------|
| 1 | **Information Security Policy** | CC1, CC2, CC5 | CRITICAL | Master security policy covering organizational approach to infosec |
| 2 | **Access Control Policy** | CC5, CC6 | CRITICAL | Who gets access, how access is granted/revoked, RBAC definitions |
| 3 | **Change Management Policy** | CC8 | CRITICAL | How code/infrastructure changes are requested, reviewed, approved, deployed |
| 4 | **Incident Response Plan** | CC7 | CRITICAL | Step-by-step procedures for security incident handling |
| 5 | **Risk Assessment & Mitigation Plan** | CC3, CC9 | CRITICAL | Methodology for identifying, assessing, and treating risks |
| 6 | **Business Continuity Plan** | CC9 | HIGH | Maintaining operations during disruptions |
| 7 | **Disaster Recovery Plan** | CC7, CC9 | HIGH | Recovering systems after major incidents (with RTO/RPO) |
| 8 | **Data Classification Policy** | CC6, Confidentiality | HIGH | How data is categorized (Public, Internal, Confidential, Restricted) |
| 9 | **Vendor Management Policy** | CC9 | HIGH | Assessing and managing third-party risks |
| 10 | **Acceptable Use Policy** | CC1, CC6 | HIGH | Permitted use of company systems, devices, networks |
| 11 | **Encryption Policy** | CC6 | HIGH | What data is encrypted, algorithms used, key management |
| 12 | **Password Policy** | CC6 | HIGH | Complexity requirements, rotation, MFA requirements |
| 13 | **Code of Conduct** | CC1 | HIGH | Ethics, integrity, behavioral expectations |
| 14 | **Confidentiality Policy** | Confidentiality | MEDIUM | Handling of confidential information (customers, partners, internal) |
| 15 | **Physical Security Policy** | CC6 | MEDIUM | Office/data center physical access controls |
| 16 | **Remote Access Policy** | CC6 | MEDIUM | VPN requirements, remote work security |
| 17 | **Logging and Monitoring Policy** | CC4, CC7 | MEDIUM | What is logged, how logs are monitored, retention periods |
| 18 | **Software Development Lifecycle (SDLC) Policy** | CC8 | MEDIUM | Secure development practices, code review, testing |
| 19 | **Backup Policy** | CC7, Availability | MEDIUM | Backup frequency, retention, testing, recovery procedures |
| 20 | **Workstation Security Policy** | CC6 | MEDIUM | Endpoint security, disk encryption, screen locks |

### 5.2 Additional Documents Needed

| Document | Purpose |
|----------|---------|
| **System Description** | Detailed narrative of corteX services, components, boundaries, and data flows |
| **Network Architecture Diagram** | Visual showing all systems, connections, security boundaries |
| **Data Flow Diagram** | How customer data moves through the SDK (input to LLM to output) |
| **Asset Inventory** | Complete list of hardware, software, cloud resources |
| **Vendor Inventory** | List of all third-party services with risk classifications |
| **Risk Register** | Living document of identified risks and mitigations |
| **Employee Roster with Roles** | Who has access to what, with dates |
| **Training Records** | Security awareness training completion logs |
| **Penetration Test Report** | Results from external security testing |
| **Vulnerability Scan Reports** | Regular scan results and remediation tracking |

### 5.3 Data Classification Scheme for corteX

| Classification | Description | Examples | Handling Requirements |
|---------------|-------------|----------|----------------------|
| **Restricted** | Highest sensitivity, significant harm if disclosed | Customer API keys, encryption keys, license signing keys | Encrypted at rest + transit, MFA access, audit logging, need-to-know basis |
| **Confidential** | Business-sensitive, harm if disclosed | Customer prompts/data, source code, internal architecture docs, employee data | Encrypted at rest + transit, role-based access, audit logging |
| **Internal** | Not for public, limited harm if disclosed | Internal procedures, meeting notes, non-sensitive configs | Access limited to employees, standard security controls |
| **Public** | Intended for public consumption | SDK documentation, public API specs, marketing materials, open-source components | No access restrictions, integrity controls only |

---

## 6. Technical Controls Checklist

### 6.1 Identity and Access Management

- [ ] Multi-factor authentication (MFA) on ALL systems (GitHub, cloud, email, admin panels)
- [ ] Single Sign-On (SSO) where possible
- [ ] Role-Based Access Control (RBAC) documented and enforced
- [ ] Principle of least privilege applied everywhere
- [ ] User provisioning procedure documented
- [ ] User deprovisioning procedure (within 24 hours of departure)
- [ ] Quarterly access reviews with manager sign-off
- [ ] Unique accounts per person (no shared accounts)
- [ ] Service accounts documented with owners assigned
- [ ] Password policy enforced (minimum 12 chars, complexity, no reuse)
- [ ] Privileged access management (PAM) for admin accounts

### 6.2 Network Security

- [ ] Firewall configured with default-deny rules
- [ ] Network segmentation (production vs development vs corporate)
- [ ] Intrusion Detection/Prevention System (IDS/IPS) deployed
- [ ] VPN required for remote access to internal systems
- [ ] TLS 1.2+ (preferably 1.3) for all external communications
- [ ] DDoS protection for public-facing services
- [ ] DNS security (DNSSEC if applicable)
- [ ] Network monitoring with alerting

### 6.3 Data Protection

- [ ] Encryption at rest (AES-256 for databases, disk encryption for endpoints)
- [ ] Encryption in transit (TLS 1.3 for APIs, SSH for administrative access)
- [ ] Key management procedures documented
- [ ] Data Loss Prevention (DLP) controls
- [ ] Database access restricted and audited
- [ ] Backup encryption
- [ ] Secure data disposal procedures

### 6.4 Application Security

- [ ] Secure SDLC documented and followed
- [ ] Code reviews required for all changes (PR reviews)
- [ ] Static Application Security Testing (SAST) in CI/CD
- [ ] Dependency scanning (Dependabot, Snyk, or equivalent)
- [ ] Container scanning (if using Docker)
- [ ] Secret scanning in repositories (prevent accidental key commits)
- [ ] Branch protection rules (no direct push to main)
- [ ] Signed commits (recommended)
- [ ] SBOM (Software Bill of Materials) generation
- [ ] Regular penetration testing (at least annually)

### 6.5 Endpoint Security

- [ ] Endpoint Detection and Response (EDR) on all company devices
- [ ] Full disk encryption on all laptops
- [ ] Automatic screen lock (5 minutes or less)
- [ ] Automatic OS updates enabled
- [ ] Antivirus/anti-malware deployed
- [ ] Mobile Device Management (MDM) if company phones used

### 6.6 Logging and Monitoring

- [ ] Centralized log management (SIEM or equivalent)
- [ ] Authentication events logged
- [ ] Authorization failures logged
- [ ] Administrative actions logged
- [ ] Data access logged
- [ ] System changes logged
- [ ] Log integrity protection (append-only, tamper-evident)
- [ ] Log retention: minimum 90 days online, 1 year archived
- [ ] Alerting on security-relevant events
- [ ] Regular log review process

### 6.7 Backup and Recovery

- [ ] Automated backups for all critical data
- [ ] Backup encryption
- [ ] Off-site/cross-region backup storage
- [ ] Recovery Time Objective (RTO) defined and documented
- [ ] Recovery Point Objective (RPO) defined and documented
- [ ] Quarterly backup restoration tests
- [ ] Documented recovery procedures

### 6.8 Vulnerability Management

- [ ] Regular vulnerability scanning (at least monthly)
- [ ] Penetration testing (at least annually, ideally semi-annually)
- [ ] Vulnerability remediation SLAs (Critical: 24h, High: 7d, Medium: 30d, Low: 90d)
- [ ] Patch management process documented
- [ ] Third-party dependency monitoring
- [ ] CVE tracking for all dependencies

---

## 7. The Audit Process Step-by-Step

### Phase 1: Readiness Assessment (Optional but STRONGLY Recommended)

**Timeline**: 2-4 weeks
**Cost**: $5,000 - $15,000 (some auditors include this with the audit)

**What happens:**
1. Auditor reviews your current controls against Trust Services Criteria
2. Gap analysis identifying missing controls, policies, or evidence
3. Prioritized remediation plan
4. Scoping discussion (which TSC to include)

**Why do it:**
- Identifies gaps BEFORE the expensive formal audit
- Prevents audit failures and costly delays
- Sets realistic timeline for formal audit readiness
- Organizations that do readiness assessments have significantly smoother audits

### Phase 2: Remediation (The Hard Work)

**Timeline**: 2-4 months (depending on gaps found)

**What happens:**
1. Write/update all required policies
2. Implement technical controls
3. Set up evidence collection (automated if using compliance platform)
4. Train employees on new policies and procedures
5. Conduct internal risk assessment
6. Document system description
7. Begin collecting evidence (screenshots, logs, records)

**Common remediation tasks for startups:**
- Writing policies from scratch (most startups have none)
- Implementing MFA everywhere
- Setting up centralized logging
- Formalizing change management (even if you already do PR reviews)
- Setting up access reviews
- Getting penetration testing done
- Formalizing incident response procedures

### Phase 3: Evidence Collection

**Timeline**: Ongoing (should start during remediation)

**For Type 1, you need evidence showing controls are IN PLACE at the audit date:**
- Current policy documents (dated and approved)
- Current system configurations (screenshots)
- Current access lists
- Current network diagrams
- Training records
- Risk assessment documentation
- Vendor assessments

**Evidence collection tips:**
- Use a compliance automation platform (Vanta, Drata, Secureframe)
- Automate as much evidence collection as possible
- Maintain a shared drive/folder structure organized by CC category
- Date and label all evidence
- Keep evidence current (auditors will check dates)

### Phase 4: Formal Audit (Type 1)

**Timeline**: 4-8 weeks

**What happens:**
1. **Planning and scoping** (Week 1)
   - Auditor confirms scope, TSC selection, and system boundaries
   - Information request list (IRL) provided -- all evidence/documents needed

2. **Fieldwork** (Weeks 2-5)
   - Auditor reviews all policies and documentation
   - Auditor examines technical controls (configurations, screenshots, logs)
   - Auditor conducts interviews with key personnel
   - Auditor tests control design (are controls properly designed to meet criteria?)
   - Note: For Type 1, auditors test DESIGN only, not operational effectiveness

3. **Draft report review** (Week 6)
   - Auditor provides draft report for management review
   - Management reviews for factual accuracy
   - Management provides responses to any identified exceptions

4. **Final report issuance** (Weeks 7-8)
   - Auditor finalizes report
   - Management signs assertion letter
   - Final SOC 2 Type 1 report delivered

### Phase 5: Post-Audit

**What to do immediately after Type 1:**
1. Share report with customers requesting it (under NDA)
2. Begin operational evidence collection for Type 2
3. Address any exceptions noted in the report
4. Set up continuous monitoring for all controls
5. Schedule Type 2 audit (typically 6-12 months after Type 1 date)

### Common Audit Failures for Startups

| Failure | Why It Happens | How to Prevent |
|---------|---------------|----------------|
| **Missing documentation** | Policies never written or out of date | Write and approve all policies before audit |
| **Inconsistent access reviews** | No regular cadence established | Set up quarterly reviews with calendar reminders |
| **No formal change management** | "We use PRs" but no documented policy | Document the PR process as formal change management |
| **Missing training records** | Training happened but was not tracked | Log all training with dates, attendees, topics |
| **Shared accounts** | Developer convenience | Eliminate all shared accounts before audit |
| **Missing MFA** | Not enabled on all systems | Audit every system for MFA, no exceptions |
| **No vendor assessments** | Never done formal vendor reviews | Request SOC 2 reports from critical vendors |
| **Manual processes without evidence** | Tasks done but not recorded | Automate evidence collection; if manual, log it |
| **Incomplete system description** | Rushed or vague | Invest time in accurate, detailed system description |
| **No incident response testing** | Plan exists but never tested | Conduct at least one tabletop exercise |

### SOC 2 Type 1 Report Structure

The final report contains these sections:

**Section 1: Independent Service Auditor's Report (Opinion Letter)**
- Auditor's formal opinion on control design
- Opinion types: Unqualified (pass), Qualified (pass with issues), Adverse (fail), Disclaimer
- Target: **Unqualified opinion**

**Section 2: Management's Assertion**
- Letter from company leadership
- Summary of product/services
- Statement that controls are fairly presented
- Signed by CEO or equivalent

**Section 3: System Description**
- Detailed narrative of your services
- System boundaries and components
- Infrastructure, software, people, procedures, data
- Principal service commitments and system requirements

**Section 4: Description of Controls**
- Complete list of controls tested
- Mapped to applicable Trust Service Criteria
- For Type 1: controls listed but NO test results or exceptions shown
- (Type 2 would include test procedures, results, and exceptions)

**Section 5: Additional Information (Optional)**
- Management's response to specific items
- Complementary User Entity Controls (CUECs) -- what YOUR customers must do

---

## 8. Compliance Automation Platforms

### Platform Comparison (2026)

| Feature | Vanta | Drata | Secureframe |
|---------|-------|-------|-------------|
| **Best for** | Early-stage startups | Scale-up companies | Hands-on guidance |
| **Frameworks** | 30+ | 20+ | 25+ |
| **Pricing** | $10K-$50K+/year | $10K-$40K+/year | $15K-$25K/year |
| **Automation** | Good, some manual | Best real-time monitoring | Moderate, emphasis on guidance |
| **Setup** | Fastest | Fast | Moderate (more handholding) |
| **Integrations** | Broadest library | Strong multi-framework mapping | Pre-templated approach |
| **Support** | Self-service + support | Balanced | White-glove onboarding |

### Recommendation for corteX

**Vanta** is likely the best fit for a startup pursuing first SOC 2:
- Fastest time to audit-ready
- Broad integration library (GitHub, cloud providers, etc.)
- Startup-friendly pricing
- Most popular among early-stage companies

**Budget**: Plan for $10,000-$15,000/year for the platform

### What Compliance Platforms Do

- Automatically collect evidence from connected systems (GitHub, AWS/GCP, Okta, etc.)
- Continuous control monitoring (alert when controls fail)
- Policy template library (pre-written policies you can customize)
- Employee training modules (security awareness)
- Vendor management module
- Risk assessment tools
- Auditor collaboration portal
- Readiness scoring (how audit-ready you are)

---

## 9. 2025-2026 Updates and AI Considerations

### AICPA Updates

**2022 Revision (Still Current in 2026):**
- The AICPA issued revised **Points of Focus** for the Trust Services Criteria
- The underlying criteria (from 2017) were NOT changed
- Revised points of focus address evolving technologies, threats, and regulations
- No new criteria were added; existing criteria were clarified

**November 2025 Guidance Revision:**
- The AICPA revised guidance on applying Trust Services Criteria and SOC 2 Description Criteria
- Service organizations may need to update their controls and system descriptions
- Focus on how controls are described and evaluated, not new criteria themselves

**Key takeaway**: The SOC 2 framework has NOT been updated specifically for AI. It remains
technology-neutral. AI companies apply the same criteria as any SaaS company, but must
address AI-specific risks through the existing framework.

### AI-Specific Guidance Status

| Framework | Status | Relevance to corteX |
|-----------|--------|---------------------|
| **SOC 2** | No AI-specific criteria | Apply existing CC1-CC9 to AI risk scenarios |
| **ISO 42001** | Published 2023, adoption growing | AI Management System -- complementary to SOC 2 |
| **EU AI Act** | Effective 2024-2027 (phased) | If serving EU customers with high-risk AI |
| **NIST AI RMF** | Published 2023, voluntary | Good framework for AI risk management |
| **AICPA AI Guidance** | No formal framework yet | CPAs exploring role as AI system evaluators |

### AI Ethics Considerations (Beyond SOC 2)

While not part of SOC 2, enterprise customers increasingly ask about:
- **Bias detection and mitigation**: How does the SDK handle biased outputs?
- **Explainability**: Can agent decisions be traced and explained?
- **Human oversight**: Are there human-in-the-loop controls?
- **Data provenance**: Where does training data come from?
- **Output reliability**: How are hallucinations detected and prevented?

corteX's brain-inspired architecture provides natural answers to several of these:
- **Goal tracking** = built-in output reliability verification
- **Feedback loops** = continuous improvement and bias correction
- **Audit logging** = decision traceability and explainability
- **Safety enforcement** = human oversight capabilities
- **Loop prevention** = predictable, controlled agent behavior

---

## 10. Cost and Timeline Estimates

### Total Cost Breakdown for corteX (Estimated)

| Cost Category | Low Estimate | High Estimate | Notes |
|--------------|-------------|---------------|-------|
| **Readiness Assessment** | $5,000 | $15,000 | Optional but recommended |
| **Compliance Platform** | $10,000/yr | $25,000/yr | Vanta, Drata, or Secureframe |
| **Auditor Fee (Type 1)** | $12,000 | $30,000 | CPA firm audit fee |
| **Security Tools** | $5,000/yr | $20,000/yr | SIEM, EDR, vulnerability scanning |
| **Penetration Testing** | $5,000 | $15,000 | External pentest firm |
| **Legal Review** | $5,000 | $10,000 | Policy review by legal counsel |
| **Employee Training** | $500 | $3,000 | Security awareness training |
| **Internal Time** | 100-200 hours | 300+ hours | Opportunity cost of team time |
| **TOTAL (Year 1)** | **$42,500** | **$118,000** | Including all setup costs |
| **TOTAL (Ongoing/Year)** | **$25,000** | **$60,000** | Platform + tools + annual audit |

### Timeline (Aggressive but Realistic)

```
Month 1-2:  PREPARATION
  - Select compliance platform (Vanta recommended)
  - Select auditor and schedule readiness assessment
  - Begin writing policies (use platform templates)
  - Implement MFA everywhere
  - Set up centralized logging

Month 2-3:  REMEDIATION
  - Complete all policy documents
  - Implement remaining technical controls
  - Conduct risk assessment
  - Set up employee training
  - Begin vendor assessments
  - Complete system description document

Month 3-4:  EVIDENCE COLLECTION
  - Connect compliance platform to all systems
  - Verify automated evidence collection working
  - Conduct penetration test
  - Complete access reviews
  - Conduct tabletop incident response exercise
  - Fill any remaining gaps from readiness assessment

Month 4-5:  FORMAL AUDIT
  - Provide all evidence to auditor
  - Participate in auditor interviews
  - Address any questions or additional evidence requests
  - Review draft report

Month 5-6:  REPORT + TYPE 2 PREP
  - Receive final SOC 2 Type 1 report
  - Share with customers
  - Begin Type 2 observation period
  - Maintain all controls continuously
```

---

## 11. corteX-Specific Implementation Plan

### What corteX Already Has (Advantages)

Based on the current codebase and architecture:

| Existing Capability | SOC 2 Relevance |
|---------------------|-----------------|
| Ed25519 license validation | CC6.1 - Cryptographic controls |
| Safety enforcement layer | CC5.1 - Control activities for AI outputs |
| Input validation | CC6.1, Processing Integrity - Data validation |
| Multi-tenant session isolation | CC6.1, CC6.3 - Tenant data isolation |
| 3,355 passing unit tests | CC8.1 - Change management testing evidence |
| Comprehensive documentation (97 pages) | CC2 - Communication and information |
| BYOK (Bring Your Own Key) model | CC6.1 - API key management |
| Loop prevention (state hashing) | CC7.1 - System operations monitoring |
| Goal tracking | Processing Integrity - Output verification |
| Audit logging capabilities | CC4, CC7 - Monitoring and operations |
| Provider-agnostic design | CC9 - Risk mitigation (no vendor lock-in) |
| On-prem capability | CC6 - Customer controls their own data |

### What corteX Needs to Implement

| Gap | Priority | Effort | CC Mapping |
|-----|----------|--------|------------|
| All 20 policies documented | CRITICAL | High | CC1-CC9 |
| MFA on all company systems | CRITICAL | Low | CC6.2, CC6.6 |
| Centralized logging (SIEM) | CRITICAL | Medium | CC4, CC7 |
| Formal incident response plan | CRITICAL | Medium | CC7.4 |
| Access review process | HIGH | Low | CC6.2, CC6.3 |
| Risk assessment documentation | HIGH | Medium | CC3 |
| Vendor assessment process | HIGH | Medium | CC9.2 |
| Penetration testing | HIGH | Low (outsource) | CC4.1, CC7.1 |
| Employee security training program | HIGH | Low | CC1.4 |
| System description document | HIGH | Medium | Report requirement |
| Compliance automation platform | HIGH | Low (purchase) | All CC |
| Backup and recovery testing | MEDIUM | Low | CC7.5, CC9.1 |
| Network architecture documentation | MEDIUM | Low | CC6.1 |
| Data flow diagrams | MEDIUM | Low | CC6.1, CC6.7 |
| Formal change management policy | MEDIUM | Low (document existing PR process) | CC8.1 |

### Complementary User Entity Controls (CUECs)

These are controls that corteX's CUSTOMERS must implement. They will be listed in the
SOC 2 report. For an SDK, this is important because much of the security depends on
how customers deploy and use the SDK.

**CUECs for corteX customers:**
1. Customers must securely store their own LLM API keys
2. Customers must implement authentication/authorization for their end users
3. Customers must secure their deployment environment (on-prem or cloud)
4. Customers must keep the corteX SDK updated to latest security patches
5. Customers must configure appropriate resource limits for agent execution
6. Customers must implement their own data backup procedures
7. Customers must comply with applicable data protection regulations
8. Customers must restrict access to the corteX SDK configuration to authorized personnel
9. Customers must monitor their agent deployments for anomalous behavior
10. Customers must report any suspected security incidents to Questo

---

## Appendix A: Policy Template Structure

Every SOC 2 policy should follow this structure:

```
POLICY: [Policy Name]
Version: [X.Y]
Effective Date: [YYYY-MM-DD]
Last Reviewed: [YYYY-MM-DD]
Next Review: [YYYY-MM-DD]
Owner: [Name / Role]
Approved By: [Name / Role]

1. PURPOSE
   Why this policy exists and what it aims to achieve.

2. SCOPE
   Who and what this policy applies to (employees, contractors, systems, data).

3. DEFINITIONS
   Key terms used in the policy.

4. ROLES AND RESPONSIBILITIES
   Who is responsible for what under this policy.

5. POLICY STATEMENTS
   The actual policy rules and requirements.

6. PROCEDURES
   Step-by-step instructions for implementing the policy.

7. EXCEPTIONS
   How to request exceptions, who approves, documentation required.

8. ENFORCEMENT
   Consequences of non-compliance.

9. RELATED DOCUMENTS
   Links to related policies, standards, and procedures.

10. REVISION HISTORY
    Table of all changes with dates and authors.
```

---

## Appendix B: Auditor Interview Preparation

Common questions auditors ask during interviews:

**For CEO/Leadership:**
- How does leadership demonstrate commitment to security?
- How is the board/advisory board involved in security oversight?
- How are security risks communicated to leadership?
- What is the organization's risk appetite?

**For CTO/Engineering:**
- Walk me through the change management process
- How are access rights granted and revoked?
- Describe your incident response process
- How do you handle vulnerabilities in dependencies?
- How is customer data isolated between tenants?
- How are API keys managed and protected?

**For All Employees:**
- When was your last security training?
- Do you know how to report a security incident?
- Are you familiar with the company's information security policy?
- Do you know who is responsible for security in the organization?

---

## Appendix C: Evidence Collection Checklist by CC Category

### CC1 Evidence
- [ ] Signed Code of Conduct (all employees)
- [ ] Organization chart
- [ ] Board/advisory meeting minutes
- [ ] Background check records
- [ ] Training completion records
- [ ] Job descriptions with security duties
- [ ] Employee handbook (current version)
- [ ] Performance review template showing security accountability

### CC2 Evidence
- [ ] Policy distribution records
- [ ] Internal security communication logs
- [ ] External security communication (trust page, customer notices)
- [ ] Onboarding checklist with security steps
- [ ] Security meeting minutes

### CC3 Evidence
- [ ] Risk register
- [ ] Risk assessment methodology document
- [ ] Risk review meeting minutes
- [ ] Change impact assessments
- [ ] Vendor risk assessments
- [ ] Fraud risk assessment

### CC4 Evidence
- [ ] Internal audit/self-assessment reports
- [ ] Monitoring dashboard screenshots
- [ ] Vulnerability scan reports
- [ ] Penetration test report
- [ ] Deficiency tracking records
- [ ] Security metrics reports

### CC5 Evidence
- [ ] All policy documents (20 policies listed in Section 5)
- [ ] CI/CD configuration showing security gates
- [ ] Segregation of duties matrix
- [ ] Automated test results
- [ ] Control exception records

### CC6 Evidence
- [ ] Access control lists from ALL systems
- [ ] MFA configuration screenshots
- [ ] User provisioning/deprovisioning records
- [ ] Quarterly access review records with sign-offs
- [ ] Network architecture diagram
- [ ] Encryption configuration documentation
- [ ] Key management procedures
- [ ] Firewall rules
- [ ] IDS/IPS configurations
- [ ] Antivirus deployment records
- [ ] Asset inventory
- [ ] Data sanitization records

### CC7 Evidence
- [ ] Monitoring tool configurations and dashboards
- [ ] Alert configurations
- [ ] Vulnerability scan results
- [ ] Incident response plan
- [ ] Incident response runbooks
- [ ] Post-incident reviews OR tabletop exercise records
- [ ] Backup test records
- [ ] System hardening documentation

### CC8 Evidence
- [ ] Change management policy
- [ ] Git PR review history
- [ ] CI/CD pipeline configurations
- [ ] Test results for deployments
- [ ] Release notes / CHANGELOG
- [ ] Rollback procedures
- [ ] Emergency change procedures

### CC9 Evidence
- [ ] Business continuity plan
- [ ] Disaster recovery plan (with RTO/RPO)
- [ ] BCP/DRP test results
- [ ] Vendor inventory
- [ ] Vendor risk assessments
- [ ] Vendor contracts with security clauses
- [ ] Data processing agreements

---

## Sources and References

### SOC 2 Framework and General Guidance
- [SOC 2 Compliance: 2026 Complete Guide - StrongDM](https://www.strongdm.com/soc2/compliance)
- [SOC 2 Compliance Checklist: A Step-by-Step Guide For 2026 - Sprinto](https://sprinto.com/blog/soc-2-compliance-checklist/)
- [SOC 2 Compliance Requirements - Secureframe](https://secureframe.com/hub/soc-2/requirements)
- [SOC 2 Compliance Requirements - Onspring](https://onspring.com/resources/guide/soc-2-compliance-requirements-and-how-to-meet-them/)

### Trust Service Criteria
- [Trust Services Criteria for SOC 2 - Drata](https://drata.com/blog/trust-services-criteria)
- [SOC 2 Trust Services Criteria - Bright Defense](https://www.brightdefense.com/resources/soc-2-trust-services-criteria/)
- [SOC 2 Trust Services Criteria Guide - Cherry Bekaert](https://www.cbh.com/insights/articles/soc-2-trust-services-criteria-guide/)
- [2025 Trust Services Criteria for SOC 2 - Secureframe](https://secureframe.com/hub/soc-2/trust-services-criteria)

### Common Criteria Details
- [SOC 2 Common Criteria - Secureframe](https://secureframe.com/hub/soc-2/common-criteria)
- [SOC 2 Controls List for 2026 - Bright Defense](https://www.brightdefense.com/resources/soc-2-controls-list/)
- [SOC 2 Common Criteria: CC-Series Explained - Compass IT Compliance](https://www.compassitc.com/blog/soc-2-common-criteria-list-cc-series-explained)
- [SOC 2 Controls CC6: Logical & Physical Access - Hicomply](https://www.hicomply.com/hub/soc-2-controls-cc6-logical-and-physical-access-controls)

### AI-Specific Compliance
- [SOC 2 for AI Companies: Complete Guide - Comp AI](https://trycomp.ai/soc-2-for-ai-companies)
- [SOC 2 Compliance for AI Platforms - Compass IT Compliance](https://www.compassitc.com/blog/achieving-soc-2-compliance-for-artificial-intelligence-ai-platforms)
- [SOC 2 Compliance in the Age of AI - Userfront](https://userfront.com/blog/soc-2-ai-compliance)
- [Auditing AI Platforms: SOC 2 Considerations - Linford & Company](https://linfordco.com/blog/soc-2-audit-considerations-ai-ml-platforms/)

### Cost and Audit Process
- [How Much Does a SOC 2 Audit Cost in 2026? - Bright Defense](https://www.brightdefense.com/resources/soc-2-audit-costs/)
- [SOC 2 Budget: How Much Does SOC 2 Cost in 2026? - StrongDM](https://www.strongdm.com/blog/how-much-does-soc-2-cost)
- [SOC 2 Audit Cost - Secureframe](https://secureframe.com/hub/soc-2/audit-cost)
- [SOC 2 Readiness Assessment Guide - GRSee](https://grsee.com/resources/soc/soc-2-readiness-assessment-complete-guide-with-cost-and-timeline/)

### Policies and Templates
- [Complete List of SOC 2 Policies and Procedures - Secureframe](https://secureframe.com/hub/soc-2/policies-and-procedures)
- [12 Commonly Recommended Security Policies for SOC 2 - Drata](https://drata.com/blog/soc-2-policies)
- [SOC 2 Policy Templates - StrongDM](https://www.strongdm.com/blog/soc-2-policy-templates)

### Report Structure
- [SOC 2 Report Example Explained - Secureframe](https://secureframe.com/hub/soc-2/report-example)
- [Key Sections of a SOC 2 Report - Scytale](https://scytale.ai/center/soc-2/exploring-the-key-sections-of-a-soc-2-report-in-under-4-minutes/)
- [Complete Guide to SOC 2 Reports - Vanta](https://www.vanta.com/collection/soc-2/soc-2-report-example)

### Auditor Selection
- [Who Performs a SOC 2 Audit? - Secureframe](https://secureframe.com/hub/soc-2/who-performs-a-soc-2-audit)
- [Best SOC 2 Auditors for 2026 - Bright Defense](https://www.brightdefense.com/resources/soc-2-audit-firms/)
- [Ranking the Best SOC 2 Auditors for 2026](https://bestsoc2auditors.com/)

### Compliance Automation
- [Best SOC 2 Compliance Software for 2026 - Vanta](https://www.vanta.com/resources/best-soc-2-compliance-software)
- [10 Best SOC 2 Compliance Software for 2026 - Bright Defense](https://www.brightdefense.com/resources/best-soc-2-compliance-software/)
- [Secureframe vs Vanta vs Drata - Sprinto](https://sprinto.com/blog/secureframe-vs-vanta-vs-drata/)

### AICPA Updates
- [AICPA SOC 2 Official Page](https://www.aicpa-cima.com/topic/audit-assurance/audit-and-assurance-greater-than-soc-2)
- [AICPA Revises SOC 2 Guidance - EY](https://www.ey.com/en_us/technical/accountinglink/to-the-point-aicpa-revises-guidance-on-applying-its-trust-services-criteria-and-soc-2-description-criteria)
- [CPAs as AI System Evaluators - Journal of Accountancy](https://www.journalofaccountancy.com/issues/2025/nov/a-new-frontier-cpas-as-ai-system-evaluators/)
