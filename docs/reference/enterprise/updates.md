# Update Manager API Reference

## Module: `corteX.enterprise.updates`

SDK update manager handling delivery via private PyPI registries, signed package archives for air-gapped environments, and optional version checking. Never forces updates -- the customer controls their update cycle.

## Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `SDK_VERSION` | `"2.0.0"` | Current SDK version string |
| `SDK_VERSION_TUPLE` | `(2, 0, 0)` | Current SDK version as tuple |

## Enums

### `UpdateChannel`

```python
class UpdateChannel(str, Enum)
```

| Value | Description |
|-------|-------------|
| `STABLE` | Production-ready releases |
| `PREVIEW` | Pre-release for testing |
| `LTS` | Long-term support (enterprise) |

### `UpdateCheckResult`

```python
class UpdateCheckResult(str, Enum)
```

| Value | Description |
|-------|-------------|
| `UP_TO_DATE` | Running the latest version |
| `UPDATE_AVAILABLE` | A newer version is available |
| `CRITICAL_UPDATE` | A security update is available |
| `CHECK_FAILED` | The check failed (network/registry error) |
| `CHECK_DISABLED` | Update checks are disabled |

---

## Data Classes

### `VersionInfo`

**Type**: `@dataclass`

Information about a specific SDK version.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `version` | `str` | -- | Version string (e.g., `"2.1.0"`) |
| `channel` | `UpdateChannel` | -- | Release channel |
| `release_date` | `str` | -- | Release date (YYYY-MM-DD) |
| `changelog` | `str` | `""` | Release notes text |
| `min_python` | `str` | `"3.10"` | Minimum Python version |
| `breaking_changes` | `bool` | `False` | Whether this version has breaking changes |
| `security_fix` | `bool` | `False` | Whether this is a security fix |
| `checksum_sha256` | `str` | `""` | SHA256 checksum for verification |
| `download_url` | `str` | `""` | Package download URL |
| `size_bytes` | `int` | `0` | Package size |

### `UpdateConfig`

**Type**: `@dataclass`

Configuration for update behavior.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `check_enabled` | `bool` | `False` | Enable update checks (off by default for on-prem) |
| `check_interval_hours` | `int` | `24` | Hours between checks |
| `channel` | `UpdateChannel` | `STABLE` | Target release channel |
| `auto_notify` | `bool` | `True` | Log notification when update found |
| `registry_url` | `str` | `""` | Private PyPI registry URL |
| `registry_token` | `str` | `""` | Auth token for registry |
| `proxy_url` | `str` | `""` | HTTP proxy for air-gapped with proxy |
| `verify_signatures` | `bool` | `True` | Verify package signatures |
| `allowed_versions` | `List[str]` | `[]` | Pin to specific versions |
| `blocked_versions` | `List[str]` | `[]` | Block specific versions |

### `UpdateState`

**Type**: `@dataclass`

Persisted state for update tracking.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `current_version` | `str` | `SDK_VERSION` | Currently installed version |
| `last_check_time` | `float` | `0.0` | Timestamp of last check |
| `last_check_result` | `str` | `""` | Result of last check |
| `available_version` | `str` | `""` | Latest available version |
| `available_changelog` | `str` | `""` | Changelog for available version |
| `skipped_versions` | `List[str]` | `[]` | Versions the user chose to skip |
| `update_history` | `List[Dict]` | `[]` | History of performed updates (last 50) |

---

## Classes

### `UpdateManager`

Manages SDK update lifecycle for on-prem deployments.

Principles:

- Never force updates -- customer controls their cycle
- Never phone home without explicit opt-in
- Work fully offline -- version checks are optional
- Support air-gapped delivery via signed packages
- Provide clear upgrade paths and changelogs

#### Constructor

```python
UpdateManager(
    config: Optional[UpdateConfig] = None,
    state_path: Optional[str] = None,
)
```

**Parameters**:

- `config` (`Optional[UpdateConfig]`): Update configuration. Defaults to checks disabled
- `state_path` (`Optional[str]`): File path for persisting update state

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `current_version` | `str` | Current SDK version string |
| `version_tuple` | `tuple` | Current version as `(major, minor, patch)` |

#### Methods

##### `get_current_version`

```python
def get_current_version(self) -> VersionInfo
```

Get info about the currently installed version.

##### `should_check`

```python
def should_check(self) -> bool
```

Determine if an update check should run based on configuration and time elapsed since last check.

##### `check_for_updates`

```python
async def check_for_updates(self) -> UpdateCheckResult
```

Check for available updates. Only runs if `check_enabled=True` and a `registry_url` is configured. Respects `blocked_versions` and `allowed_versions` policies.

**Returns**: `UpdateCheckResult` indicating the outcome.

**Example**:

```python
from corteX.enterprise.updates import UpdateManager, UpdateConfig

manager = UpdateManager(
    config=UpdateConfig(
        check_enabled=True,
        registry_url="https://pypi.internal.acme.com/simple",
        channel="stable",
    ),
    state_path="~/.cortex/update_state.json",
)

result = await manager.check_for_updates()
if result == UpdateCheckResult.CRITICAL_UPDATE:
    print(f"Security update available: {manager.get_update_status()['available_version']}")
```

##### `verify_package`

```python
def verify_package(
    self,
    package_path: str,
    expected_checksum: Optional[str] = None,
) -> bool
```

Verify integrity of a downloaded package using SHA256. For air-gapped deployments where packages are transferred manually.

**Parameters**:

- `package_path` (str): Path to the `.whl` or `.tar.gz` file
- `expected_checksum` (`Optional[str]`): Expected checksum in format `"sha256:<hex>"`. If not provided, the calculated checksum is logged

**Returns**: `True` if checksum matches (or no expected checksum), `False` on mismatch or missing file.

##### `get_install_command`

```python
def get_install_command(self, version: Optional[str] = None) -> str
```

Get the pip install command for updating. Uses the configured `registry_url` if available.

**Returns**: Command string, e.g., `"pip install --index-url https://... cortex-engine==2.1.0"`.

##### `get_update_status`

```python
def get_update_status(self) -> Dict[str, Any]
```

Get current update status for reporting.

**Returns**:

```python
{
    "current_version": "2.0.0",
    "check_enabled": True,
    "last_check": 1707500000.0,
    "last_result": "up_to_date",
    "available_version": "",
    "channel": "stable",
    "registry_configured": True,
}
```

##### `record_update`

```python
def record_update(self, from_version: str, to_version: str) -> None
```

Record that an update was performed. Keeps last 50 update records.

##### `skip_version`

```python
def skip_version(self, version: str) -> None
```

Mark a version as skipped (will not notify again).

##### `generate_offline_manifest`

```python
def generate_offline_manifest(self, versions: List[VersionInfo]) -> str
```

Generate a JSON manifest for offline/air-gapped updates. This manifest can be distributed alongside signed packages to help admins verify and install updates.

**Returns**: JSON string with version metadata, checksums, and changelog.

---

## See Also

- [Updates & Versioning Guide](../../enterprise/updates.md)
- [On-Premises Deployment](../../enterprise/on-prem.md)
- [Enterprise Config API](./config.md)
