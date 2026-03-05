# Data Classification

Every piece of data flowing through corteX is automatically classified
into one of four sensitivity levels. The classification can only
escalate, never downgrade, through the processing pipeline. This
ensures that sensitive data is never accidentally sent to an
inappropriate model or destination.

## How It Works

The `DataClassifier` scans text for two categories of sensitive
content using pre-compiled regular expressions:

**PII patterns** (escalate to CONFIDENTIAL):

- Email addresses
- US phone numbers
- Social Security Numbers
- Credit card numbers (Visa, MasterCard, Amex, Discover)
- Israeli ID numbers (with Luhn-like check-digit validation)

**Secret patterns** (escalate to RESTRICTED):

- Generic API keys
- Bearer tokens
- Password fields
- AWS access keys
- GitHub tokens
- OpenAI keys
- Private key headers

Classification levels determine where data can be processed:

| Level | Cloud Models | On-Prem Models | Audit | Approval |
|-------|-------------|----------------|-------|----------|
| PUBLIC | Yes | Yes | No | No |
| INTERNAL | No | Yes | No | No |
| CONFIDENTIAL | No | Yes | Required | No |
| RESTRICTED | No | Yes | Required | Required |

## Key Features

- **Automatic PII detection** - regex-based scanning for emails, SSNs, credit cards, phone numbers
- **Secret detection** - catches API keys, tokens, passwords, and private keys
- **Post-validation** - Israeli IDs go through a Luhn-like check-digit algorithm to reduce false positives
- **Monotonic escalation** - `escalate()` always picks the stricter level; never downgrades
- **Tenant policy support** - custom keyword-to-level mappings per tenant
- **Enforcement** - `enforce()` blocks cloud model usage for non-PUBLIC data
- **IntEnum ordering** - `DataLevel` uses `IntEnum` so `max()` naturally picks the most restrictive level

## Integration

The Data Classifier feeds into multiple corteX subsystems. The Risk
Attenuator uses the classification level as one of its four risk
factors. The Compliance Engine checks data levels against regulatory
framework requirements (GDPR, HIPAA, SOC 2). The LLM routing layer
calls `enforce()` before sending any prompt to verify the target model
is appropriate for the data's sensitivity.

Tenant policies can customize classification by defining keyword
mappings - for example, mapping the word "patient" to RESTRICTED for
a healthcare tenant.

## Usage Example

```python
from corteX.security.classification import DataClassifier, DataLevel

classifier = DataClassifier()

# Classify text with PII
result = classifier.classify("Contact user@example.com for details")
assert result.level >= DataLevel.CONFIDENTIAL
assert "email" in result.pii_detected

# Classify text with secrets
result = classifier.classify("API key: sk-abc123longkeyhere...")
assert result.level == DataLevel.RESTRICTED
assert "openai_key" in result.secrets_detected

# Enforce routing rules
allowed, reason = classifier.enforce(
    data_level=DataLevel.CONFIDENTIAL,
    target_model="gpt-4",
    is_local=False,
)
assert not allowed  # CONFIDENTIAL cannot go to cloud

# Tenant-specific policy
result = classifier.classify(
    "Patient record #12345",
    tenant_policy={
        "keywords": {"patient": 3},  # 3 = RESTRICTED
    },
)
assert result.level == DataLevel.RESTRICTED

# Escalation is monotonic
combined = DataClassifier.escalate(DataLevel.INTERNAL, DataLevel.CONFIDENTIAL)
assert combined == DataLevel.CONFIDENTIAL  # Stricter wins
```

## See Also

- [Risk-Based Attenuation](risk-attenuation.md) - uses data level as a risk factor
- [Compliance Engine](compliance-engine.md) - enforces regulatory rules based on data level
- [Key Vault](key-vault.md) - secure storage for the secrets that classification detects
