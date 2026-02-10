# Update Delivery

corteX supports multiple update delivery mechanisms, all designed with on-prem deployments in mind. Updates are never forced -- the customer controls their update cycle.

## Update Channels

| Channel | Description |
|---------|-------------|
| `STABLE` | Production-ready releases |
| `PREVIEW` | Pre-release builds for testing |
| `LTS` | Long-term support for enterprise deployments |

## Update Mechanisms

### 1. Private PyPI Registry (Connected)

For deployments with network access to an internal registry:

```python
from corteX.enterprise.updates import UpdateManager, UpdateConfig

manager = UpdateManager(
    config=UpdateConfig(
        check_enabled=True,
        check_interval_hours=24,
        channel=UpdateChannel.STABLE,
        registry_url="https://pypi.internal.acme.com/simple",
        registry_token="token-xxx",
        verify_signatures=True,
    ),
    state_path="~/.cortex/update_state.json",
)

# Check for updates (async, only hits the registry)
result = await manager.check_for_updates()
# UpdateCheckResult.UP_TO_DATE / UPDATE_AVAILABLE / CRITICAL_UPDATE

# Get the install command
cmd = manager.get_install_command()
# "pip install --index-url https://pypi.internal.acme.com/simple cortex-engine"
```

### 2. Signed Package Archives (Air-Gapped)

For environments with no network access:

```python
# Verify a manually transferred package
is_valid = manager.verify_package(
    package_path="cortex_engine-2.1.0-py3-none-any.whl",
    expected_checksum="sha256:abc123def456..."
)

if is_valid:
    # Install manually: pip install cortex_engine-2.1.0-py3-none-any.whl
    manager.record_update("2.0.0", "2.1.0")
```

### 3. Offline Manifest

Generate a manifest file for distribution alongside signed packages:

```python
from corteX.enterprise.updates import VersionInfo, UpdateChannel

versions = [
    VersionInfo(
        version="2.1.0",
        channel=UpdateChannel.STABLE,
        release_date="2026-02-15",
        changelog="Bug fixes and performance improvements",
        checksum_sha256="sha256:abc123...",
    ),
]

manifest_json = manager.generate_offline_manifest(versions)
# Write to file for distribution alongside .whl files
```

## Version Pinning

Admins can control which versions are allowed:

```python
config = UpdateConfig(
    allowed_versions=["2.0.0", "2.1.0"],   # Only these versions
    blocked_versions=["2.0.1"],              # Known bad version
)
```

## Update State

The `UpdateManager` persists its state to disk:

```python
status = manager.get_update_status()
# {
#   "current_version": "2.0.0",
#   "check_enabled": True,
#   "last_check": 1707500000.0,
#   "available_version": "2.1.0",
#   "channel": "stable",
#   "registry_configured": True,
# }
```

## Skipping Versions

Users can skip specific versions to suppress notifications:

```python
manager.skip_version("2.1.0")  # Won't notify about this version again
```

## Security

- **Signature verification**: When `verify_signatures=True`, package integrity is checked via SHA-256 checksums
- **No forced updates**: The SDK never auto-installs updates. It only notifies.
- **Single network call**: The version check (`_fetch_latest_version`) is the only network operation in the entire update system, and it only runs when explicitly enabled
- **Proxy support**: Air-gapped deployments with a proxy can set `proxy_url` in `UpdateConfig`
