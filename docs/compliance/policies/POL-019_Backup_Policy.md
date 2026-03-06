# Backup and Recovery Policy

**Document ID**: POL-019
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Backup and Recovery Policy defines the requirements for backing up, protecting, and restoring Questo Ltd information assets, including source code, business data, operational configurations, and compliance documentation. It ensures that data can be recovered following accidental deletion, corruption, hardware failure, security incidents, or disasters.

As the corteX AI Agent SDK is a customer-deployed, on-premises product, this policy also addresses the distinction between Questo Ltd's backup responsibilities for its own systems and the customer's backup responsibilities for their on-premises corteX deployments.

**SOC 2 Mapping**: CC7.5 (Recovery), A1.2-A1.3 (Availability), CC9.1 (Risk Mitigation)

## 2. Scope

This policy applies to:

- **Source Code**: The corteX SDK repository and all associated repositories hosted on GitHub.
- **Business Data**: Financial records, contracts, employee data, customer records, and communication archives.
- **Compliance Documentation**: SOC 2 policies, evidence artifacts, risk registers, and audit records.
- **Infrastructure Configuration**: CI/CD pipeline configurations, cloud infrastructure-as-code, DNS settings, and deployment configurations.
- **Operational Data**: Logging data, monitoring configurations, alert rules, and incident records.
- **Documentation**: MkDocs Material documentation source (97 pages), published documentation site, and API references.

This policy does **not** apply to customer data stored in customer on-premises corteX deployments, which is the customer's responsibility. However, Section 5.7 provides guidance that Questo Ltd includes in the SDK deployment documentation.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **RPO** | Recovery Point Objective; the maximum acceptable amount of data loss measured in time |
| **RTO** | Recovery Time Objective; the maximum acceptable time to restore service after a disruption |
| **Full Backup** | A complete copy of all data in the backup scope |
| **Incremental Backup** | A copy of only the data that has changed since the last backup |
| **Backup Verification** | The process of confirming that a backup is complete, intact, and restorable |
| **Restore Test** | An actual restoration of data from backup to verify the process works end-to-end |
| **Off-Site Backup** | A backup stored in a geographically separate location from the primary data |
| **Encryption at Rest** | Data encrypted when stored on any medium, per POL-011 |
| **Git Mirror** | A complete clone of a Git repository maintained as a backup on a separate platform |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and ensures its implementation
- Approves RPO/RTO targets and backup strategies
- Reviews backup and restore test results quarterly
- Authorizes data restoration from backups

### 4.2 Security Lead

- Executes backup configuration and monitoring for operational systems
- Conducts quarterly backup verification and restore tests
- Maintains backup inventory and documentation
- Responds to backup failures and data recovery requests
- Ensures backup encryption meets POL-011 requirements

### 4.3 All Personnel

- Follow data storage practices that keep business data in backed-up systems (not on local devices only)
- Report data loss or suspected data corruption immediately to the Security Lead
- Do not store critical business data exclusively on local workstations

## 5. Policy Statements

### 5.1 Backup Requirements by Asset Category

| Asset Category | Backup Method | Frequency | RPO | RTO | Retention | Off-Site |
|---------------|--------------|-----------|-----|-----|-----------|---------|
| **Source Code (GitHub)** | Git distributed + mirror | Continuous (Git) + daily mirror | Near-zero (Git) | 4 hours | Indefinite (Git history) | Yes (mirror) |
| **Business Data** | Cloud provider backup | Daily incremental, weekly full | 24 hours | 8 hours | 12 months + 24 months archive | Yes |
| **Compliance Documentation** | Cloud storage backup + Git | Daily | 24 hours | 8 hours | 7 years | Yes |
| **Infrastructure Config** | Git (IaC) + cloud snapshots | Continuous (Git) + daily snapshots | Near-zero (IaC) | 4 hours | 12 months | Yes |
| **CI/CD Configuration** | Git (config as code) | Continuous (Git) | Near-zero | 2 hours | Indefinite (Git history) | Yes |
| **Documentation Source** | Git (MkDocs source) | Continuous (Git) | Near-zero | 2 hours | Indefinite (Git history) | Yes |
| **Operational Logs** | Cloud provider backup | Daily | 24 hours | 24 hours | Per POL-017 retention schedule | Yes |
| **Communication Archives** | Platform-native backup | Per platform schedule | 24 hours | 24 hours | 3 years | Platform-managed |

### 5.2 Source Code Backup Strategy

Source code is the most critical Questo Ltd asset. The following multi-layer backup strategy applies:

1. **Git Distributed Nature**: Every developer's local clone of the repository is a complete backup of the entire history. This provides inherent redundancy across multiple machines and geographic locations.
2. **GitHub Primary**: The `QuestoM/cortex-sdk` private repository on GitHub serves as the primary authoritative source. GitHub provides infrastructure redundancy, including geographic replication.
3. **Daily Git Mirror**: A complete mirror of all repositories is maintained on a separate platform (independent cloud storage or alternative Git hosting). The mirror is updated daily via automated script.
4. **Mirror Verification**: The daily mirror process verifies that the mirror is a complete and accurate reflection of the primary repository, including all branches and tags.
5. **Branch Protection as Backup**: Branch protection on `main` (no force pushes, no deletion) prevents accidental loss of the primary branch history.

### 5.3 Business Data Backup Strategy

1. **Cloud-Native Backup**: Business data stored in cloud platforms (Google Workspace, cloud storage) uses the platform's native backup capabilities.
2. **Daily Incremental Backup**: Changes since the last backup are captured daily.
3. **Weekly Full Backup**: A complete backup is created weekly to provide a reliable restore point.
4. **Geographic Separation**: Backups are stored in a geographic region separate from the primary data.
5. **Encryption**: All backups are encrypted at rest using AES-256 per POL-011. Backup encryption keys are stored separately from the backup data.

### 5.4 Backup Encryption and Access Control

1. **Encryption at Rest**: All backups are encrypted at rest per POL-011. The minimum encryption standard is AES-256.
2. **Encryption in Transit**: Backup data in transit uses TLS 1.2+ encryption.
3. **Key Management**: Backup encryption keys are managed separately from the backed-up data. Key storage follows the same protection level as the data it protects.
4. **Access Control**: Backup data access is restricted to the CTO and Security Lead. Access is logged per POL-017.
5. **Restore Authorization**: Data restoration from backups requires CTO authorization, except in emergency situations where the Security Lead may authorize with CTO notification within 4 hours.

### 5.5 Backup Verification and Restore Testing

1. **Daily Automated Verification**: Each backup job includes automated integrity verification (checksum validation) upon completion. Backup failures trigger an immediate alert to the Security Lead.
2. **Quarterly Restore Tests**: The Security Lead conducts a restore test each quarter, rotating through different asset categories. The test verifies end-to-end restoration, including: data completeness, data integrity, time to restore (compared against RTO), and accessibility of restored data.
3. **Annual Full Recovery Test**: Once per year, a comprehensive recovery test is conducted that simulates restoring all critical systems from backups. This test is coordinated with the disaster recovery plan (POL-007).
4. **Test Documentation**: All backup verification and restore test results are documented with: date, scope, result (pass/fail), time to restore, issues encountered, and remediation actions.

### 5.6 Backup Monitoring and Alerting

1. All backup jobs are monitored for successful completion.
2. Backup failures trigger an alert to the Security Lead within 1 hour of the scheduled backup time.
3. Two consecutive backup failures for the same asset category are escalated to the CTO.
4. Backup job status (last successful backup date, next scheduled backup, storage utilization) is reviewed during the monthly operational review.

### 5.7 Customer On-Premises Backup Guidance

Questo Ltd does not manage backups for customer on-premises corteX deployments. The following guidance is provided in the SDK deployment documentation:

1. **Audit Log Backup**: Customers should back up corteX audit logs (`AuditFileStore` data) to preserve the tamper-evident hash chain for compliance. Audit logs contain the SHA-256 chain that provides legal-grade evidence of AI agent operations.
2. **Configuration Backup**: Customers should back up `TenantConfig`, `SafetyPolicy`, and `ToolPolicy` configurations to enable rapid recovery.
3. **Memory Persistence**: If customers use persistent memory features (long-term memory, episodic memory), the backing store should be included in the customer's backup strategy.
4. **KeyVault Backup**: Customers should back up the `KeyVault` data store with appropriate encryption. Loss of KeyVault data results in loss of stored API keys, requiring re-entry.
5. **License Files**: Customers should back up their corteX license files (Ed25519-signed), as reissuance requires contacting Questo Ltd.

### 5.8 Backup Retention and Disposal

1. Backups are retained per the schedule defined in Section 5.1.
2. After the retention period expires, backups are securely deleted using methods appropriate to the storage medium (cryptographic erasure for encrypted backups, secure overwrite for unencrypted media).
3. Legal hold requests override retention schedules. Backups subject to legal hold are preserved until the hold is released.
4. Backup disposal is documented with: asset category, date range covered, disposal date, disposal method, and executor.

## 6. Procedures

### 6.1 Daily Backup Monitoring Procedure

1. The Security Lead reviews backup job completion status each business morning.
2. Failed backups are investigated. Common causes (storage capacity, network issues, credential expiry) are resolved and the backup is re-run.
3. Persistent failures (same job failing 2+ consecutive days) are escalated to the CTO.
4. Backup status is documented in the operational log.

### 6.2 Quarterly Restore Test Procedure

1. The Security Lead selects the asset category for the quarterly restore test, rotating through: source code (Q1), business data (Q2), compliance documentation (Q3), infrastructure configuration (Q4).
2. The test is conducted in an isolated environment that does not affect production systems.
3. The Security Lead restores the selected data from the most recent backup.
4. The restored data is verified for: completeness (all expected files/records present), integrity (checksums match or data is readable and correct), and timeliness (restore completes within the defined RTO).
5. Results are documented in the Backup Restore Test Report and presented to the CTO.
6. Issues identified during testing are remediated within 10 business days.

### 6.3 Data Recovery Procedure

1. The requestor contacts the Security Lead with: the data to be recovered, the reason for recovery, and the approximate date of the data state needed.
2. The Security Lead obtains CTO authorization for the restoration (except in emergencies per Section 5.4).
3. The Security Lead identifies the appropriate backup and initiates restoration to a staging area.
4. The restored data is verified for completeness and integrity before delivery to the requestor.
5. The recovery event is documented with: requestor, authorization, data recovered, backup source, recovery time, and verification result.

### 6.4 Daily Git Mirror Procedure

1. An automated script runs daily to mirror all GitHub repositories to the off-site backup location.
2. The script performs: `git clone --mirror` (initial) or `git remote update` (subsequent) for each repository.
3. The script verifies that the mirror's `HEAD` and branch references match the primary repository.
4. Mirror status (success/failure, last sync time) is logged and monitored alongside other backup jobs.
5. Mirror failures are addressed per the backup monitoring procedure (Section 6.1).

## 7. Exceptions

1. Exceptions to this policy require written approval from the CTO.
2. Exception requests must include: the specific policy provision, business justification, compensating controls, and requested duration (maximum 6 months).
3. No exceptions are permitted for: source code backup (GitHub + mirror), backup encryption at rest, or the quarterly restore test requirement.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Failure to maintain backup schedules is a process deficiency that will be identified during quarterly reviews and addressed with corrective action plans.
- Storing critical business data exclusively on local workstations (without backup) is a policy violation. Personnel are required to use backed-up systems for all business data.
- Unauthorized access to backup data is a serious violation investigated per POL-004.
- The CTO reviews backup compliance quarterly as part of the operational security review.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-004 | Incident Response Plan |
| POL-006 | Business Continuity Plan |
| POL-007 | Disaster Recovery Plan |
| POL-008 | Data Classification Policy |
| POL-011 | Encryption Policy |
| POL-017 | Logging and Monitoring Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
