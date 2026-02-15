# Data Classifier API Reference

## Module: `corteX.security.classification`

Automatic PII detection and data-level enforcement. Every piece of data flowing through corteX is classified into one of four levels. Classification can only escalate, never downgrade, through the processing pipeline.

## Classes

### `DataLevel`

**Type**: `IntEnum`

Data classification levels ordered by sensitivity. Uses `IntEnum` so that `max()` picks the most restrictive.

| Value | Name | Allowed Destinations | Requirements |
|-------|------|---------------------|-------------|
| `0` | `PUBLIC` | Any model (local or cloud) | None |
| `1` | `INTERNAL` | On-prem models only | None |
| `2` | `CONFIDENTIAL` | On-prem models only | Audit required |
| `3` | `RESTRICTED` | On-prem models only | Approval + full audit |

---

### `ClassificationResult`

**Type**: `@dataclass`

Outcome of a data classification operation.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `level` | `DataLevel` | The determined classification level |
| `reasons` | `List[str]` | Human-readable explanations for the classification |
| `pii_detected` | `List[str]` | PII types found (e.g., `["email", "ssn"]`) |
| `secrets_detected` | `List[str]` | Secret types found (e.g., `["openai_key"]`) |
| `patterns_matched` | `List[str]` | All pattern names that fired |

---

### `DataClassifier`

Assigns classification levels to text data based on PII detection, secret detection, and tenant policy.

#### Methods

##### `classify`

```python
def classify(
    self, text: str,
    tenant_policy: Optional[Dict[str, Any]] = None
) -> ClassificationResult
```

Classify text based on PII, secrets, and tenant policy.

**Parameters**:

- `text` (`str`): The text to classify.
- `tenant_policy` (`Optional[Dict[str, Any]]`): Optional policy dict with keys:
    - `default_level` (`int`): Base classification level.
    - `keywords` (`Dict[str, int]`): Keyword to `DataLevel` int mapping for tenant-specific escalation.
    - `always_restricted_patterns` (`List[str]`): Patterns that always escalate to RESTRICTED.

**Returns**: `ClassificationResult` with the computed level and detection details.

**Escalation rules**:

| Detection | Escalation |
|-----------|-----------|
| PII detected | At least `CONFIDENTIAL` |
| Secrets detected | At least `RESTRICTED` |
| Tenant keyword match | To the keyword's level |

##### `enforce`

```python
def enforce(
    self, data_level: DataLevel, target_model: str, is_local: bool
) -> Tuple[bool, Optional[str]]
```

Check if data at a classification level can be sent to a target model.

**Parameters**:

- `data_level` (`DataLevel`): Classification of the data.
- `target_model` (`str`): Model identifier (for logging).
- `is_local` (`bool`): Whether the target model runs on-prem.

**Returns**: `Tuple[bool, Optional[str]]` -- `(True, None)` if allowed, `(False, reason)` if blocked.

**Enforcement rules**:

| Level | Cloud Model | Local Model |
|-------|------------|------------|
| `PUBLIC` | Allowed | Allowed |
| `INTERNAL` | Blocked | Allowed |
| `CONFIDENTIAL` | Blocked | Allowed (audit) |
| `RESTRICTED` | Blocked | Allowed (approval + audit) |

##### `escalate`

```python
@staticmethod
def escalate(base_level: DataLevel, derived_level: DataLevel) -> DataLevel
```

Return the stricter of two classification levels. Classification can only escalate, never downgrade.

---

## PII Detection Patterns

| Pattern Name | Detects |
|-------------|---------|
| `email` | Email addresses |
| `phone_us` | US phone numbers (with optional country code) |
| `ssn` | US Social Security Numbers |
| `credit_card` | Visa, Mastercard, Amex, Discover card numbers |

## Secret Detection Patterns

| Pattern Name | Detects |
|-------------|---------|
| `api_key_generic` | Generic `api_key=...` patterns |
| `bearer_token` | Bearer authentication tokens |
| `password_field` | Password field assignments |
| `aws_key` | AWS access key IDs (`AKIA...`) |
| `github_token` | GitHub personal access tokens (`ghp_...`) |
| `openai_key` | OpenAI API keys (`sk-...`) |
| `private_key_header` | PEM private key headers |

All patterns are pre-compiled at module load for performance.

---

## Example

```python
from corteX.security.classification import DataClassifier, DataLevel

classifier = DataClassifier()

# Classify text with PII
result = classifier.classify("Send report to user@example.com")
assert result.level >= DataLevel.CONFIDENTIAL
assert "email" in result.pii_detected

# Classify text with secrets
result = classifier.classify("API key: sk-abc123def456ghi789")
assert result.level == DataLevel.RESTRICTED
assert "openai_key" in result.secrets_detected

# Enforce data routing
allowed, reason = classifier.enforce(
    DataLevel.CONFIDENTIAL, "gpt-4", is_local=False
)
assert not allowed  # CONFIDENTIAL cannot go to cloud models

allowed, reason = classifier.enforce(
    DataLevel.CONFIDENTIAL, "local-llama", is_local=True
)
assert allowed  # OK for local models

# Tenant-specific policy
result = classifier.classify(
    "Process the patient records",
    tenant_policy={
        "keywords": {"patient": 3},  # RESTRICTED
    },
)
assert result.level == DataLevel.RESTRICTED

# Escalation (never downgrades)
level = DataClassifier.escalate(DataLevel.INTERNAL, DataLevel.CONFIDENTIAL)
assert level == DataLevel.CONFIDENTIAL
```

---

## See Also

- [Risk Attenuator](./attenuation.md) -- Uses data classification for risk scoring
- [Compliance Engine](./compliance.md) -- Enforces regulatory rules per data level
- [Key Vault](./vault.md) -- Stores keys classified as RESTRICTED
