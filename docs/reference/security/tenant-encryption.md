# Tenant Encryption API Reference

## Module: `corteX.security.tenant_encryption`

Per-tenant encryption key manager (GAP-G12). Derives unique Fernet (AES-128-CBC + HMAC-SHA256) keys per tenant from a single master key using HKDF-SHA256. HKDF produces a 256-bit key that Fernet splits into 128-bit AES + 128-bit HMAC per the Fernet specification. Supports per-tenant key rotation with transparent re-encryption of existing data.

GDPR: Art. 32 (encryption), Art. 25 (per-tenant isolation).

> **Thread Safety**: All public methods are thread-safe. Internal state is protected by a `threading.Lock`, making `TenantKeyManager` safe for concurrent use across multiple threads.

---

## TenantKeyInfo

**Type**: `@dataclass`

Metadata about a tenant's encryption key.

### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `tenant_id` | `str` | -- | Tenant identifier |
| `key_version` | `int` | `1` | Current key version number |
| `created_at` | `float` | `time.time()` | Key creation timestamp |
| `rotated_at` | `Optional[float]` | `None` | Last rotation timestamp (`None` if never rotated) |
| `rotation_count` | `int` | `0` | Total number of rotations performed |

---

## TenantKeyManager

Derive, cache, and rotate per-tenant Fernet encryption keys. Uses HKDF-SHA256 key derivation from a 32-byte master key.

### Constructor

```python
TenantKeyManager(master_key: Optional[bytes] = None)
```

**Parameters**:

- `master_key` (`Optional[bytes]`): Exactly 32 bytes. If `None`, a random key is generated (with a logged warning). **Raises** `ValueError` if the key is not exactly 32 bytes.

### Tenant ID Validation

All public methods validate that `tenant_id` is a non-empty, non-whitespace string. **Raises** `ValueError` for empty or whitespace-only `tenant_id` values.

### Cache Limit

The internal key cache is limited to 10,000 entries. **Raises** `RuntimeError` when a new key derivation would exceed this limit. This protects against memory exhaustion from unbounded tenant creation.

### Methods

#### `get_tenant_key`

```python
def get_tenant_key(self, tenant_id: str) -> bytes
```

Get or derive the Fernet key bytes for a tenant. Returns cached key if available, otherwise derives and caches a new one.

**Raises**: `ValueError` if `tenant_id` is empty. `RuntimeError` if cache exceeds 10,000 entries.

#### `get_tenant_fernet`

```python
def get_tenant_fernet(self, tenant_id: str) -> Fernet
```

Get a ready-to-use `Fernet` instance for a tenant.

**Raises**: `ValueError` if `tenant_id` is empty. `RuntimeError` if cache exceeds 10,000 entries.

#### `rotate_tenant_key`

```python
def rotate_tenant_key(self, tenant_id: str) -> bytes
```

Rotate the encryption key for a tenant. The old key is retired (kept for decryption fallback) and a new key is derived. Updates the `TenantKeyInfo` with the new version, rotation timestamp, and rotation count.

**Behavior when no prior key exists**: If the tenant has no existing key, the rotation counter is still incremented (starting from 0 to 1) and a new key is derived. This means the first `rotate_tenant_key` call on a new tenant effectively creates the key at version 2, with rotation count 1. The `TenantKeyInfo` is created during derivation if it does not already exist.

**Raises**: `ValueError` if `tenant_id` is empty. `RuntimeError` if cache exceeds 10,000 entries.

#### `encrypt`

```python
def encrypt(self, tenant_id: str, plaintext: bytes) -> bytes
```

Encrypt data with the tenant's current Fernet key.

**Raises**: `ValueError` if `tenant_id` is empty.

#### `decrypt`

```python
def decrypt(self, tenant_id: str, ciphertext: bytes) -> bytes
```

Decrypt data, trying the current key first, then retired keys (newest first). This enables transparent decryption of data encrypted with previous key versions.

**Raises**: `ValueError` if `tenant_id` is empty, or if no key (current or retired) can decrypt the ciphertext.

#### `re_encrypt`

```python
def re_encrypt(self, tenant_id: str, ciphertext: bytes) -> bytes
```

Re-encrypt data under the tenant's current key. Decrypts with any available key (current or retired), then encrypts with the current key. Use after key rotation to migrate old ciphertext.

**Raises**: `ValueError` if `tenant_id` is empty or decryption fails.

#### `get_key_info`

```python
def get_key_info(self, tenant_id: str) -> Optional[TenantKeyInfo]
```

Get metadata about a tenant's current key. Returns `None` if no key has been derived for the tenant.

#### `list_tenants`

```python
def list_tenants(self) -> List[str]
```

List all tenant IDs that have derived keys.

#### `destroy_tenant_keys`

```python
def destroy_tenant_keys(self, tenant_id: str) -> bool
```

Crypto-shred: destroy all keys (current and retired) for a tenant (GDPR Art. 17 -- right to erasure). Returns `True` if keys were found and destroyed, `False` if the tenant had no keys.

**Raises**: `ValueError` if `tenant_id` is empty.

#### `destroy_all`

```python
def destroy_all(self) -> int
```

Destroy all cached keys for all tenants. Returns the number of tenants whose keys were destroyed.

---

## Key Derivation

Keys are derived using HKDF-SHA256:

- **Salt**: SHA-256 hash of the tenant ID (UTF-8 encoded)
- **Info**: `b"corteX.tenant_encryption.v1:{tenant_id}:v{rotation_counter}"`
- **Output**: 32 bytes, base64url-encoded for Fernet compatibility

Each rotation increments the counter, producing a deterministic but unique key per tenant per version.

---

## Example

```python
import os
from corteX.security.tenant_encryption import TenantKeyManager

# Create manager with a stable master key
master_key = os.urandom(32)
manager = TenantKeyManager(master_key=master_key)

# Encrypt data for a tenant
ciphertext = manager.encrypt("acme_corp", b"sensitive customer data")

# Decrypt (uses current key)
plaintext = manager.decrypt("acme_corp", ciphertext)
assert plaintext == b"sensitive customer data"

# Rotate the key
new_key = manager.rotate_tenant_key("acme_corp")

# Old ciphertext still decrypts (retired key fallback)
plaintext = manager.decrypt("acme_corp", ciphertext)
assert plaintext == b"sensitive customer data"

# Re-encrypt under the new key
new_ciphertext = manager.re_encrypt("acme_corp", ciphertext)

# Check key metadata
info = manager.get_key_info("acme_corp")
print(f"Version: {info.key_version}, Rotations: {info.rotation_count}")

# GDPR Art. 17: Crypto-shred all tenant keys
manager.destroy_tenant_keys("acme_corp")

# List remaining tenants
print(manager.list_tenants())
```

---

## See Also

- [Capability Set](./capabilities.md) -- Permission enforcement
- [Data Classifier](./classification.md) -- Data level classification
- [Compliance Engine](./compliance.md) -- Compliance policy enforcement (GDPR, SOC2)
- [Key Vault](./vault.md) -- Secure key storage
- [Tenant Manager](../tenancy/manager.md) -- Multi-tenant management
- [Fernet encryption](https://cryptography.io/en/latest/fernet/) -- Fernet specification (AES-128-CBC + HMAC-SHA256, 256-bit key split into 128-bit AES + 128-bit HMAC)
