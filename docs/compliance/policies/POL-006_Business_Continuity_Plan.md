# Business Continuity Plan

**Document ID**: POL-006
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Business Continuity Plan (BCP) establishes the framework for maintaining essential Questo Ltd business operations during and after a disruptive event. It defines business impact analysis results, recovery priorities, continuity strategies, and communication procedures specific to the corteX AI Agent SDK platform and Questo's operational environment.

Because the corteX SDK is designed as an on-premises-first product distributed via PyPI, the continuity posture differs from traditional SaaS offerings. Customer deployments operate independently of Questo infrastructure. This plan focuses on continuity of Questo's development, distribution, support, and documentation services.

**SOC 2 Mapping**: CC9.1 (Risk Mitigation), A1.1-A1.3 (Availability)

## 2. Scope

This plan applies to:

- **Business Functions**: All Questo Ltd business functions including SDK development, quality assurance, SDK distribution (PyPI), documentation hosting, customer support, license management, and corporate operations.
- **Personnel**: All employees and contractors of Questo Ltd.
- **Systems**: All systems supporting Questo operations, including development infrastructure, CI/CD pipelines, cloud hosting, communication platforms, and the corteX documentation site.
- **Third Parties**: Critical vendors whose services are essential to business continuity (GitHub, cloud providers, LLM providers, PyPI).

This plan does not cover customer-side corteX deployments, which are the responsibility of each customer per their own BCP. However, Questo provides guidance for customer-side continuity through the SDK documentation.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **BCP** | Business Continuity Plan; the documented strategy for maintaining operations during disruption |
| **BIA** | Business Impact Analysis; the assessment of the impact of disruptions to business functions |
| **RTO** | Recovery Time Objective; the maximum acceptable time to restore a function after disruption |
| **RPO** | Recovery Point Objective; the maximum acceptable data loss measured in time |
| **MTPD** | Maximum Tolerable Period of Disruption; the absolute limit before unrecoverable business harm |
| **Critical Function** | A business function whose loss would cause unacceptable harm within hours or days |
| **Essential Function** | A business function required for sustained operations but tolerant of brief interruption |
| **Disruption** | Any event that interrupts normal business operations |
| **Failover** | The process of switching to a redundant system or site when the primary becomes unavailable |

## 4. Roles and Responsibilities

### 4.1 CEO (Business Continuity Sponsor)

- Approves this plan and allocates resources for business continuity
- Serves as the primary authority for decisions during an organizational disruption
- Authorizes activation of this plan and communication with external stakeholders
- Reviews the BCP annually and after any activation

### 4.2 CTO (Business Continuity Coordinator)

- Owns this plan and ensures its implementation, testing, and maintenance
- Coordinates technical recovery activities during a disruption
- Leads the Business Continuity Team during an event
- Ensures recovery procedures are documented, tested, and current

### 4.3 Security Lead

- Ensures security controls are maintained during continuity operations
- Validates that recovery procedures do not introduce security vulnerabilities
- Coordinates with the CTO on the security aspects of alternative operating procedures

### 4.4 Engineering Team

- Executes technical recovery procedures as directed by the CTO
- Maintains familiarity with recovery procedures through annual testing
- Reports disruptions to the CTO immediately upon detection

### 4.5 All Personnel

- Familiarize themselves with this plan and their role during a disruption
- Participate in annual continuity testing exercises
- Report disruptions or potential disruptions through the established channels

## 5. Policy Statements

### 5.1 Business Impact Analysis

Questo Ltd has identified the following business functions, their criticality, and recovery objectives:

#### 5.1.1 Critical Functions

| Function | Description | RTO | RPO | MTPD |
|----------|-------------|-----|-----|------|
| **SDK Source Code** | The corteX codebase on GitHub | 1 hour | 0 (real-time replication) | 4 hours |
| **CI/CD Pipeline** | Automated testing and build pipeline | 4 hours | 1 hour | 24 hours |
| **PyPI Distribution** | SDK package availability on PyPI | N/A (PyPI-managed) | N/A | 48 hours |
| **License Validation** | Ed25519 license key generation and validation infrastructure | 4 hours | 24 hours | 72 hours |

#### 5.1.2 Essential Functions

| Function | Description | RTO | RPO | MTPD |
|----------|-------------|-----|-----|------|
| **Documentation Site** | MkDocs Material site at docs.cortex-ai.com | 8 hours | 24 hours | 72 hours |
| **Customer Support** | Communication channels for SDK users | 4 hours | N/A | 24 hours |
| **Internal Communication** | Team messaging and email | 2 hours | N/A | 8 hours |
| **Development Environments** | Individual developer workstations and tools | 24 hours | 8 hours | 72 hours |

#### 5.1.3 Supporting Functions

| Function | Description | RTO | RPO | MTPD |
|----------|-------------|-----|-----|------|
| **Corporate Email** | Business email services | 4 hours | N/A | 24 hours |
| **Financial Systems** | Accounting and billing | 72 hours | 24 hours | 1 week |
| **Marketing/Sales** | Website, CRM, marketing tools | 72 hours | 24 hours | 1 week |

### 5.2 On-Premises-First Continuity Advantage

The corteX SDK's on-premises-first architecture provides an inherent continuity advantage for customers:

1. **Independent Operation**: Customer corteX deployments operate independently of Questo infrastructure. A Questo outage does not affect running customer agents.
2. **No Runtime Dependency**: The SDK does not phone home or require Questo services to function. License validation uses offline Ed25519 signature verification.
3. **Local LLM Support**: Customers using local OpenAI-compatible models (e.g., Ollama, vLLM) can continue agent operations even during LLM provider outages.
4. **Multi-Provider Failover**: The SDK's `ProviderHealth` module (`corteX/engine/provider_health.py`) supports automatic failover between LLM providers, mitigating single-provider outages.

### 5.3 Disruption Scenarios and Continuity Strategies

#### Scenario 1: Cloud Infrastructure Outage

**Trigger**: Primary cloud provider experiences extended outage.

**Strategy**:
- Source code is replicated across GitHub (primary) and local developer clones (distributed backup).
- Documentation source is in the same repository and can be rebuilt from any clone.
- CI/CD can be temporarily executed from developer workstations for critical releases.
- Alternative cloud deployment can be provisioned within 4 hours using infrastructure-as-code templates.

#### Scenario 2: GitHub Outage or Account Compromise

**Trigger**: GitHub platform outage or unauthorized access to the Questo GitHub organization.

**Strategy**:
- Every developer maintains a full local clone of all repositories, providing distributed backup with RPO of the last push (typically < 1 hour).
- An encrypted offline backup of all repositories is maintained weekly on a separate storage service.
- In case of account compromise: revoke all access tokens, rotate all secrets, engage GitHub support, and restore from verified clean backup.

#### Scenario 3: LLM Provider Outage

**Trigger**: One or more LLM providers (OpenAI, Google Gemini, Anthropic) experience extended outage.

**Strategy**:
- The corteX SDK supports multiple LLM providers through its provider-agnostic architecture (`corteX/core/` LLM provider contracts).
- `ProviderHealth` monitors provider availability and can trigger automatic failover.
- Customer guidance documents recommend configuring at least two LLM providers for production deployments.
- For Questo's internal development and testing: maintain active API keys with at least two providers at all times.

#### Scenario 4: Key Personnel Unavailability

**Trigger**: The CTO or other critical team member becomes unavailable for an extended period.

**Strategy**:
- All critical procedures are documented in this policy framework and the internal runbook.
- The Security Lead serves as backup for the CTO for technical operations.
- The CEO serves as backup for organizational decisions.
- No single person holds exclusive knowledge required for critical operations. Cross-training is conducted annually.
- PyPI publishing credentials and cloud admin access have designated backup holders.

#### Scenario 5: Office or Regional Disruption

**Trigger**: Physical office inaccessible or regional event (natural disaster, conflict) affecting operations.

**Strategy**:
- Questo operates as a remote-capable organization. All development work can be conducted remotely.
- All systems are cloud-hosted and accessible from any location with internet access.
- Communication shifts to backup channels (secondary messaging platform, mobile phones) if primary channels are affected.

#### Scenario 6: PyPI Distribution Disruption

**Trigger**: PyPI platform outage or corteX package removal.

**Strategy**:
- Built SDK packages are retained in GitHub Releases as backup distribution points.
- Customers can install from direct download or private package repositories.
- The corteX documentation includes instructions for installation from alternative sources.
- A mirrored package can be published to a secondary Python package index if PyPI disruption is prolonged.

### 5.4 Communication During Disruption

#### 5.4.1 Internal Communication

1. The CTO activates the BCP and notifies all personnel via the primary communication channel.
2. If the primary channel is unavailable, the backup communication chain is activated (mobile phone tree, personal email).
3. Status updates are provided every 2 hours during an active disruption.
4. The CEO communicates with external stakeholders (investors, partners) as needed.

#### 5.4.2 Customer Communication

1. Customers are notified of disruptions affecting SDK distribution, documentation, or support within 4 hours.
2. Notifications are sent via the status page, email to registered contacts, and the SDK documentation site (if available).
3. Notifications include: nature of the disruption, affected services, estimated recovery time, and customer actions (if any).
4. Resolution notifications are sent when normal operations resume.

### 5.5 Plan Activation Criteria

This plan is activated when any of the following conditions are met:

- A critical function (Section 5.1.1) is disrupted or expected to be disrupted beyond its RTO.
- An essential function (Section 5.1.2) is disrupted beyond its MTPD.
- A security incident (POL-004) escalates to a level requiring operational changes.
- A regional event makes the primary work location unusable.
- The CEO or CTO determines that activation is warranted based on evolving circumstances.

### 5.6 Recovery Prioritization

Recovery follows this priority order:

1. **Priority 1**: Internal communication and team coordination
2. **Priority 2**: Source code access and integrity verification
3. **Priority 3**: CI/CD pipeline and testing capability
4. **Priority 4**: Customer communication and support channels
5. **Priority 5**: Documentation site and SDK distribution
6. **Priority 6**: License management infrastructure
7. **Priority 7**: Supporting business functions

## 6. Procedures

### 6.1 BCP Activation Procedure

1. The CTO (or CEO, if the CTO is unavailable) declares BCP activation.
2. All personnel are notified via the communication chain (Section 5.4.1).
3. The CTO convenes the Business Continuity Team for an initial assessment call within 30 minutes.
4. The team assesses: which functions are affected, the estimated duration, and the recovery strategy to execute.
5. Recovery procedures are initiated per the prioritization in Section 5.6.
6. Status tracking begins with updates every 2 hours.

### 6.2 Recovery Execution Procedure

1. For each affected function, execute the corresponding recovery procedure from the Disaster Recovery Plan (POL-007).
2. Document each recovery action with timestamp, personnel involved, and outcome.
3. Verify the integrity and security of each restored system before resuming operations.
4. Confirm that security controls (MFA, access controls, encryption) are fully operational in the recovery environment.
5. Notify the CTO when each function is restored.

### 6.3 Return to Normal Operations

1. The CTO confirms that all critical and essential functions are operational and verified.
2. Enhanced monitoring is activated for 7 days following return to normal operations.
3. A post-disruption review is conducted within 10 business days.
4. Lessons learned are documented and incorporated into plan updates.
5. The CTO formally declares the end of the BCP activation.

### 6.4 Annual BCP Testing

1. The BCP is tested at least annually through one of the following methods:
   - **Tabletop exercise**: The team walks through a disruption scenario and response actions.
   - **Functional test**: A specific recovery procedure is executed in a non-production environment.
   - **Full simulation**: A comprehensive test simulating an actual disruption (conducted at least every 2 years).
2. Test results are documented, including: scenario tested, participants, timeline, gaps identified, and improvement actions.
3. The CTO reviews test results and approves plan updates within 30 days of the test.
4. Testing is coordinated with the incident response tabletop exercise (POL-004 Section 5.7) when possible.

## 7. Exceptions

1. The annual testing requirement may be deferred by up to 90 days with CEO approval if operational constraints prevent timely execution.
2. RTO/RPO targets are objectives, not guarantees. Force majeure events may result in longer recovery times.
3. Exceptions to communication timeframes during a disruption may be granted by the CEO when circumstances prevent timely notification.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- All personnel are required to familiarize themselves with this plan and participate in annual testing.
- Failure to maintain documented recovery procedures for assigned systems is a policy violation.
- The CTO is responsible for ensuring this plan is current, tested, and adequate.
- Post-disruption reviews that identify plan deficiencies must result in plan updates within 30 days.
- Non-participation in required BCP testing without approved justification is a policy violation.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-004 | Incident Response Plan |
| POL-005 | Risk Assessment and Mitigation Policy |
| POL-007 | Disaster Recovery Plan |
| POL-009 | Vendor Management Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial plan creation |
