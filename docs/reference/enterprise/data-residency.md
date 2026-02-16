# Data Residency Manager API Reference

## Module: `corteX.enterprise.data_residency`

GDPR Art. 44-49 compliance -- data residency controls. Ensures personal data stays within configured geographic regions by validating LLM provider endpoints against tenant-specific region policies. Enterprise tenants restrict which endpoints are allowed based on where those endpoints process data.

Thread-safe via internal lock for concurrent access.

## Enums

### `Region`

**Type**: `str, Enum`

Standard geographic regions for data residency.

| Value | Description |
|-------|-------------|
| `us` | United States |
| `eu` | European Union |
| `uk` | United Kingdom |
| `asia_pacific` | Asia-Pacific region |
| `canada` | Canada |
| `australia` | Australia |
| `japan` | Japan |
| `india` | India |
| `brazil` | Brazil |
| `middle_east` | Middle East |
| `global` | Global / unspecified |

---

## Data Classes

### `ResidencyCheck`

**Type**: `@dataclass`

Result of a data residency validation.

| Attribute | Type | Description |
|-----------|------|-------------|
| `allowed` | `bool` | Whether the provider endpoint is allowed |
| `region` | `str` | Detected region of the provider |
| `provider` | `str` | Provider name or URL |
| `reason` | `str` | Human-readable explanation |
| `tenant_id` | `str` | Tenant that was checked |
| `checked_at` | `float` | Timestamp of the check |

### `TenantResidencyConfig`

**Type**: `@dataclass`

Per-tenant data residency configuration.

| Attribute | Type | Description |
|-----------|------|-------------|
| `tenant_id` | `str` | Tenant identifier |
| `allowed_regions` | `List[str]` | Regions where data may be processed |
| `blocked_regions` | `List[str]` | Explicitly blocked regions (take precedence) |
| `strict_mode` | `bool` | If `True`, unknown regions are blocked |
| `updated_at` | `float` | Last update timestamp |
| `updated_by` | `str` | Who last updated the config |

---

## Built-in Provider Mappings

The module includes pre-built mappings for major LLM providers and their regional endpoints:

| Provider | Endpoint Pattern | Region |
|----------|-----------------|--------|
| **OpenAI** | `api.openai.com` | US |
| **Azure OpenAI** | `eastus*.openai.azure.com` | US |
| **Azure OpenAI** | `westeurope*.openai.azure.com` | EU |
| **Azure OpenAI** | `uksouth*.openai.azure.com` | UK |
| **Azure OpenAI** | `japaneast*.openai.azure.com` | Japan |
| **Azure OpenAI** | `australiaeast*.openai.azure.com` | Australia |
| **Azure OpenAI** | `canadacentral*.openai.azure.com` | Canada |
| **Gemini** | `generativelanguage.googleapis.com` | US |
| **Gemini** | `europe-west*-aiplatform.googleapis.com` | EU |
| **Anthropic** | `api.anthropic.com` | US |
| **Local** | `localhost`, `127.0.0.1`, RFC 1918 ranges | Local |

Local/on-prem endpoints are always allowed regardless of region policy.

---

## Classes

### `DataResidencyManager`

Enforce data residency requirements per tenant.

#### Constructor

```python
DataResidencyManager()
```

No parameters. Creates a new manager with empty tenant configurations.

#### Methods

##### `set_allowed_regions`

```python
def set_allowed_regions(
    self, tenant_id: str, regions: List[str],
    strict_mode: bool = True, updated_by: str = "",
) -> TenantResidencyConfig
```

Configure allowed data regions for a tenant.

**Parameters**:

- `tenant_id` (`str`): Non-empty tenant identifier.
- `regions` (`List[str]`): Non-empty list of allowed region codes (e.g., `["eu", "uk"]`).
- `strict_mode` (`bool`): If `True`, unknown/undetectable regions are blocked. Default: `True`.
- `updated_by` (`str`): Who is making this change (for audit).

**Returns**: `TenantResidencyConfig` -- The created configuration.

**Raises**: `ValueError` if `tenant_id` is empty or `regions` is empty.

##### `set_blocked_regions`

```python
def set_blocked_regions(self, tenant_id: str, regions: List[str]) -> None
```

Set explicitly blocked regions (take precedence over allowed regions).

**Raises**: `ValueError` if no residency config exists for the tenant.

##### `validate_provider`

```python
def validate_provider(self, tenant_id: str, provider_url: str) -> ResidencyCheck
```

Validate that a provider endpoint is in an allowed region.

**Parameters**:

- `tenant_id` (`str`): Tenant to check against.
- `provider_url` (`str`): Full URL of the provider endpoint.

**Returns**: `ResidencyCheck` -- Validation result with region detection and explanation.

**Behavior**:

- If no policy is configured, all regions are allowed.
- Local/on-prem endpoints are always allowed.
- Blocked regions take precedence over allowed regions.
- In `strict_mode`, unknown regions are blocked.

##### `get_tenant_regions`

```python
def get_tenant_regions(self, tenant_id: str) -> List[str]
```

Get allowed regions for a tenant (empty list if unconfigured).

##### `get_tenant_config`

```python
def get_tenant_config(self, tenant_id: str) -> Optional[TenantResidencyConfig]
```

Get the full residency config for a tenant.

##### `remove_tenant_config`

```python
def remove_tenant_config(self, tenant_id: str) -> bool
```

Remove residency config for a tenant (GDPR erasure). Returns `True` if config existed.

##### `add_custom_mapping`

```python
def add_custom_mapping(self, pattern: str, region: str, provider: str = "") -> None
```

Add a custom provider-to-region mapping. Custom mappings are checked before built-in ones. Use this for internal or custom LLM endpoints.

**Parameters**:

- `pattern` (`str`): Regex pattern to match against the provider URL hostname.
- `region` (`str`): Region code to assign.
- `provider` (`str`): Optional provider name for the result.

##### `detect_region`

```python
def detect_region(self, provider_url: str) -> str
```

Public helper: detect the region of a provider URL. Returns the region code string or `"unknown"`.

##### `get_validation_log`

```python
def get_validation_log(self) -> List[Dict]
```

Get the validation log for compliance auditing. Returns all `ResidencyCheck` results.

##### `get_supported_regions`

```python
def get_supported_regions(self) -> List[str]
```

List all standard supported region codes.

---

## Example

```python
from corteX.enterprise.data_residency import DataResidencyManager

drm = DataResidencyManager()

# EU-only tenant (GDPR Art. 44-49)
drm.set_allowed_regions("acme_eu", ["eu", "uk"], strict_mode=True)

# Validate provider endpoints
check = drm.validate_provider("acme_eu", "https://api.openai.com/v1")
print(check.allowed)  # False -- OpenAI is US, not EU/UK
print(check.reason)   # "Provider 'openai' is in region 'us', which is NOT in..."

check = drm.validate_provider(
    "acme_eu",
    "https://westeurope.openai.azure.com/v1"
)
print(check.allowed)  # True -- Azure West Europe is EU

# Local endpoints are always allowed
check = drm.validate_provider("acme_eu", "http://localhost:8080/v1")
print(check.allowed)  # True -- on-prem always allowed

# Block specific regions
drm.set_blocked_regions("acme_eu", ["us"])

# Custom mapping for internal endpoints
drm.add_custom_mapping(
    pattern=r"llm\.internal\.acme\.com",
    region="eu",
    provider="acme_internal",
)

# Detect region without validation
region = drm.detect_region("https://api.anthropic.com/v1")
print(region)  # "us"

# Compliance audit log
log = drm.get_validation_log()
print(f"{len(log)} validations recorded")

# GDPR erasure
drm.remove_tenant_config("acme_eu")
```

---

## See Also

- [GDPR Manager](./gdpr.md) -- Full DSAR lifecycle
- [Compliance Engine](../security/compliance.md) -- Framework-level policy enforcement
- [Tenant Manager](../tenancy/manager.md) -- Per-tenant configuration
- [LLM Router](../llm/router.md) -- Model routing (integrates with residency checks)
