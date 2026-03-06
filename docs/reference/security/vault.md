# Key Vault API Reference

## Module: `corteX.security.vault`

Per-tenant in-memory API key store. Keys are encrypted with Fernet (AES-128-CBC + HMAC-SHA256) using a PBKDF2-derived 256-bit master key from the tenant_id (600,000 iterations). PBKDF2 derives a 256-bit key, but the Fernet specification uses AES-128-CBC for the symmetric cipher (the 256-bit key is split into a 128-bit AES key and a 128-bit HMAC key). This provides SOC 2 CC6.1 compliant encryption-at-rest for in-memory key storage. Keys never touch disk or logs. No `os.environ` fallback - `ValueError` if a key is not found. Atomic rotation zeros the old key before storing the new one.

## Classes

### `KeyVault`

In-memory encrypted API key store, scoped to a single tenant.

#### Constructor

```python
KeyVault(tenant_id: str)
```

**Parameters**:

- `tenant_id` (`str`): Non-empty tenant identifier. Used to derive the Fernet encryption key via PBKDF2.

**Raises**: `ValueError` if `tenant_id` is empty.

#### Methods

##### `store`

```python
def store(self, provider: str, api_key: str) -> None
```

Encrypt and store an API key for a provider.

**Parameters**:

- `provider` (`str`): Provider identifier (e.g., `"openai"`, `"gemini"`).
- `api_key` (`str`): The raw API key string.

**Raises**: `ValueError` if `provider` or `api_key` is empty, or key exceeds 4096 bytes.

##### `retrieve`

```python
def retrieve(self, provider: str) -> str
```

Decrypt and return the API key for a provider. corteX does NOT fall back to environment variables.

**Parameters**:

- `provider` (`str`): Provider identifier.

**Returns**: `str` -- The decrypted API key.

**Raises**: `ValueError` if no key is stored for the provider.

##### `rotate`

```python
def rotate(self, provider: str, new_key: str) -> None
```

Atomically rotate the key for a provider. The old key bytes are zeroed before the new key is written.

**Parameters**:

- `provider` (`str`): Provider to rotate.
- `new_key` (`str`): New API key.

**Raises**: `ValueError` if no existing key exists for the provider, or if `new_key` is invalid.

##### `detect_leak`

```python
def detect_leak(self, text: str) -> Optional[str]
```

Scan text for any stored API key pattern. Compares against the full decrypted key (short-lived in memory).

**Parameters**:

- `text` (`str`): Text to scan (e.g., LLM output).

**Returns**: `Optional[str]` -- Provider name if a leak is detected, `None` otherwise.

##### `list_providers`

```python
def list_providers(self) -> List[str]
```

Return the list of providers that have stored keys.

##### `remove`

```python
def remove(self, provider: str) -> None
```

Remove and zero the key for a provider.

**Raises**: `ValueError` if no key exists.

##### `destroy`

```python
def destroy(self) -> None
```

Destroy all keys in this vault. GDPR erasure support.

---

## Security Design

| Property | Implementation |
|----------|---------------|
| **Encryption** | Fernet (AES-128-CBC + HMAC-SHA256) via `cryptography` library, with PBKDF2-derived 256-bit master key |
| **Key derivation** | PBKDF2-HMAC-SHA256 with 600,000 iterations (OWASP 2024 minimum) |
| **Salt** | Static prefix `corteX.security.vault.v1:` + tenant_id (per-tenant uniqueness) |
| **Key fingerprint** | `first_4_chars...last_4_chars` for safe logging |
| **Leak detection** | Full key comparison against output text (short-lived decryption) |
| **Rotation** | Old key zeroed before new key stored |
| **No env fallback** | `ValueError` if key not found (no `os.environ`) |
| **Max key size** | 4096 bytes |

!!! success "Production-Grade Cryptography"
    The vault uses Fernet encryption (AES-128-CBC + HMAC-SHA256) with a PBKDF2-derived 256-bit master key at 600,000 iterations. Fernet splits the 256-bit derived key into a 128-bit AES encryption key and a 128-bit HMAC signing key per the Fernet specification. This meets SOC 2 CC6.1 requirements for encryption-at-rest and follows OWASP 2024 guidelines for key derivation.

---

## Example

```python
from corteX.security.vault import KeyVault

vault = KeyVault("acme")

# Store keys
vault.store("openai", "sk-abc123def456...")
vault.store("gemini", "AIza...")

# Retrieve
key = vault.retrieve("openai")

# Rotate
vault.rotate("openai", "sk-new789xyz...")

# Leak detection
leaked = vault.detect_leak("Response contained sk-new789xyz...")
if leaked:
    print(f"LEAK DETECTED: {leaked}")  # "openai"

# List providers
providers = vault.list_providers()  # ["openai", "gemini"]

# GDPR erasure
vault.destroy()
```

---

## See Also

- [Tenant Manager](../tenancy/manager.md) -- Creates and manages per-tenant vaults
- [Capability Set](./capabilities.md) -- Permission control for key access
- [Data Classifier](./classification.md) -- Classifies key-containing text as RESTRICTED
