# Software Development Lifecycle Policy

**Document ID**: POL-018
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Software Development Lifecycle (SDLC) Policy defines the secure development practices, quality gates, testing requirements, and release controls that govern the development of the corteX AI Agent SDK. It ensures that security, quality, and reliability are integrated into every phase of the development process, from design through deployment and maintenance.

The corteX SDK is an enterprise-grade AI agent engine with brain-inspired architecture, multi-tenant isolation, and on-premises deployment capabilities. The complexity and sensitivity of this platform demand a rigorous SDLC that protects customer trust, maintains processing integrity, and prevents the introduction of vulnerabilities.

**SOC 2 Mapping**: CC8.1 (Change Management), CC7.1-CC7.2 (System Operations), CC5.2 (Control Activities)

## 2. Scope

This policy applies to:

- **Code**: All source code in the corteX SDK repository (`QuestoM/cortex-sdk`), including application code, tests, configuration files, build scripts, documentation, and compliance documents.
- **Personnel**: All engineers, contractors, and contributors who develop, review, test, or release code for the corteX SDK.
- **Processes**: All development activities including design, implementation, code review, testing, CI/CD, release management, and post-release maintenance.
- **Dependencies**: All third-party libraries and frameworks used in the corteX SDK.
- **Documentation**: All technical documentation including API references, developer guides, and MkDocs Material documentation (97 pages).

## 3. Definitions

| Term | Definition |
|------|-----------|
| **SDLC** | Software Development Lifecycle; the end-to-end process for developing, maintaining, and retiring software |
| **Secure Coding Standards** | The coding rules for corteX (max 300 lines/file, type hints, async I/O, logging over print, no hardcoded secrets) |
| **Branch Protection** | GitHub-enforced rules on `main` preventing direct pushes and requiring reviews and CI passage |
| **CI/CD** | Continuous Integration/Delivery; the automated pipeline that builds, tests, and validates every change |
| **Secret Scanning** | Automated detection of accidentally committed credentials, API keys, or tokens |
| **Threat Modeling** | Identifying threats, attack vectors, and mitigations for a system or feature |
| **SemVer** | Semantic Versioning (MAJOR.MINOR.PATCH) per semver.org |
| **CAB** | Change Advisory Board; reviews significant changes per POL-003 |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and the overall SDLC process
- Approves architectural decisions and significant design changes
- Serves as Release Manager (or delegates to a senior engineer)
- Chairs the CAB for significant changes
- Reviews and approves security-sensitive code changes

### 4.2 Security Lead

- Reviews all security-relevant code changes (security modules, authentication, encryption, data handling)
- Conducts threat modeling for new features that affect the security boundary
- Manages dependency vulnerability scanning and coordinates remediation
- Validates that security tests cover identified threat vectors

### 4.3 Engineers (Developers)

- Follow secure coding standards defined in this policy
- Write unit tests for all new code and maintain test coverage
- Create complete Pull Request descriptions per POL-003
- Address code review feedback promptly and thoroughly
- Do not merge their own Pull Requests (segregation of duties)

### 4.4 Code Reviewers

- Review code for correctness, security, maintainability, and adherence to coding standards
- Verify that tests adequately cover the change
- Ensure documentation is updated for public API changes
- Approve or request changes with documented rationale

## 5. Policy Statements

### 5.1 SDLC Phases

The corteX SDK follows a structured development lifecycle with security integrated into every phase:

#### Phase 1: Requirements and Design

1. New features and significant changes begin with a design document or RFC that includes: functional requirements, security considerations, impact on multi-tenant isolation, impact on existing tests, and API design.
2. Features affecting the security boundary (security modules, LLM provider integration, data handling, tool execution) require threat modeling before implementation.
3. Design documents are reviewed by the CTO and, for security-relevant features, the Security Lead.
4. Significant changes require CAB approval per POL-003 before implementation begins.

#### Phase 2: Implementation

1. All code is written in accordance with the corteX Secure Coding Standards (Section 5.2).
2. Engineers work on feature branches created from `main`. Branch naming follows the convention: `feature/`, `fix/`, `security/`, `docs/`, or `refactor/` prefix.
3. Commits are atomic and descriptive. Each commit should represent a single logical change.
4. Engineers write tests concurrent with implementation, not as an afterthought.

#### Phase 3: Code Review

1. All changes are submitted as GitHub Pull Requests per POL-003.
2. PRs must include a complete description: what changed, why, impact analysis, and testing performed.
3. At least one peer reviewer must approve the PR. Significant changes require two reviewers including the CTO or Security Lead.
4. Self-approval is prohibited. The author of a PR cannot be its approver.
5. Review feedback is addressed before merge. Unresolved comments block the merge.

#### Phase 4: Automated Testing and Validation

1. The CI/CD pipeline executes automatically on every PR (see Section 5.3).
2. All 8,082+ unit tests must pass with zero failures.
3. Integration tests execute against supported configurations.
4. Static analysis (type checking, linting) must pass with no new errors.
5. Secret scanning and dependency audit must pass with no new findings.
6. A PR cannot be merged if any CI/CD gate fails.

#### Phase 5: Release

1. Releases follow Semantic Versioning per POL-003.
2. The Release Manager (CTO or delegate) coordinates the release process.
3. A release PR updates the version number and CHANGELOG.
4. The CI/CD pipeline builds and publishes the package to PyPI upon merge.
5. Post-publish verification confirms the correct version is available.

#### Phase 6: Post-Release Monitoring

1. The team monitors for issues reported by SDK users after release.
2. Security vulnerabilities in released versions are addressed per the severity-based SLA in Section 5.6.
3. Patch releases are issued for critical bugs and security fixes.

### 5.2 Secure Coding Standards

All corteX SDK code must adhere to the following standards:

#### 5.2.1 Structural Standards

| Standard | Requirement | Rationale |
|----------|------------|-----------|
| File length | Maximum 300 lines per file | Maintainability, reviewability |
| Type hints | Required on all public functions and methods | Code clarity, static analysis |
| Async I/O | Async by default for all I/O operations | Performance, concurrency |
| Logging | `logging` module only; `print` statements are prohibited | Structured observability |
| Secrets | Never hardcoded; environment variables or KeyVault only | Security (enforced by secret scanning) |
| Imports | Explicit imports; no wildcard imports (`from x import *`) | Code clarity, predictability |

#### 5.2.2 Security Coding Standards

| Standard | Requirement | Rationale |
|----------|------------|-----------|
| Input validation | All external inputs are validated before processing | Prompt injection prevention, data integrity |
| Output protection | All outputs are checked for leaked secrets via `KeyVault.detect_leak()` and for PII patterns via `SafetyPolicy.check_output()`, which blocks (not sanitizes) unsafe outputs | Prevent credential and PII exposure |
| Tenant isolation | All data structures are scoped by `tenant_id` | Multi-tenant confidentiality |
| Error handling | Exceptions must not expose internal state, stack traces, or secrets to external callers | Information disclosure prevention |
| Data classification | Data flowing through the SDK is classified by `DataClassifier` and handled per its level | Compliance with POL-008 |
| Dependency minimization | New dependencies require justification in the PR description | Attack surface reduction |

#### 5.2.3 AI-Specific Coding Standards

| Standard | Requirement | Rationale |
|----------|------------|-----------|
| Prompt injection defense | `SafetyPolicy` validates all inputs against known injection patterns | AI safety |
| Output safety | `SafetyPolicy.check_output()` blocks outputs containing PII patterns or unsafe content (returns rejection, not sanitization). `PIITokenizer` restores tenant-owned PII tokens in responses by design. | AI safety |
| Goal verification | `GoalTracker` verifies every agent step against the original goal | Processing integrity |
| Loop prevention | `LoopDetector` uses state hashing and drift detection to prevent infinite loops | Resource protection |
| Tool execution control | `ToolPolicy` enforces allowlisting; default posture is deny-all | Security by default |

### 5.3 CI/CD Pipeline Requirements

The CI/CD pipeline is the automated quality gate for all code changes. Every Pull Request must pass all stages before merge is permitted.

#### 5.3.1 Pipeline Stages

| Stage | Tool | Pass Criteria | Blocking |
|-------|------|---------------|----------|
| **Unit Tests** | pytest | 8,082+ tests pass, 0 failures | Yes |
| **Integration Tests** | pytest (integration markers) | All integration tests pass | Yes |
| **Type Checking** | mypy or equivalent | No new type errors | Yes |
| **Linting** | ruff or equivalent | No linting errors | Yes |
| **Secret Scanning** | GitHub secret scanning + pre-commit hooks | Zero secrets detected | Yes |
| **Dependency Audit** | pip-audit or equivalent | No HIGH/CRITICAL CVEs in new/updated dependencies | Yes |
| **Build Verification** | pip install / wheel build | Package builds without errors | Yes |
| **Documentation Build** | MkDocs | Documentation builds without errors | Yes |

#### 5.3.2 Pipeline Integrity

- CI/CD pipeline configuration files are subject to the same change management process as application code per POL-003.
- Pipeline secrets (API tokens, signing keys) are stored in GitHub Actions secrets, scoped to the minimum required environments.
- Pipeline execution logs are retained for 12 months per POL-017.
- Only the CTO can modify branch protection rules and required status checks.

### 5.4 Branch Protection and Repository Controls

The `main` branch of `QuestoM/cortex-sdk` enforces: no direct pushes (PRs only), minimum 1 approved review (2 for significant changes), all CI/CD status checks must pass, force pushes disabled, branch deletion restricted, stale review dismissal on new commits, and signed commits recommended (required for releases). The repository is private with access controlled per POL-002.

### 5.5 Dependency Management

1. **Approved Sources**: Dependencies may only be sourced from PyPI. Copyleft licenses (GPL) require CTO approval.
2. **Security Review**: New dependencies are reviewed for: maintenance status, known vulnerabilities, download volume, and security posture. Justification is required in the PR description.
3. **Version Pinning**: Production dependencies are pinned to specific versions in `pyproject.toml`. Version ranges are permitted only for development dependencies.
4. **Automated Scanning**: Dependencies are scanned for known vulnerabilities in every CI/CD run. Newly discovered vulnerabilities are remediated per Section 5.9 SLAs.

### 5.6 Static Application Security Testing (SAST) and Secret Scanning

1. **SAST in CI/CD**: Static analysis tools (mypy for type safety, ruff for linting and security patterns) execute on every pull request. Findings must be resolved before merge.
2. **Secret Scanning**: GitHub Advanced Security secret scanning with push protection is enabled on all repositories. Detected secrets (API keys for OpenAI, Google Gemini, Anthropic, AWS, Azure, GCP; SSH keys; TLS certificates; PyPI tokens; JWT secrets) block the push and alert the Security Lead. Compromised credentials are rotated within 1 hour.
3. **Dependency Scanning (SCA)**: `pip-audit` or equivalent scans all dependencies for known CVEs. CRITICAL and HIGH findings block merge. MEDIUM and LOW findings are tracked for remediation per the SLA in Section 5.9.

### 5.7 Software Bill of Materials (SBOM)

1. An SBOM in CycloneDX format is generated for every release using `scripts/generate_sbom.py`.
2. SBOMs are attached to GitHub Releases and retained for the lifetime of each release version.
3. Weekly scheduled CI runs generate updated SBOMs for the `main` branch to track dependency drift between releases.
4. SBOMs include: all direct and transitive dependencies, their versions, licenses, and known vulnerability status.
5. The SBOM is reviewed as part of the pre-release checklist defined in PROC-001 (PyPI Release Process).

### 5.8 Vulnerability Disclosure

1. **Reporting Channel**: Security vulnerabilities in the corteX SDK may be reported via a `SECURITY.md` file in the repository, which provides a private reporting channel (GitHub Security Advisories).
2. **Triage SLA**: Reported vulnerabilities are triaged by the Security Lead within 48 hours of receipt.
3. **Coordinated Disclosure**: Questo Ltd follows a coordinated disclosure process. Reporters are credited (with consent) in the security advisory.
4. **Security Advisories**: Confirmed vulnerabilities are published as GitHub Security Advisories with CVE identifiers where applicable.
5. **Customer Notification**: Enterprise customers are notified of security vulnerabilities affecting their deployed SDK versions per the notification procedures in POL-004.

### 5.9 Vulnerability Management SLAs

| Severity | Detection-to-Fix SLA | Release SLA | Examples |
|----------|----------------------|-------------|---------|
| **CRITICAL** | 24 hours | Patch release within 48 hours | Remote code execution, authentication bypass, tenant data leak |
| **HIGH** | 72 hours | Patch release within 1 week | Privilege escalation, data exposure, significant integrity issue |
| **MEDIUM** | 2 weeks | Next scheduled release | Denial of service, information disclosure (non-sensitive), logic error |
| **LOW** | 30 days | Next scheduled release | Minor issues, hardening recommendations |

### 5.10 Testing Requirements

#### 5.10.1 Test Coverage Expectations

- The corteX SDK maintains a comprehensive test suite of 8,082+ unit tests across 105 test files.
- New features must include unit tests that cover the primary success path, error handling paths, and edge cases.
- Bug fixes must include a regression test that demonstrates the bug and verifies the fix.
- Security-sensitive modules (security/, observability/) require tests for: normal operation, boundary conditions, malicious input handling, and tenant isolation.

#### 5.10.2 Test Categories

| Category | Scope | Execution | Requirements |
|----------|-------|-----------|--------------|
| **Unit Tests** | Individual functions and classes | Every CI/CD run | 8,082+ tests, 0 failures |
| **Integration Tests** | Module interactions, pipeline flows | Every CI/CD run | All pass (external services may be mocked) |
| **Security Tests** | Prompt injection, data leakage, tenant isolation | Every CI/CD run | All pass |
| **Performance Tests** | Latency, throughput, resource usage | On-demand / pre-release | No regressions beyond defined thresholds |

#### 5.10.3 Test Data Requirements

- Tests must use synthetic data only. Real customer data, production API keys, or personally identifiable information must never appear in test code or test fixtures.
- Test API keys and tokens use obviously fake values (e.g., `test-key-xxx`) that cannot be mistaken for real credentials.
- Mock LLM responses are used for unit tests to avoid external API dependencies and ensure tests run in air-gapped environments.

### 5.11 Documentation and Environment Separation

1. All public API changes must be reflected in the API reference documentation (MkDocs Material). New features require corresponding documentation entries. Security-relevant changes are flagged in the CHANGELOG. Breaking changes include migration guides.
2. Documentation is built and verified as part of the CI/CD pipeline.
3. Environments are segregated: Development (individual, synthetic data), CI/CD (service accounts, mock APIs), Staging (team, synthetic data), and Production/PyPI (CTO publishing). No customer data or production API keys are used in non-production environments.

## 6. Procedures

### 6.1 Feature Development Procedure

1. Engineer creates a GitHub issue with requirements and acceptance criteria. Significant features require a design document and CAB approval before implementation.
2. Engineer creates a feature branch (`feature/description`) from `main`, implements the change following Secure Coding Standards (Section 5.2), and writes tests concurrently.
3. Engineer creates a Pull Request per POL-003. CI/CD executes. Engineer addresses failures.
4. Peer reviewer(s) approve. The reviewer (not the author) merges. Feature branch is deleted.

### 6.2 Bug Fix Procedure

1. Engineer identifies or creates a GitHub issue with reproduction steps.
2. Engineer creates a fix branch (`fix/description`), writes a regression test demonstrating the bug, then implements the fix.
3. PR, CI/CD, and review follow the standard flow. PR includes root cause analysis.

### 6.3 Security Hotfix Procedure

1. Vulnerability is reported. Security Lead assesses severity per Section 5.9. For CRITICAL/HIGH: emergency change process per POL-003 Section 5.7 is invoked.
2. Engineer creates a `security/description` branch. Security Lead reviews with priority.
3. Upon merge, a patch release is prepared per SLA. Affected customers are notified per POL-004.

### 6.4 Release Procedure

1. Release Manager verifies all changes are merged and CI/CD passes, then creates a release PR updating the version number and CHANGELOG. CTO approves.
2. Upon merge, CI/CD builds and publishes to PyPI. Release Manager creates a GitHub Release.
3. Post-publish verification: `pip install cortex-ai==<version>` succeeds and version is correct. Documentation site is updated if necessary.

## 7. Exceptions

1. Exceptions to this policy are limited to emergency changes per POL-003 Section 5.7 and require CTO authorization.
2. No exceptions are permitted for: branch protection on `main`, the prohibition on self-merging PRs, the requirement for CI/CD pipeline execution, or the prohibition on hardcoded secrets.
3. Temporary test skips for emergency changes require a follow-up issue and must be resolved within 5 business days.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Circumventing branch protection rules, CI/CD requirements, or code review is a serious policy violation subject to disciplinary action.
- Committing secrets to the repository triggers immediate revocation and rotation of the exposed credentials, plus an incident investigation per POL-004.
- Self-approval of Pull Requests is prohibited and flagged during quarterly access reviews.
- Violations of secure coding standards are identified during code review and must be corrected before merge.
- The CTO reviews SDLC compliance monthly as part of operational metrics.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-002 | Access Control Policy |
| POL-003 | Change Management Policy |
| POL-004 | Incident Response Plan |
| POL-005 | Risk Assessment and Mitigation Policy |
| POL-008 | Data Classification Policy |
| POL-011 | Encryption Policy |
| POL-014 | Confidentiality Policy |
| POL-017 | Logging and Monitoring Policy |
| POL-020 | Workstation Security Policy |
| PROC-001 | PyPI Release Process |
| PROC-002 | Code Review Process |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
