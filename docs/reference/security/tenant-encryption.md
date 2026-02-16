# Tenant Key Manager API Reference

## Module: `corteX.security.tenant_encryption`

Per-tenant encryption key management. Derives unique Fernet (AES-256-CBC + HMAC-SHA256) keys per tenant from a single master key using HKDF-SHA256. Supports per-tenant key rotation with transparent re-encryption of existing data.

GDPR compliance: Art. 32 (encryption as a security measure), Art. 25 (per-tenant isolation by design).

## Data Classes

### `TenantKeyInfo`

**Type**: `@dataclass`

Metadata about a tenant's encryption key.

| Attribute | Type | Description |
|-----------|------|-------------|
| `tenant_id` | `str` | Tenant identifier |
| `key_version` | `int` | Current key version (starts at 1) |
| `created_at` | `float` | When the key was first derived |
| `rotated_at` | `Optional[float]` | When the key was last rotated |
| `rotation_count` | `int` | Number of rotations performed |

---

## Classes

### `TenantKeyManager`

Derive, cache, and rotate per-tenant Fernet encryption keys.

#### Constructor

```python
TenantKeyManager(master_key: Optional[bytes] = None)
```

**Parameters**:

- `master_key` (`Optional[bytes]`): 32-byte master key for key derivation. If `None`, a random master key is generated (with a warning log).

**Raises**: `ValueError` if `master_key` is not exactly 32 bytes.

!!! warning "Production Usage"
    Always provide a persistent master key in production. A random key means all derived tenant keys are lost on restart, making previously encrypted data unrecoverable.

#### Methods

##### `get_tenant_key`

```python
def get_tenant_key(self, tenant_id: str) -> bytes
```

Get or derive the Fernet key for a tenant. Keys are cached after first derivation.

**Returns**: `bytes` -- The Fernet-compatible key (URL-safe base64 encoded, 44 bytes).

##### `get_tenant_fernet`

```python
def get_tenant_fernet(self, tenant_id: str) -> Fernet
```

Get a ready-to-use `Fernet` instance for a tenant. Convenience method for direct encryption/decryption.

**Returns**: `Fernet` -- Initialized Fernet object.

##### `encrypt`

```python
def encrypt(self, tenant_id: str, plaintext: bytes) -> bytes
```

Encrypt data with the tenant's current key.

**Parameters**:

- `tenant_id` (`str`): Tenant identifier.
- `plaintext` (`bytes`): Data to encrypt.

**Returns**: `bytes` -- Fernet ciphertext (includes timestamp and HMAC).

##### `decrypt`

```python
def decrypt(self, tenant_id: str, ciphertext: bytes) -> bytes
```

Decrypt data, trying the current key first, then retired keys (newest first). This allows transparent decryption of data encrypted with previous key versions.

**Raises**: `ValueError` if no key (current or retired) can decrypt the data.

##### `rotate_tenant_key`

```python
def rotate_tenant_key(self, tenant_id: str) -> bytes
```

Rotate the encryption key for a tenant. The old key is retired (kept for decryption fallback) and a new key is derived.

**Returns**: `bytes` -- The new Fernet key.

##### `re_encrypt`

```python
def re_encrypt(self, tenant_id: str, ciphertext: bytes) -> bytes
```

Re-encrypt data under the current tenant key. Decrypts with any available key (current or retired), then encrypts with the current key.

##### `get_key_info`

```python
def get_key_info(self, tenant_id: str) -> Optional[TenantKeyInfo]
```

Get metadata about a tenant's current key. Returns `None` if no key has been derived.

##### `list_tenants`

```python
def list_tenants(self) -> List[str]
```

List all tenant IDs that have derived keys.

##### `destroy_tenant_keys`

```python
def destroy_tenant_keys(self, tenant_id: str) -> bool
```

Crypto-shred: destroy all keys for a tenant (GDPR Art. 17). Destroys both the current key and all retired keys. Key bytes are zeroed in memory.

**Returns**: `bool` -- `True` if keys existed and were destroyed.

!!! danger "Irreversible"
    After destroying tenant keys, all data encrypted for that tenant becomes permanently unrecoverable. This is by design for GDPR Art. 17 (right to erasure via crypto-shredding).

##### `destroy_all`

```python
def destroy_all(self) -> int
```

Destroy all cached keys for all tenants. Returns the number of tenants whose keys were destroyed.

---

## Key Derivation Design

| Property | Implementation |
|----------|---------------|
| **Algorithm** | HKDF-SHA256 |
| **Master key** | 32 bytes (user-provided or random) |
| **Salt** | `SHA-256(tenant_id)` |
| **Info** | `corteX.tenant_encryption.v1:{tenant_id}:v{rotation_counter}` |
| **Output** | 32-byte key, base64url-encoded for Fernet |
| **Encryption** | Fernet (AES-256-CBC + HMAC-SHA256) |
| **Rotation** | Old key retired (not deleted) for decryption fallback |
| **Crypto-shred** | Key bytes zeroed in memory on destroy |
| **Cache limit** | 10,000 tenant keys |

---

## Example

```python
import os
from corteX.security.tenant_encryption import TenantKeyManager

# Production: use a persistent master key
master_key = os.urandom(32)  # Store this securely!
manager = TenantKeyManager(master_key=master_key)

# Encrypt data for a tenant
plaintext = b"Sensitive user data for ACME Corp"
ciphertext = manager.encrypt("acme", plaintext)

# Decrypt
decrypted = manager.decrypt("acme", ciphertext)
assert decrypted == plaintext

# Key rotation
old_key = manager.get_tenant_key("acme")
new_key = manager.rotate_tenant_key("acme")
assert old_key != new_key

# Old data still decryptable (retired key fallback)
decrypted = manager.decrypt("acme", ciphertext)
assert decrypted == plaintext

# Re-encrypt under new key
new_ciphertext = manager.re_encrypt("acme", ciphertext)

# Key info
info = manager.get_key_info("acme")
print(f"Version: {info.key_version}, Rotations: {info.rotation_count}")

# List all tenants with keys
tenants = manager.list_tenants()

# GDPR Art. 17: Crypto-shredding
# Destroys ALL keys -- data becomes permanently unrecoverable
manager.destroy_tenant_keys("acme")

# Verify destruction
try:
    manager.decrypt("acme", ciphertext)
except ValueError as e:
    print(f"Expected: {e}")  # Cannot decrypt -- keys destroyed
```

---

## See Also

- [Key Vault](./vault.md) -- API key storage with XOR obfuscation
- [Data Classifier](./classification.md) -- Data level classification
- [GDPR Manager](../enterprise/gdpr.md) -- DSAR lifecycle with erasure
- [Tenant Manager](../tenancy/manager.md) -- Per-tenant configuration
