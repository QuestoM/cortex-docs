# PII Tokenizer API Reference

## Module: `corteX.security.pii_tokenizer`

Reversible PII redaction for the LLM pipeline. Replaces PII (email, phone, SSN, credit card, name, address, IP address) with deterministic tokens like `[PII_EMAIL_001]` before sending text to an LLM. After the LLM responds, `detokenize()` restores original values.

Per-tenant isolation guarantees that tenant A's PII tokens never leak to tenant B. Thread safety is provided via per-tenant locks.

## Supported PII Types

Patterns are processed in priority order (first match wins for overlapping regions):

| PII Type | Token Format | Example Match |
|----------|-------------|---------------|
| SSN | `[PII_SSN_001]` | `123-45-6789` |
| Credit Card | `[PII_CREDIT_CARD_001]` | `4111 1111 1111 1111` |
| Email | `[PII_EMAIL_001]` | `user@example.com` |
| Phone | `[PII_PHONE_001]` | `(555) 123-4567` |
| IP Address | `[PII_IP_ADDRESS_001]` | `192.168.1.100` |
| Name | `[PII_NAME_001]` | `Dr. John Smith` |
| Address | `[PII_ADDRESS_001]` | `123 Main St, NY 10001` |

---

## Data Classes

### `TokenizedResult`

**Type**: `@dataclass`

Result of PII tokenization.

| Attribute | Type | Description |
|-----------|------|-------------|
| `text` | `str` | Text with PII replaced by tokens |
| `token_map` | `Dict[str, str]` | Mapping of token string to original PII value |
| `pii_count` | `int` | Total number of PII values tokenized |

---

## Classes

### `PIITokenizer`

Reversible PII tokenizer with per-tenant isolation. Replaces detected PII with tokens of the form `[PII_{TYPE}_{NNN}]` and maintains a per-tenant mapping for restoration.

Thread-safe: each tenant has its own lock so concurrent tokenize/detokenize calls for different tenants never block each other.

#### Constructor

```python
PIITokenizer(classifier: Optional[DataClassifier] = None)
```

**Parameters**:

- `classifier` (`Optional[DataClassifier]`): Optional `DataClassifier` instance. If `None`, a default one is created.

#### Methods

##### `tokenize`

```python
def tokenize(self, text: str, tenant_id: str) -> TokenizedResult
```

Replace PII values in text with reversible tokens.

**Parameters**:

- `text` (`str`): Input text potentially containing PII.
- `tenant_id` (`str`): Non-empty tenant identifier for isolation.

**Returns**: `TokenizedResult` with tokenized text and the mapping.

**Behavior**:

- Same PII value within a tenant session always gets the same token (consistent tokenization).
- Overlapping matches are resolved by priority order (SSN > credit card > email > phone > IP > name > address).
- Empty or whitespace-only text returns immediately with `pii_count=0`.

**Raises**: `ValueError` if `tenant_id` is empty.

##### `detokenize`

```python
def detokenize(self, text: str, tenant_id: str) -> str
```

Restore original PII values from tokens in text.

**Parameters**:

- `text` (`str`): Text containing PII tokens (e.g., `[PII_EMAIL_001]`).
- `tenant_id` (`str`): Must match the tenant used during tokenization.

**Returns**: `str` -- Text with tokens replaced by original PII values. Unknown tokens are left unchanged.

##### `clear_tenant`

```python
def clear_tenant(self, tenant_id: str) -> None
```

Destroy all token mappings for a tenant (GDPR erasure support). After clearing, previous tokens for this tenant can no longer be detokenized.

##### `get_token_map`

```python
def get_token_map(self, tenant_id: str) -> Dict[str, str]
```

Return a copy of the current token-to-value map for a tenant. Returns an empty dict if no tokens exist.

##### `get_pii_count`

```python
def get_pii_count(self, tenant_id: str) -> int
```

Return the number of unique PII values tracked for a tenant.

---

## Security Design

| Property | Implementation |
|----------|---------------|
| **Tenant isolation** | Separate token state per tenant with individual locks |
| **Consistent tokens** | Same PII value always maps to the same token within a session |
| **Priority ordering** | High-sensitivity PII (SSN, credit card) detected first |
| **Overlap handling** | First match wins; overlapping regions are skipped |
| **Thread safety** | Per-tenant `threading.Lock`; global lock for state creation |
| **GDPR erasure** | `clear_tenant()` destroys all mappings for a tenant |

!!! warning "Token Persistence"
    Token mappings are held in memory only. If the process restarts, previous tokens cannot be detokenized. For durable PII tokenization, persist the token map externally.

---

## Example

```python
from corteX.security.pii_tokenizer import PIITokenizer

tokenizer = PIITokenizer()

# Tokenize text before sending to LLM
text = "Contact John at john.doe@acme.com or call (555) 123-4567"
result = tokenizer.tokenize(text, tenant_id="acme")
print(result.text)
# "Contact John at [PII_EMAIL_001] or call [PII_PHONE_001]"
print(result.pii_count)  # 2

# Send result.text to LLM... LLM responds with tokens intact
llm_response = "I'll contact [PII_EMAIL_001] right away."

# Restore original PII values
restored = tokenizer.detokenize(llm_response, tenant_id="acme")
print(restored)
# "I'll contact john.doe@acme.com right away."

# Consistent tokenization: same PII -> same token
result2 = tokenizer.tokenize("Email: john.doe@acme.com", tenant_id="acme")
print(result2.text)  # "Email: [PII_EMAIL_001]" (same token)

# Per-tenant isolation
result3 = tokenizer.tokenize("Email: john.doe@acme.com", tenant_id="other_co")
print(result3.text)  # "Email: [PII_EMAIL_001]" (different tenant state)

# Multi-type tokenization
sensitive = "SSN: 123-45-6789, Card: 4111 1111 1111 1111, IP: 10.0.0.1"
result4 = tokenizer.tokenize(sensitive, tenant_id="acme")
print(result4.text)
# "SSN: [PII_SSN_001], Card: [PII_CREDIT_CARD_001], IP: [PII_IP_ADDRESS_001]"

# GDPR erasure
tokenizer.clear_tenant("acme")
print(tokenizer.get_pii_count("acme"))  # 0
```

---

## See Also

- [Data Classifier](./classification.md) -- Data level classification with PII detection
- [Compliance Engine](./compliance.md) -- Policy enforcement with data level checks
- [GDPR Manager](../enterprise/gdpr.md) -- Full DSAR lifecycle
- [Key Vault](./vault.md) -- API key protection (leak detection)
