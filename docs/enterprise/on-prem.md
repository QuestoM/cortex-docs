# On-Premises Deployment

corteX is designed on-prem first. Every feature works without internet access.

## Zero Network Dependency

The core SDK has zero network dependencies for operation:

- **Weight engine**: Pure in-process computation
- **Memory fabric**: In-memory or file-based backends
- **Plasticity, adaptation, feedback**: All local
- **Enterprise config**: JSON files on local disk
- **License validation**: Ed25519 signatures verified locally
- **Audit logging**: File-based or syslog

The only components that require network access are LLM providers (which can be replaced with local models) and explicitly configured tools.

## Local Model Support

For fully air-gapped deployments, configure local LLM providers:

```python
from corteX.enterprise.config import ModelPolicy

models = ModelPolicy(
    orchestrator_model="local-llama-70b",
    worker_model="local-llama-8b",
    allowed_models=["local-llama-70b", "local-llama-8b"],
)
```

## File-Based Persistence

All data can be stored as local JSON files:

```python
from corteX.engine.memory import MemoryFabric, FileBackend

# Persistent memory using local files
fabric = MemoryFabric(
    working_backend=FileBackend("/data/cortex/working"),
    episodic_backend=FileBackend("/data/cortex/episodic"),
    semantic_backend=FileBackend("/data/cortex/semantic"),
)

# Weight persistence
engine = WeightEngine()
engine.save("/data/cortex/weights.json")
engine.load("/data/cortex/weights.json")

# Enterprise config persistence
config.save("/data/cortex/tenant_config.json")
```

## Air-Gapped Updates

For environments with no internet access, corteX supports signed package delivery. See [Updates](updates.md) for the full air-gapped update workflow.

## Deployment Checklist

1. **License key**: Obtain and activate before deployment. The 30-day grace period ensures continuity during renewal.
2. **Tenant config**: Create and save to a known path. Load at application startup.
3. **Model provider**: Configure local models or ensure the LLM API endpoint is reachable from the deployment.
4. **Storage paths**: Set up directories for weights, memory, audit logs, and license state.
5. **Update channel**: Configure `UpdateConfig` if connected, or plan manual update delivery if air-gapped.

## Disabling All Network Features

To guarantee zero network traffic:

```python
from corteX.enterprise.config import TenantConfig, ModelPolicy
from corteX.enterprise.updates import UpdateConfig

config = TenantConfig(
    tenant_id="airgapped",
    models=ModelPolicy(
        allowed_models=["local-model"],
        orchestrator_model="local-model",
    ),
)

# Updates: disabled by default, but be explicit
update_config = UpdateConfig(
    check_enabled=False,   # No version checks
    registry_url="",       # No registry
)

# Global weight sync: disabled by default
# Tier4Global.enabled = False (this is the default)
```
