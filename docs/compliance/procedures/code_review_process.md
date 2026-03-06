# Code Review Process

**Document ID**: PROC-002
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2026-08-16
**Owner**: CTO
**Approved By**: CTO
**Classification**: INTERNAL

**SOC 2 Mapping**: CC6.1 (Logical Access Controls), CC6.6 (Security Over System Changes), CC8.1 (Change Management)

---

## 1. Purpose

This procedure defines the mandatory code review process for all changes to the `cortex-ai` codebase. It ensures that every change is reviewed for correctness, security, and compliance before merging into protected branches.

## 2. Scope

This procedure applies to:

- All pull requests targeting `main` or `release/*` branches
- All code changes to the `corteX/` package directory
- All changes to CI/CD workflows (`.github/`)
- All changes to documentation (`docs/`)
- All changes to test suites (`tests/`)

## 3. Roles and Responsibilities

| Role | Responsibility |
|------|---------------|
| **Author** | Creates the PR, ensures it meets all requirements, addresses review feedback |
| **Reviewer** | Reviews code for correctness, security, style, and test coverage |
| **Security Reviewer** | Additional review for security-relevant changes |
| **CTO** | Approves major version releases and architectural changes |
| **CI System** | Automated checks (lint, type-check, tests, SBOM) |

## 4. Pull Request Requirements

Every pull request must include:

### 4.1. Description

- Clear summary of what the change does and why
- Link to the relevant issue or ticket (if applicable)
- List of breaking changes (if any)
- Testing approach description

### 4.2. Code Quality

- [ ] No hardcoded secrets, API keys, or credentials
- [ ] Type hints on all public functions
- [ ] Max 300 lines per file (per project code rules)
- [ ] `logging` used instead of `print` statements
- [ ] Async by default for I/O operations
- [ ] No Google Cloud hard dependencies in core modules

### 4.3. Testing

- [ ] New or changed functionality has corresponding tests
- [ ] All existing tests pass
- [ ] Test coverage does not decrease below 80%
- [ ] Integration tests clearly marked with `@pytest.mark.integration`

### 4.4. Documentation

- [ ] Public API changes reflected in docstrings
- [ ] User-facing changes noted in CHANGELOG.md
- [ ] New modules added to MkDocs navigation (if applicable)

## 5. Review Requirements

### 5.1. Standard Changes

All pull requests require:

- **Minimum 1 approved review** from a team member
- **All CI checks passing** (lint, type-check, tests)
- **No unresolved review comments**

### 5.2. Security-Relevant Changes

Changes that touch any of the following require **2 approved reviews**, including one from the Security Lead:

- `corteX/enterprise/` -- tenant isolation, licensing, safety
- `corteX/security/` -- vault, key management, encryption
- `corteX/core/contracts.py` -- core data models and event contracts
- `.github/workflows/` -- CI/CD pipeline changes
- Any file handling API keys, credentials, or PII
- Changes to safety policies, injection protection, or PII detection
- Dependency version changes in `pyproject.toml`

### 5.3. Architectural Changes

Changes that modify the fundamental architecture require:

- **CTO approval**
- Design document or ADR (Architecture Decision Record) linked in PR
- Impact assessment on existing tenants

## 6. Branch Protection Rules

The following branch protection rules must be configured on GitHub:

### 6.1. `main` Branch

| Setting | Value |
|---------|-------|
| Require pull request reviews | Yes |
| Required number of reviewers | 1 |
| Dismiss stale reviews on new pushes | Yes |
| Require review from code owners | Yes (when CODEOWNERS configured) |
| Require status checks to pass | Yes |
| Required status checks | `lint`, `type-check`, `test` |
| Require branches to be up to date | Yes |
| Require signed commits | Recommended |
| Include administrators | Yes |
| Allow force pushes | No |
| Allow deletions | No |

### 6.2. `release/*` Branches

Same as `main`, with the addition of:

| Setting | Value |
|---------|-------|
| Required number of reviewers | 2 |
| Restrict who can push | Release Engineers only |

### 6.3. CODEOWNERS File

The repository should include a `CODEOWNERS` file:

```
# Default owner for everything
* @QuestoM/core-team

# Enterprise and security modules require security review
/corteX/enterprise/ @QuestoM/security-team
/corteX/security/   @QuestoM/security-team

# CI/CD changes require DevOps review
/.github/           @QuestoM/devops-team

# Documentation changes
/docs/              @QuestoM/docs-team
```

## 7. Review Checklist

Reviewers should evaluate each PR against this checklist:

### 7.1. Correctness

- [ ] Logic is correct and handles edge cases
- [ ] Error handling is appropriate (no silent failures)
- [ ] Resource cleanup is handled (context managers, finally blocks)
- [ ] Concurrent/async code is safe (no race conditions)

### 7.2. Security

- [ ] No secrets or credentials in code
- [ ] Input validation is present where needed
- [ ] SQL/command injection risks mitigated
- [ ] Prompt injection protections maintained
- [ ] PII handling follows data retention policies
- [ ] Cross-tenant data isolation preserved

### 7.3. Performance

- [ ] No unnecessary I/O in hot paths
- [ ] Memory usage is reasonable
- [ ] No N+1 query patterns or unbounded loops
- [ ] Timeouts configured for external calls

### 7.4. Maintainability

- [ ] Code is readable and well-structured
- [ ] Functions are focused (single responsibility)
- [ ] Naming is clear and consistent
- [ ] Complex logic has explanatory comments

## 8. Merge Criteria

A PR may be merged when ALL of the following are true:

1. **Required reviews obtained** (1 for standard, 2 for security)
2. **All CI checks pass** (no failures or skipped required checks)
3. **No unresolved conversations** (all reviewer comments addressed)
4. **Branch is up to date** with the target branch
5. **CHANGELOG updated** (for user-facing changes)
6. **No open TODO/FIXME** introduced without a linked issue

### 8.1. Merge Strategy

- **Squash merge** for feature branches (clean history)
- **Merge commit** for release branches (preserve release commits)
- **Rebase** only for small, single-commit fixes

## 9. Emergency Hotfix Process

For P0/Critical security vulnerabilities or production-breaking bugs:

### 9.1. Criteria for Emergency Hotfix

- Active security vulnerability being exploited
- Complete failure of core SDK functionality
- Data corruption or loss risk

### 9.2. Expedited Process

1. **Create hotfix branch** from the latest release tag
2. **Minimal change** -- fix only the critical issue, nothing else
3. **Expedited review** -- 1 reviewer minimum (Security Lead for security issues)
4. **Fast-track merge** -- reviewer may approve and merge immediately
5. **Post-merge validation** -- full test suite run after merge
6. **Follow-up PR** -- comprehensive fix with full review within 48 hours
7. **Incident report** -- document the hotfix per incident management policy

### 9.3. Documentation

Emergency hotfixes must be documented:

- In the PR description: reason for expedited process
- In the CHANGELOG: marked as `[HOTFIX]`
- In the incident log: per `POL-003_Change_Management_Policy`

## 10. Metrics and Monitoring

The following metrics are tracked for process health:

| Metric | Target | Review Frequency |
|--------|--------|-----------------|
| PR review turnaround time | < 24 hours | Monthly |
| CI pass rate | > 95% | Weekly |
| Review comment resolution time | < 48 hours | Monthly |
| Emergency hotfix frequency | < 1 per quarter | Quarterly |
| PRs merged without review | 0 (enforced by branch protection) | Continuous |

## 11. Audit Trail

Every code change produces the following audit artifacts:

- Pull request with full discussion history
- Review approvals with reviewer identity and timestamp
- CI/CD check results (lint, tests, type-check, SBOM)
- Merge commit linking the PR number
- Git blame history for every line of code

## 12. Review Schedule

This procedure is reviewed:

- Every 6 months (routine)
- After any security incident involving code changes
- When team size or structure changes significantly
- When CI/CD tooling is modified
