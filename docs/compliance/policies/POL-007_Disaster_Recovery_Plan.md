# Disaster Recovery Plan

**Document ID**: POL-007
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Disaster Recovery Plan (DRP) defines the technical procedures for recovering Questo Ltd's information systems and the corteX AI Agent SDK infrastructure following a disruptive event. It complements the Business Continuity Plan (POL-006) by providing the specific technical recovery procedures, backup strategies, and system restoration steps necessary to meet the defined Recovery Time Objectives (RTOs) and Recovery Point Objectives (RPOs).

This plan addresses the unique aspects of the corteX platform: a distributed SDK architecture, brain-inspired engine state, multi-provider LLM configuration, and on-premises-first deployments that operate independently of Questo infrastructure.

**SOC 2 Mapping**: CC7.5 (Recovery), CC9.1 (Risk Mitigation), A1.2 (Recovery Mechanisms)

## 2. Scope

This plan covers disaster recovery for:

- **Source Code Infrastructure**: GitHub repositories, branch protection, CI/CD pipeline configurations, and GitHub Actions secrets.
- **Build and Distribution**: CI/CD pipeline, PyPI publishing pipeline, package signing, and SDK artifact storage.
- **Documentation Infrastructure**: MkDocs Material site, domain configuration, and hosting platform.
- **License Infrastructure**: Ed25519 signing keys, license generation tools, and license validation components.
- **Cloud Infrastructure**: Virtual machines, databases, networking, DNS, and monitoring services.
- **Security Infrastructure**: KeyVault encryption keys, audit log storage, and compliance configurations.
- **Communication Infrastructure**: Email, messaging platforms, and customer notification systems.

This plan does not cover recovery of customer-side corteX deployments. Customer recovery guidance is provided in the SDK documentation.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **DRP** | Disaster Recovery Plan; procedures for restoring IT systems after a disaster |
| **RTO** | Recovery Time Objective; maximum time to restore a system |
| **RPO** | Recovery Point Objective; maximum acceptable data loss |
| **Hot Standby** | A fully operational backup system ready to take over immediately |
| **Warm Standby** | A backup system requiring minimal configuration before activation |
| **Cold Standby** | A backup system requiring significant setup time before activation |
| **Backup** | A copy of data or system configuration stored separately for recovery purposes |
| **Restore** | The process of recovering data or systems from backups |
| **Failover** | Switching operations from a failed system to a backup system |
| **Failback** | Returning operations from the backup system to the restored primary system |
| **Infrastructure as Code (IaC)** | System configurations stored as version-controlled code |

## 4. Roles and Responsibilities

### 4.1 CTO (Recovery Coordinator)

- Declares a disaster and activates this plan
- Directs the overall recovery effort
- Prioritizes recovery activities based on business impact
- Approves return to normal operations after recovery verification
- Ensures recovery procedures are tested annually

### 4.2 Security Lead (Security Recovery Lead)

- Ensures that recovered systems maintain security controls
- Validates the integrity of restored data and configurations
- Manages the recovery of security infrastructure (encryption keys, audit logs)
- Verifies that access controls are correctly restored

### 4.3 Senior Engineer (Technical Recovery Lead)

- Executes technical recovery procedures for assigned systems
- Verifies system functionality after restoration
- Documents recovery actions and timelines
- Identifies and resolves technical issues during recovery

### 4.4 All Engineers

- Execute recovery procedures as assigned
- Report recovery progress and issues to the CTO
- Validate recovered systems within their area of responsibility

## 5. Policy Statements

### 5.1 Backup Strategy

#### 5.1.1 Source Code Backups

| Asset | Primary | Backup Method | Frequency | RPO | Storage |
|-------|---------|--------------|-----------|-----|---------|
| corteX SDK repository | GitHub | Distributed Git clones on all developer machines | Continuous (every push) | 0 (multiple live copies) | Local developer machines |
| corteX SDK repository | GitHub | Encrypted full clone to secondary storage | Weekly | 1 week | Encrypted cloud storage (separate provider) |
| GitHub Actions workflows | GitHub | Included in repository | Continuous | 0 | Same as repository |
| GitHub Actions secrets | GitHub | Encrypted backup in password manager | On change | 0 | Password manager vault |

**Recovery Note**: Because Git is distributed, every developer clone is a full backup of the repository, including all history. The RPO for source code is effectively zero as long as at least one developer has pushed their changes.

#### 5.1.2 Infrastructure Configuration Backups

| Asset | Backup Method | Frequency | RPO | Storage |
|-------|--------------|-----------|-----|---------|
| Cloud infrastructure (IaC) | Version-controlled in repository | Continuous | 0 | Same as source code |
| DNS configuration | Documented in IaC + manual export | Monthly | 30 days | Repository + encrypted storage |
| CI/CD pipeline configuration | Version-controlled in repository | Continuous | 0 | Same as source code |
| MkDocs configuration | Version-controlled in repository | Continuous | 0 | Same as source code |

#### 5.1.3 Security Asset Backups

| Asset | Backup Method | Frequency | RPO | Storage |
|-------|--------------|-----------|-----|---------|
| Ed25519 license signing keys | Encrypted backup with split custody | On generation | 0 | Two separate encrypted locations, each held by different personnel |
| PyPI API tokens | Stored in GitHub Actions secrets + password manager | On rotation | 0 | GitHub Secrets + password manager vault |
| SSL/TLS certificates | Automated renewal (Let's Encrypt) + backup | On renewal | 0 | Automated provisioning, backup in password manager |
| Audit logs | Replicated to secondary storage | Daily | 24 hours | Encrypted secondary cloud storage |

#### 5.1.4 Backup Verification

1. Backup integrity is verified monthly by restoring a random sample of backed-up assets.
2. Source code backups are verified by cloning from the backup and running the test suite.
3. Security asset backups are verified by confirming decryption succeeds (without using the keys operationally).
4. Verification results are documented and reviewed by the CTO.

### 5.2 System Recovery Procedures

#### 5.2.1 Source Code Recovery (RTO: 1 Hour)

**Scenario**: GitHub is unavailable or the repository is compromised.

**Procedure**:
1. Identify the most recent verified local clone across all developer machines.
2. Verify clone integrity by checking the latest known-good commit hash against the team's records.
3. If GitHub is down temporarily: work from local clones, push when service resumes.
4. If the repository is compromised:
   a. Revoke all GitHub access tokens and rotate all repository secrets.
   b. Create a new repository (or restore the existing one from GitHub support).
   c. Push the verified clean clone to the restored repository.
   d. Reconfigure branch protection rules per POL-003.
   e. Reconfigure GitHub Actions secrets from the password manager backup.
5. Notify all team members of the new repository state and any required actions.

#### 5.2.2 CI/CD Pipeline Recovery (RTO: 4 Hours)

**Scenario**: CI/CD pipeline is non-functional.

**Procedure**:
1. Verify GitHub Actions service status. If GitHub Actions is down, wait or use alternative.
2. If the pipeline configuration is corrupted:
   a. Restore workflow files from the repository backup.
   b. Re-enter GitHub Actions secrets from the password manager backup.
   c. Run a validation build to confirm pipeline functionality.
3. If an alternative CI/CD platform is needed:
   a. Deploy the pipeline configuration to an alternative runner (self-hosted GitHub Actions runner or alternative CI platform).
   b. Configure secrets and environment variables.
   c. Validate by running the full test suite (8,082+ tests).
4. For critical emergency releases during pipeline outage:
   a. Build and test locally on a verified development machine.
   b. Require CTO sign-off for local builds.
   c. Document the manual process for post-incident review.

#### 5.2.3 Documentation Site Recovery (RTO: 8 Hours)

**Scenario**: docs.cortex-ai.com is unavailable.

**Procedure**:
1. Check hosting platform status. If temporary outage, monitor for resolution.
2. If hosting platform is compromised or unavailable:
   a. Build the documentation locally using `mkdocs build` from the repository.
   b. Deploy to an alternative hosting platform.
   c. Update DNS records to point to the new hosting.
   d. Verify site functionality and content integrity.
3. As a temporary measure, serve documentation from GitHub Pages while primary hosting is restored.
4. DNS changes may require up to 24 hours for full propagation. Use low TTL values in normal operations (300 seconds recommended).

#### 5.2.4 License Infrastructure Recovery (RTO: 4 Hours)

**Scenario**: License generation or validation infrastructure is unavailable.

**Procedure**:
1. Existing customer licenses continue to work offline (Ed25519 verification is performed locally by the SDK, no Questo infrastructure dependency).
2. For new license generation:
   a. Retrieve the Ed25519 signing key from the encrypted split-custody backup.
   b. Reconstruct the signing key on a secure workstation.
   c. Generate required licenses using the license generation tool.
   d. Verify generated licenses against the validation logic in the SDK.
3. Restore the license management system from backup when infrastructure is available.

#### 5.2.5 Cloud Infrastructure Recovery (RTO: 4 Hours)

**Scenario**: Primary cloud infrastructure is unavailable.

**Procedure**:
1. Assess the scope of the cloud outage (regional, service-specific, or account-level).
2. For regional outage:
   a. Deploy to an alternative region using IaC templates.
   b. Update DNS and load balancer configurations.
   c. Verify all services are operational in the new region.
3. For account compromise:
   a. Contact cloud provider support immediately.
   b. Rotate all credentials and API keys.
   c. Provision a new environment from IaC templates if account recovery is delayed.
   d. Restore data from backups.
4. Verify security controls are active in the recovered environment before resuming operations.

#### 5.2.6 Communication Infrastructure Recovery (RTO: 2 Hours)

**Scenario**: Primary communication platforms are unavailable.

**Procedure**:
1. Activate the backup communication channel (designated in the contact list).
2. If email is unavailable: use messaging platform. If messaging is unavailable: use email.
3. If both are unavailable: activate the phone tree using the emergency contact list.
4. Provision alternative communication tools if the outage is expected to exceed 24 hours.
5. Update customer communication channels to use alternative platforms.

### 5.3 corteX Customer Recovery Guidance

While customer deployments are not covered by this plan, Questo provides the following recovery guidance in the SDK documentation:

1. **Memory Persistence Recovery**: The `corteX/memory/persistence.py` module supports durable memory storage. Customers should configure persistent storage backends with their own backup procedures.
2. **LLM Provider Failover**: Configure multiple LLM providers in `TenantConfig`. The `ProviderHealth` module (`corteX/engine/provider_health.py`) monitors provider availability.
3. **Agent State Recovery**: The `corteX/engine/recovery.py` module provides agent state checkpointing and recovery capabilities for long-running agent tasks.
4. **Weight and Learning Recovery**: Brain-inspired weights (`corteX/engine/weights.py`) and plasticity state (`corteX/engine/plasticity.py`) can be exported and restored through the SDK's serialization interfaces.

### 5.4 Data Integrity Verification

After any recovery operation, the following integrity checks are performed:

1. **Source Code**: Verify Git commit hashes match known-good state. Run the full test suite (8,082+ tests) to confirm code integrity.
2. **Security Assets**: Verify Ed25519 keys can produce valid signatures. Verify Fernet encryption keys (AES-128-CBC with PBKDF2-derived 256-bit master key) can decrypt test data.
3. **Configuration**: Verify all environment variables and secrets are correctly set. Confirm MFA enforcement is active on all systems.
4. **Access Controls**: Verify RBAC configurations match the access control matrix (POL-002). Confirm branch protection rules are enforced.
5. **Audit Logs**: Verify audit log continuity. Document any gaps from the disruption period.

## 6. Procedures

### 6.1 Disaster Declaration Procedure

1. The individual detecting the disaster notifies the CTO immediately.
2. The CTO assesses the situation against the BCP activation criteria (POL-006 Section 5.5).
3. If the criteria are met, the CTO declares a disaster and activates this plan.
4. The CTO designates recovery leads and assigns systems per Section 4.
5. Recovery begins immediately per the prioritization in POL-006 Section 5.6.

### 6.2 Recovery Progress Tracking

1. The CTO maintains a recovery status board tracking: system, status, assigned lead, start time, target completion, and actual completion.
2. Status updates are communicated to the team every 2 hours.
3. Blockers are escalated immediately to the CTO for resolution.
4. Each recovered system is marked as restored only after the integrity verification in Section 5.4 passes.

### 6.3 Failback Procedure

1. When the primary infrastructure is restored, the CTO plans the failback.
2. The failback is executed during a low-activity period when possible.
3. Data synchronization is performed from the recovery environment to the restored primary.
4. Systems are switched back to the primary one at a time, with verification after each switch.
5. The recovery environment is maintained for 7 days after failback as a precaution.
6. The recovery environment is then decommissioned following the standard access revocation procedures.

### 6.4 Annual DR Testing Procedure

1. At least one DR recovery procedure is tested annually.
2. Testing rotates through critical systems on a cycle so that every system is tested at least once every 3 years.
3. Test scenarios include: source code recovery from backup, CI/CD pipeline reconstruction, documentation site redeployment, and license key recovery.
4. Test results document: scenario, start time, completion time, RTO met (yes/no), RPO met (yes/no), issues encountered, and corrective actions.
5. The CTO reviews results and updates recovery procedures as needed within 30 days.
6. Testing is coordinated with the BCP testing schedule (POL-006 Section 6.4).

## 7. Exceptions

1. Recovery procedures may deviate from documented steps when the specific disaster scenario is not anticipated. The CTO authorizes deviations and documents them for post-incident review.
2. RTO targets are objectives. The CTO may re-prioritize recovery efforts based on the specific disaster circumstances.
3. Annual DR testing may be deferred up to 90 days with CEO approval per POL-006 Section 7.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- The CTO is accountable for maintaining current and tested recovery procedures.
- Engineering team members assigned as recovery leads must maintain familiarity with their assigned recovery procedures.
- Failure to maintain backup schedules or verify backup integrity is a policy violation.
- Recovery actions are documented and reviewed post-incident. Deviations from established procedures without authorization are policy violations.
- Annual DR testing is mandatory. Non-completion is a control deficiency for SOC 2 purposes.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-003 | Change Management Policy |
| POL-004 | Incident Response Plan |
| POL-005 | Risk Assessment and Mitigation Policy |
| POL-006 | Business Continuity Plan |
| POL-011 | Encryption Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial plan creation |
