# Change Management Policy

**Document ID**: POL-003
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO
**Classification**: INTERNAL

---

## 1. Purpose

This Change Management Policy ensures that all changes to the corteX AI Agent SDK codebase, infrastructure, configuration, and documentation are properly authorized, tested, reviewed, and documented. It establishes the GitHub Pull Request workflow as the formal change management process and defines the CI/CD pipeline requirements that gate all changes to the production codebase.

**SOC 2 Mapping**: CC8.1 (Change Management)

## 2. Scope

This policy applies to all changes affecting:

- **Source Code**: All files in the corteX SDK repository, including application code, tests, configuration files, and build scripts.
- **Infrastructure**: Cloud infrastructure, CI/CD pipeline configurations, deployment configurations, and server components.
- **Documentation**: SDK documentation (MkDocs site), API reference documentation, and compliance documentation.
- **Dependencies**: Third-party library additions, updates, or removals.
- **Configuration**: Environment variables, feature flags, safety policies, and tenant configuration defaults.
- **Distribution**: Releases published to PyPI, documentation site deployments, and server component deployments.

This policy does not apply to changes in personal development environments that have not been proposed for merging into shared branches.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Change** | Any modification to source code, infrastructure, configuration, or documentation that affects shared environments or the distributed SDK |
| **Pull Request (PR)** | A GitHub-based change proposal that includes the changeset, description, test results, and review history |
| **Standard Change** | A pre-approved, low-risk change that follows an established procedure (e.g., dependency patch update) |
| **Minor Change** | A change of moderate complexity requiring standard review and approval |
| **Significant Change** | A change with broad impact, breaking changes, or security implications requiring enhanced review |
| **Emergency Change** | A change required to address an active security incident or critical production failure |
| **CI/CD Pipeline** | The automated Continuous Integration and Continuous Delivery system that builds, tests, and validates changes |
| **CAB** | Change Advisory Board; a review body for significant changes, consisting of the CTO, Security Lead, and senior engineers |
| **Semantic Versioning (SemVer)** | Version numbering in the format MAJOR.MINOR.PATCH per semver.org |
| **Rollback** | The process of reverting a change to restore the previous state |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and the overall change management process
- Serves as the final approver for significant changes and releases
- Chairs the Change Advisory Board
- Approves emergency changes and reviews them post-implementation

### 4.2 Security Lead

- Reviews all changes with security implications
- Verifies that security tests pass in the CI/CD pipeline
- Validates that changes do not introduce new vulnerabilities
- Participates in the Change Advisory Board

### 4.3 Change Author (Any Engineer)

- Creates the Pull Request with a complete description, including: what changed, why, impact analysis, and testing performed
- Ensures all CI/CD checks pass before requesting review
- Addresses reviewer feedback and updates the PR accordingly
- Does not merge their own Pull Request (segregation of duties)

### 4.4 Reviewer (Peer Engineer)

- Reviews the code for correctness, security, maintainability, and adherence to coding standards
- Verifies that the change description accurately reflects the changeset
- Approves or requests changes with documented rationale
- Ensures the change does not violate the principles in POL-001

### 4.5 Release Manager

- Coordinates SDK releases to PyPI
- Verifies that all release criteria are met before publishing
- Maintains the CHANGELOG and release notes
- The CTO serves as the Release Manager or delegates to a senior engineer

## 5. Policy Statements

### 5.1 Change Categories

All changes are categorized based on risk and impact:

| Category | Description | Approval Required | Examples |
|----------|-------------|-------------------|---------|
| **Standard** | Low-risk, pre-approved procedures | 1 peer reviewer | Dependency patch updates, typo fixes, documentation corrections, test additions |
| **Minor** | Moderate complexity, localized impact | 1 peer reviewer + CI pass | New features, bug fixes, refactoring within a module, non-breaking API additions |
| **Significant** | Broad impact, breaking changes, or security implications | 2 reviewers including CTO or Security Lead + CAB review | Breaking API changes, new security controls, architecture changes, new LLM provider integration |
| **Emergency** | Active incident or critical vulnerability | CTO verbal approval (documented within 24h) | Security hotfix, critical bug in production |

### 5.2 GitHub Pull Request Workflow

The GitHub Pull Request is the formal change request mechanism for all code and configuration changes.

#### 5.2.1 PR Requirements

Every Pull Request must include:

1. **Title**: A concise description of the change (imperative mood, e.g., "Add rate limiting to orchestrator").
2. **Description**: A detailed explanation including:
   - What changed and why
   - Impact analysis (which modules, tenants, or features are affected)
   - Testing performed beyond automated tests (if applicable)
   - Breaking changes (if any) and migration instructions
3. **Linked Issues**: Reference to relevant GitHub issues or tickets.
4. **Labels**: Category label (standard/minor/significant) and area labels (engine, memory, security, etc.).

#### 5.2.2 Branch Protection Rules

The `main` branch is protected with the following enforced rules:

- No direct pushes to `main`; all changes must come through Pull Requests
- At least one approved review required (two for significant changes)
- All CI/CD status checks must pass
- Force pushes are disabled
- Branch deletion is restricted
- Stale review dismissal is enabled (re-review required after new commits)

### 5.3 CI/CD Pipeline Requirements

Every Pull Request triggers the CI/CD pipeline, which must complete successfully before the PR can be merged. The pipeline includes:

| Stage | Description | Pass Criteria |
|-------|-------------|---------------|
| **Unit Tests** | Execute all unit tests | 8,082+ tests must pass with 0 failures |
| **Integration Tests** | Execute integration test suite | All integration tests pass (external service tests may be skipped with documented reason) |
| **Type Checking** | Static type analysis | No new type errors introduced |
| **Linting** | Code style and quality checks | No linting errors |
| **Secret Scanning** | Scan for hardcoded secrets | Zero secrets detected |
| **Dependency Audit** | Check for known vulnerabilities in dependencies | No HIGH or CRITICAL CVEs in new or updated dependencies |
| **Build Verification** | Verify the package builds correctly | Build succeeds without errors |

A change cannot be merged if any required CI/CD stage fails. There are no overrides for CI/CD failures except through the emergency change process (Section 5.7).

### 5.4 Code Review Standards

Reviewers evaluate changes against the following criteria:

1. **Correctness**: Does the change do what the description claims?
2. **Security**: Does the change introduce any security risks? Are inputs validated? Are secrets protected?
3. **Coding Standards**: Does the change adhere to the corteX coding standards (max 300 lines/file, type hints on public functions, async for I/O, logging over print)?
4. **Testing**: Are new features and bug fixes accompanied by tests? Do existing tests still pass?
5. **Documentation**: Are public API changes reflected in documentation?
6. **Performance**: Does the change introduce performance regressions?
7. **Tenant Isolation**: Does the change maintain multi-tenant data isolation guarantees?
8. **Backward Compatibility**: Are breaking changes documented and versioned appropriately?

### 5.5 Semantic Versioning and Release Management

The corteX SDK follows Semantic Versioning (semver.org):

- **MAJOR** (X.0.0): Breaking changes to the public API. Requires CAB review and customer notification.
- **MINOR** (0.X.0): New features added in a backward-compatible manner. Requires standard review.
- **PATCH** (0.0.X): Backward-compatible bug fixes. Requires standard review.

#### 5.5.1 Release Process

1. The Release Manager creates a release PR that updates the version number and CHANGELOG.
2. The CHANGELOG entry includes: all changes since the last release, categorized as Added/Changed/Fixed/Security/Deprecated/Removed.
3. The CTO reviews and approves the release PR.
4. Upon merge, the CI/CD pipeline builds and publishes the package to PyPI.
5. The Release Manager creates a GitHub Release with release notes.
6. Security-relevant changes are explicitly flagged in the release notes.
7. Breaking changes include migration guides.

#### 5.5.2 PyPI Publishing Controls

- Publishing to PyPI requires the CTO's approval (or explicit delegation).
- The publishing CI/CD job uses a project-scoped API token stored as a GitHub Actions secret.
- The published package is verified against the approved release artifacts.
- A post-publish verification confirms the correct version is available on PyPI.

### 5.6 Rollback Procedures

#### 5.6.1 Code Rollback

1. Create a revert PR that reverses the problematic change using `git revert`.
2. The revert PR follows the standard review process but with expedited review (target: 1 hour).
3. If the revert is complex (multiple interrelated commits), the CTO may approve a direct revert merge.

#### 5.6.2 PyPI Release Rollback

1. If a published release contains a critical defect, the Release Manager publishes a new PATCH version with the fix (PyPI does not allow overwriting published versions).
2. The defective version is yanked on PyPI (marked as not recommended for installation).
3. A security advisory is published if the defect has security implications.
4. Affected customers are notified per the communication procedures in POL-004.

#### 5.6.3 Infrastructure Rollback

1. Infrastructure changes deployed through IaC are rolled back by reverting to the previous IaC state and re-deploying.
2. The rollback is documented with the reason, timeline, and post-rollback verification results.

### 5.7 Emergency Change Procedures

Emergency changes are changes required to address an active security incident (POL-004) or a critical production failure that cannot wait for the standard change process.

1. **Authorization**: The CTO verbally authorizes the emergency change. If the CTO is unavailable, the Security Lead may authorize with CTO notification within 4 hours.
2. **Implementation**: The engineer implements the fix with a focus on containment and minimal change scope.
3. **Expedited Review**: The change is submitted as a PR with the `emergency` label. A single reviewer may approve the change.
4. **CI/CD**: All CI/CD checks must still pass. If a test failure is directly related to the emergency (e.g., the test itself is broken), the CTO may authorize a temporary test skip with a follow-up issue created.
5. **Documentation**: Within 24 hours of the emergency change, the author documents: the incident that triggered the change, the change description, the impact, and any follow-up actions required.
6. **Post-Implementation Review**: Emergency changes are reviewed by the CAB within 5 business days to verify correctness, assess whether the standard process should have been followed, and identify process improvements.

### 5.8 Change Advisory Board (CAB)

The CAB reviews all significant changes and consists of:

- CTO (Chair)
- Security Lead
- At least one Senior Engineer

The CAB convenes on an as-needed basis for significant changes and meets at least monthly to review change trends and process effectiveness. CAB decisions are documented in meeting minutes.

### 5.9 Change Documentation

All changes are documented through the following mechanisms:

- **Git History**: Every commit provides an immutable record of what changed, when, and by whom.
- **Pull Request Records**: GitHub retains the PR description, review comments, approval history, and CI/CD results.
- **CHANGELOG.md**: A human-readable record of all notable changes, maintained per the Keep a Changelog format.
- **Release Notes**: Each GitHub Release includes summary notes for the version.
- **Post-Implementation Records**: Emergency and significant changes include additional documentation as described in their respective sections.

## 6. Procedures

### 6.1 Standard Change Procedure

1. Engineer creates a feature branch from `main`.
2. Engineer implements the change, including appropriate tests.
3. Engineer pushes the branch and creates a Pull Request per Section 5.2.1.
4. CI/CD pipeline executes automatically.
5. Peer reviewer reviews the change per Section 5.4.
6. Upon approval and CI/CD pass, the reviewer or author merges the PR.
7. The feature branch is deleted after merge.

### 6.2 Significant Change Procedure

1. Engineer creates a design document or RFC describing the change, its impact, and migration plan.
2. The design document is reviewed by the CAB.
3. Upon CAB approval, the engineer implements the change following Section 6.1.
4. The PR requires two reviewers, including the CTO or Security Lead.
5. Post-merge, the engineer verifies the change in the target environment.
6. The CHANGELOG is updated to reflect the significant change.

### 6.3 Dependency Update Procedure

1. Engineer identifies the dependency update (new version, security patch, or new dependency).
2. For new dependencies: a justification is included in the PR description, including license compatibility, security posture, and maintenance status of the dependency.
3. For security patches: the PR references the CVE(s) being addressed.
4. The CI/CD pipeline verifies that the dependency update does not introduce regressions.
5. The dependency audit stage checks for known vulnerabilities in the updated dependency.

### 6.4 Release Procedure

1. Release Manager verifies all intended changes are merged to `main`.
2. Release Manager creates a release branch and updates the version number and CHANGELOG.
3. The release PR is reviewed and approved by the CTO.
4. Upon merge, the CI/CD pipeline builds, tests, and publishes to PyPI.
5. Release Manager creates a GitHub Release with release notes.
6. Release Manager performs post-publish verification on PyPI.
7. Documentation site is updated if necessary.

## 7. Exceptions

1. Exceptions to this policy are limited to emergency changes (Section 5.7) and require CTO authorization.
2. No exceptions are permitted for: branch protection rules on `main`, the prohibition on self-merging PRs, or the requirement for CI/CD pipeline execution.
3. If an exception is required outside the emergency change process, the requesting party submits a written request to the CTO with business justification, risk analysis, and compensating controls.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Changes that bypass the PR process (e.g., direct pushes to `main`) are technically prevented by branch protection rules. Any circumvention of these rules is considered a serious policy violation.
- Merging a PR with failing CI/CD checks (outside the documented emergency process) is a policy violation subject to disciplinary action.
- Self-approval of Pull Requests is prohibited and will be flagged during quarterly access reviews.
- Repeated violations of change management procedures result in escalating disciplinary action per POL-001 Section 8.
- The CTO reviews change management compliance monthly as part of operational metrics.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-004 | Incident Response Plan |
| POL-005 | Risk Assessment and Mitigation Policy |
| POL-011 | Encryption Policy |
| POL-017 | Logging and Monitoring Policy |
| POL-018 | Software Development Lifecycle Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
