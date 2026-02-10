# Security Model

corteX follows a zero-trust, zero-outbound security model. This page describes how data isolation, network controls, and policy enforcement work.

## Zero Outbound Calls

The corteX SDK makes **no outbound network calls** unless the developer explicitly configures an integration or tool that requires one. This is a fundamental design constraint, not a configuration option.

The following components are the *only* sources of network traffic:

| Component | When It Calls Out | How to Disable |
|-----------|------------------|----------------|
| LLM Provider (Gemini, Claude, etc.) | When generating responses | Use local models |
| Tool with network access (browser, web_search) | When explicitly invoked | Remove from `ToolPolicy.allowed_tools` |
| Update checker | Only when `UpdateConfig.check_enabled=True` AND a registry URL is set | Leave `check_enabled=False` (default) |
| Global weight sync (Tier 4) | Only when explicitly opted in | Leave `Tier4Global.enabled=False` (default) |
| License sync | Optional, never required | Don't configure sync endpoint |

Everything else -- the weight engine, memory fabric, plasticity rules, feedback engine, goal tracker, prediction engine, context engine, and all enterprise configuration -- operates entirely in-process with zero network I/O.

## Data Isolation

### Per-Tenant Boundaries

Each `TenantConfig` creates a hard isolation boundary. There is no shared state between tenants:

- Separate weight snapshots
- Separate memory stores (working, episodic, semantic)
- Separate audit logs
- Separate license state
- Separate tool policies

### Data Retention Controls

The `DataRetention` enum controls what persists beyond a session:

```python
from corteX.enterprise.config import DataRetention

# Nothing stored anywhere
config.data_retention = DataRetention.NONE

# Only for the duration of the session
config.data_retention = DataRetention.SESSION

# Persisted across sessions (file-based)
config.data_retention = DataRetention.PERSISTENT

# Only audit logs are stored, no user data
config.data_retention = DataRetention.AUDIT_ONLY
```

## Tool Policy Enforcement

The `ToolPolicy` controls which tools agents can invoke:

```python
from corteX.enterprise.config import ToolPolicy

policy = ToolPolicy(
    allowed_tools=["code_interpreter", "file_read"],  # Whitelist
    blocked_tools=["bash_unrestricted"],                # Blacklist
    require_approval_tools=["file_delete", "deploy"],   # Human-in-the-loop
    max_tool_calls_per_turn=10,
    tool_timeout_seconds=30,
)

# Runtime check
policy.is_tool_allowed("code_interpreter")    # True
policy.is_tool_allowed("bash_unrestricted")   # False
```

## Model Policy Enforcement

Control which LLM models are accessible:

```python
from corteX.enterprise.config import ModelPolicy

models = ModelPolicy(
    allowed_models=["claude-opus-4-20250514"],
    blocked_models=["gpt-4o"],
    max_tokens_per_request=32000,
    max_tokens_per_session=500000,
    allow_code_execution=False,  # Disable code interpreter features
)

models.is_model_allowed("claude-opus-4-20250514")  # True
models.is_model_allowed("gpt-4o")                  # False
```

## Safety Policy as Security Layer

The `SafetyPolicy` acts as an additional security layer:

- **Blocked topics**: Prevent agents from discussing specific subjects
- **Blocked patterns**: Regex-based content filtering
- **Injection protection**: Guards against prompt injection attacks
- **PII detection**: Automatically detects and masks personal information
- **Autonomy cap**: Limits how independently agents can act (0.0-1.0)
- **Human approval gates**: Require explicit approval for specific actions

## Enterprise Modulation Policies

The `TargetedModulator` provides an additional security control layer via enterprise policies. These policies use SHA-256 integrity hashes for tamper detection and priority levels that override all user-level controls:

```python
from corteX.engine.modulator import Policy, ModulationType

policy = Policy(
    name="block_dangerous_tools",
    target_pattern="tool_dangerous_*",
    modulation_type=ModulationType.SILENCE,
    priority=100,  # Enterprise policies always >= 100
    created_by="security_admin",
)

# Integrity verification
assert policy.verify_integrity()  # Detects tampering
```

## Audit Trail

Every security-relevant event is logged when audit is enabled. See [Audit](audit.md) for details on configuring the audit trail for compliance.
