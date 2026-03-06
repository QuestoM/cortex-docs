# PyPI Release Process

**Document ID**: PROC-001
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2026-08-16
**Owner**: CTO
**Approved By**: CTO
**Classification**: INTERNAL

**SOC 2 Mapping**: CC6.1 (Logical and Physical Access Controls), CC7.1 (System Operations), CC8.1 (Change Management)

---

## 1. Purpose

This procedure defines the step-by-step process for releasing the `cortex-ai` Python package to PyPI. It ensures that every release is traceable, tested, reviewed, and compliant with SOC 2 change management requirements.

## 2. Scope

This procedure applies to all releases of the `cortex-ai` package, including:

- **Major releases** (e.g., 1.0.0 -> 2.0.0) -- breaking changes
- **Minor releases** (e.g., 1.0.0 -> 1.1.0) -- new features, backward-compatible
- **Patch releases** (e.g., 1.0.0 -> 1.0.1) -- bug fixes, security patches
- **Pre-releases** (e.g., 1.1.0rc1) -- release candidates for validation

## 3. Roles and Responsibilities

| Role | Responsibility |
|------|---------------|
| **Release Engineer** | Prepares the release, runs checks, executes the publish |
| **CTO** | Approves major version releases; reviews all release PRs |
| **Security Lead** | Reviews security-relevant changes before release |
| **QA Lead** | Confirms all tests pass and coverage meets thresholds |

## 4. Versioning Strategy

corteX follows [Semantic Versioning 2.0.0](https://semver.org/):

- **MAJOR** (`X.0.0`): Incompatible API changes. Requires CTO sign-off.
- **MINOR** (`0.X.0`): New features that are backward-compatible. Requires one reviewer.
- **PATCH** (`0.0.X`): Bug fixes and security patches. Requires one reviewer.

Version is defined in a single source of truth:

- `pyproject.toml` -> `[project] version = "X.Y.Z"`

## 5. Pre-Release Checklist

Before initiating a release, the Release Engineer must verify all items:

### 5.1. Code Quality

- [ ] All CI checks pass on the release branch (lint, type-check, tests)
- [ ] Full test suite passes: `pytest tests/ -v`
- [ ] Code coverage meets threshold: `pytest --cov=corteX --cov-fail-under=80`
- [ ] No `ruff` lint errors: `ruff check corteX/`
- [ ] No critical `mypy` errors: `mypy corteX/`

### 5.2. Documentation

- [ ] `CHANGELOG.md` updated with release notes for this version
- [ ] API documentation reflects any new or changed public interfaces
- [ ] Migration guide written for any breaking changes (major releases)
- [ ] MkDocs builds without errors: `mkdocs build --strict`

### 5.3. Security

- [ ] No hardcoded secrets, API keys, or credentials in the release
- [ ] All dependencies scanned for known vulnerabilities
- [ ] SBOM generated: `python scripts/generate_sbom.py`
- [ ] License file present and correct
- [ ] `py.typed` marker present for typed package

### 5.4. Compliance

- [ ] SBOM artifact generated and stored
- [ ] Release PR reviewed by at least one approved reviewer
- [ ] Major versions reviewed by CTO
- [ ] Security-relevant changes reviewed by Security Lead

## 6. Release Procedure

### Step 1: Create Release Branch

```bash
git checkout main
git pull origin main
git checkout -b release/vX.Y.Z
```

### Step 2: Bump Version

Update the version in `pyproject.toml`:

```toml
[project]
version = "X.Y.Z"
```

Update `corteX/enterprise/updates.py` if SDK_VERSION is defined there:

```python
SDK_VERSION = "X.Y.Z"
SDK_VERSION_TUPLE = (X, Y, Z)
```

### Step 3: Update CHANGELOG

Add a new section to `CHANGELOG.md`:

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Added
- ...

### Changed
- ...

### Fixed
- ...

### Security
- ...
```

### Step 4: Generate SBOM

```bash
python scripts/generate_sbom.py --output cortex-ai-sbom-vX.Y.Z.cdx.json
```

### Step 5: Run Full Validation

```bash
# Lint
ruff check corteX/

# Type check
mypy corteX/

# Full test suite
pytest tests/ -v --cov=corteX --cov-fail-under=80

# Docs build
mkdocs build --strict
```

### Step 6: Create Release PR

```bash
git add -A
git commit -m "release: prepare vX.Y.Z"
git push origin release/vX.Y.Z
gh pr create --title "Release vX.Y.Z" --body "Release checklist: [link]"
```

### Step 7: Obtain Approvals

- Patch/Minor: At least 1 approved review
- Major: CTO approval required
- Security changes: Security Lead approval required

### Step 8: Merge and Tag

After PR approval:

```bash
# Merge via GitHub (squash or merge commit per team preference)
git checkout main
git pull origin main
git tag -a vX.Y.Z -m "Release vX.Y.Z"
git push origin vX.Y.Z
```

### Step 9: Build and Publish

```bash
# Clean build
rm -rf dist/ build/ *.egg-info

# Build sdist and wheel
python -m build

# Verify the built packages
twine check dist/*

# Upload to PyPI (requires PyPI API token)
twine upload dist/*
```

**Important**: The PyPI API token must be stored as a GitHub secret (`PYPI_API_TOKEN`) or in a secure credential store. Never use personal PyPI passwords.

### Step 10: Create GitHub Release

```bash
gh release create vX.Y.Z \
  --title "cortex-ai vX.Y.Z" \
  --notes-file CHANGELOG_EXCERPT.md \
  dist/*.whl dist/*.tar.gz cortex-ai-sbom-vX.Y.Z.cdx.json
```

### Step 11: Post-Release Verification

```bash
# Verify package is available on PyPI
pip install cortex-ai==X.Y.Z --dry-run

# Smoke test installation
pip install cortex-ai==X.Y.Z
python -c "from corteX.sdk import CortexSDK; print('OK')"

# Verify version
python -c "from corteX.enterprise.updates import SDK_VERSION; print(SDK_VERSION)"
```

## 7. Rollback Procedure

If a critical issue is discovered after release:

### 7.1. Yank from PyPI

Yanking makes a version non-installable by default but does not delete it:

```bash
# Yank the broken version
pip install twine
# Log in to PyPI and yank via the web UI, or:
# twine does not support yank directly; use the PyPI web interface
```

Navigate to `https://pypi.org/manage/project/cortex-ai/release/X.Y.Z/` and click "Yank".

### 7.2. Publish Hotfix

```bash
# Create hotfix branch from the tag
git checkout -b hotfix/vX.Y.Z+1 vX.Y.Z

# Apply fix
# ... make changes ...

# Bump to patch version X.Y.Z+1
# Follow Steps 2-11 above with expedited review
```

### 7.3. Communication

- Notify affected users via GitHub Release notes
- Update CHANGELOG with rollback notice
- File an incident report per `POL-003_Change_Management_Policy`

## 8. Emergency Hotfix Process

For critical security vulnerabilities:

1. **Severity Assessment**: Confirm the issue is P0/Critical
2. **Expedited Review**: Security Lead + one engineer (minimum)
3. **Fast-Track Release**: Follow Steps 2-11 with expedited timeline
4. **Post-Incident**: Full incident report within 48 hours
5. **Retrospective**: Process improvement review within 1 week

## 9. Automation

The following CI/CD workflows support this process:

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `sbom.yml` | Release, weekly | Generate SBOM |
| (future) `release.yml` | Tag push | Automated PyPI publish |
| (future) `verify-release.yml` | Post-release | Smoke tests |

## 10. Audit Trail

Every release produces the following audit artifacts:

- Git tag with GPG signature (when configured)
- GitHub Release with attached SBOM and built packages
- PR review history with approval records
- CI/CD workflow logs (retained per GitHub policy)
- SBOM artifact (retained 90 days in CI, permanently in Release)

## 11. Review Schedule

This procedure is reviewed:

- Every 6 months (routine)
- After any failed or rolled-back release (incident-driven)
- When tooling or infrastructure changes significantly
