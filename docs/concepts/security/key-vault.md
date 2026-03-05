# Key Vault

The Key Vault provides per-tenant, in-memory API key storage with
Fernet encryption. Keys are encrypted at rest using AES-128-CBC with
HMAC-SHA256 verification, derived from the tenant ID through PBKDF2.
Keys never touch disk or log output.

## How It Works

When you create a `KeyVault` for a tenant, the vault derives a
Fernet-compatible encryption key using PBKDF2-HMAC-SHA256 with
600,000 iterations (OWASP 2024 minimum). The tenant ID itself serves
as both the password and part of the salt, combined with a static
prefix (`corteX.security.vault.v1:`) to ensure domain separation.

Every API key stored through `vault.store()` is immediately encrypted
into a Fernet token. Retrieval via `vault.retrieve()` decrypts on
demand - the plaintext exists in memory only for the duration of the
call. There is no environment-variable fallback; if a key is missing,
the vault raises a `ValueError` with a clear message.

## Key Features

- **Fernet encryption** - AES-128-CBC + HMAC-SHA256 per the Fernet spec
- **PBKDF2 key derivation** - 600k iterations from tenant ID, SOC 2 CC6.1 compliant
- **Atomic key rotation** - old key bytes are zeroed before new key is stored
- **Leak detection** - `detect_leak(text)` scans output for any stored key
- **Key fingerprinting** - stores first-4/last-4 character fingerprints for safe logging
- **GDPR erasure** - `destroy()` zeros all keys and clears fingerprints
- **No env fallback** - explicit storage only, preventing accidental credential leaks
- **Max key length** - 4,096 bytes to prevent abuse

## Integration

The Key Vault is the single source of truth for API credentials across
corteX. The LLM providers (OpenAI, Gemini, Anthropic, Local) retrieve
keys from the vault at call time rather than reading environment
variables. The Risk Attenuator and Compliance Engine can inspect vault
state to enforce security policies.

When multi-tenancy is enabled, each tenant gets an isolated `KeyVault`
instance with its own derived encryption key, ensuring complete
cryptographic separation between tenants.

## Usage Example

```python
from corteX.security.vault import KeyVault

# Create a vault scoped to a tenant
vault = KeyVault("acme-corp")

# Store API keys (encrypted immediately)
vault.store("openai", "sk-abc123...")
vault.store("gemini", "AIza...")

# Retrieve (decrypts on demand)
key = vault.retrieve("openai")

# Rotate a key atomically (old bytes zeroed first)
vault.rotate("openai", "sk-new456...")

# Scan LLM output for leaked keys
leaked_provider = vault.detect_leak(llm_response_text)
if leaked_provider:
    print(f"Key leak detected for provider: {leaked_provider}")

# List stored providers
providers = vault.list_providers()  # ["openai", "gemini"]

# Remove a single key
vault.remove("gemini")

# GDPR erasure - destroy all keys
vault.destroy()
```

## See Also

- [Capability-Based Security](capabilities.md) - permission model that controls vault access
- [Data Classification](data-classification.md) - classifies data flowing through the system
- [Compliance Engine](compliance-engine.md) - enforces SOC 2 and regulatory requirements
