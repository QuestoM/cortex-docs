# Batch 6: Security Documentation Audit

**Auditor**: Claude Opus 4.6 (automated)
**Date**: 2026-02-22
**Scope**: 8 security API reference pages vs. actual source code
**Total Gaps Found**: 14

---

## Summary

| # | Page | Source | Status | Gaps |
|---|------|--------|--------|------|
| 1 | vault.md | `corteX/security/vault.py` | [A] | 3 |
| 2 | capabilities.md | `corteX/security/capabilities.py` | [OK] | 0 |
| 3 | attenuation.md | `corteX/security/attenuation.py` | [A] | 1 |
| 4 | classification.md | `corteX/security/classification.py` | [OK] | 0 |
| 5 | compliance.md | `corteX/security/compliance.py` | [A] | 3 |
| 6 | audit-logger.md | `corteX/security/audit_logger.py` + `audit_types.py` | [OK] | 0 |
| 7 | pii-tokenizer.md | `corteX/security/pii_tokenizer.py` | [OK] | 0 |
| 8 | tenant-encryption.md | `corteX/security/tenant_encryption.py` | [A] | 7 |

---

## Detailed Findings

---

### 1. reference/security/vault.md -> corteX/security/vault.py
**Status**: [A] (3 gaps found)

#### WRONG_DESCRIPTION
1. **Encryption method mismatch**: Doc (lines 5, 119, 127-128) describes encryption as "XOR + tenant-derived SHA-256 hash" and warns it is "Not Production Cryptography" with an "upgrade path to `cryptography.fernet.Fernet`". However, the actual code (lines 1-5, 20, 36-40, 59-60, 205-207) uses **Fernet (AES-256-CBC + HMAC-SHA256)** with **PBKDF2-HMAC-SHA256 (600k iterations)** for key derivation. The upgrade has already happened. The XOR obfuscation is completely gone.

   - Doc says: "XOR with tenant-derived SHA-256 pad, then base64 encode"
   - Code does: `Fernet(PBKDF2_derived_key).encrypt()` -- real AES-256-CBC + HMAC-SHA256
   - Doc says: "Pad derivation: SHA-256(tenant_id) expanded via iterative hashing"
   - Code does: `PBKDF2-HMAC-SHA256` with 600k iterations, static salt prefix `corteX.security.vault.v1:`
   - Doc warning "Not Production Cryptography" is now inaccurate -- this IS production cryptography

2. **Security Design table is entirely stale**: The entire "Security Design" table at doc lines 117-126 describes the old XOR-based approach. Every row needs updating:
   - "Obfuscation" row should say "Fernet (AES-256-CBC + HMAC-SHA256)"
   - "Pad derivation" row should say "PBKDF2-HMAC-SHA256, 600k iterations"
   - The `TODO` warning box should be removed since Fernet is now implemented

#### WRONG_DESCRIPTION (See Also)
3. **See Also link text mismatch**: Doc line 220 says "Key Vault -- API key storage with XOR obfuscation" in `tenant-encryption.md` See Also section. Should say "with Fernet encryption" since XOR is gone.

---

### 2. reference/security/capabilities.md -> corteX/security/capabilities.py
**Status**: [OK]

All classes, methods, attributes, parameters, return types, and code examples match the source code exactly.

- `Capability` dataclass: `resource`, `actions` (FrozenSet[str]), `constraints` (Dict), `expires_at` (Optional[float]) -- all match (lines 22-38 of code vs doc lines 15-22).
- `to_dict()` / `from_dict()` -- match (lines 40-59 of code).
- `CapabilitySet.__init__`, `has()`, `get_capabilities()`, `is_expired()`, `count` property -- all match.
- `attenuate()` -- drops non-subset capabilities, match (lines 116-141 of code).
- `for_sub_agent()` -- strips `write`, `delete`, `admin` (line 151 `_disallowed = {"write", "delete", "admin"}`) -- match.
- `to_dict()` / `from_dict()` on CapabilitySet -- match.
- Code examples use valid API. No gaps.

---

### 3. reference/security/attenuation.md -> corteX/security/attenuation.py
**Status**: [A] (1 gap found)

#### WRONG_DESCRIPTION
1. **Risk calculation example comment is wrong**: Doc line 173 shows a manual calculation:
   ```
   # risk ~= 0.35*0.9 + 0.25*0.7 + 0.20*0.5 + 0.20*0.15 = 0.62
   ```
   However, the code at lines 181-183 computes `confidence_risk = 1.0 - confidence`, so with `confidence=0.5`, confidence_risk = `0.5`. The drift_score goes through `_risk_from_drift()` (lines 218-228): for `drift_score=0.3` (which is >= 0.2 and < 0.5), the result is `0.3 * 0.5 = 0.15`. So the doc's computation `0.20*0.5` for confidence and `0.20*0.15` for drift gives `0.35*0.9 + 0.25*0.7 + 0.20*0.5 + 0.20*0.15 = 0.315 + 0.175 + 0.1 + 0.03 = 0.62`. This is actually correct. Let me re-verify...

   Actually the math checks out: 0.315 + 0.175 + 0.100 + 0.030 = 0.620. The comment is correct.

   However, there is still a minor discrepancy: the doc shows `0.20*0.5` for confidence risk, which corresponds to `1.0 - 0.5 = 0.5` -- correct. And `0.20*0.15` for drift, where `0.3 * 0.5 = 0.15` via `_risk_from_drift` -- also correct. So the example calculation is accurate.

   Let me re-examine for other gaps...

   **`for_sub_agent()` docstring discrepancy**: Doc line 148 at `for_sub_agent` code says docstring "Removes ``"write"`` and ``"delete"`` actions" but the actual `_disallowed` set on line 151 includes `"admin"` as well. The code DOES strip `admin`, and the doc on line 119 says "Removes `write`, `delete`, and `admin` actions" and doc line 154 lists `admin` as stripped. So the doc is correct; the code docstring is slightly incomplete but that's a code issue, not a doc issue.

   Actually, re-checking the doc, line 119 states:
   > Removes `write`, `delete`, and `admin` actions from every capability, keeping only `read` and `execute`. Expired capabilities are excluded.

   And the code at line 148 docstring says:
   > Removes ``"write"`` and ``"delete"`` actions from every capability, keeping ``"read"`` and ``"execute"``.

   The code docstring OMITS `admin` but the actual code includes it in `_disallowed`. The **doc** page correctly lists all three. This is a code docstring issue, not a doc page issue. No doc gap here.

   Revised: After thorough re-examination, there is 1 genuine gap:

1. **Drift risk description incomplete**: Doc line 109 says "Drift risk | 0.20 | Goal drift contribution" but doesn't explain the non-linear mapping. The code's `_risk_from_drift()` (lines 218-228) applies a non-linear transform: drift < 0.2 returns 0.0, drift 0.2-0.5 returns `drift * 0.5`, drift >= 0.5 returns `min(1.0, drift * 1.2)`. The doc's risk factor table on line 109 simply says "Goal drift contribution" with weight 0.20, implying a linear pass-through. The example calculation compensates by showing the transformed value (0.15), but the table itself is misleading about how drift feeds into risk. This is a **minor** WRONG_DESCRIPTION.

---

### 4. reference/security/classification.md -> corteX/security/classification.py
**Status**: [OK]

All classes, methods, attributes, and code examples match exactly.

- `DataLevel(IntEnum)`: PUBLIC=0, INTERNAL=1, CONFIDENTIAL=2, RESTRICTED=3 -- match (lines 25-33 of code).
- `ClassificationResult` dataclass: `level`, `reasons`, `pii_detected`, `secrets_detected`, `patterns_matched` -- all match (lines 72-86).
- `DataClassifier.classify()` signature and behavior -- match. Escalation rules (PII -> CONFIDENTIAL, secrets -> RESTRICTED, keyword match) all match code logic (lines 105-175).
- `DataClassifier.enforce()` signature `(data_level, target_model, is_local) -> Tuple[bool, Optional[str]]` -- match (lines 177-227).
- `DataClassifier.escalate()` static method -- match (lines 229-235).
- PII patterns: email, phone_us, ssn, credit_card -- all match `_PII_PATTERNS` dict (lines 39-51).
- Secret patterns: api_key_generic, bearer_token, password_field, aws_key, github_token, openai_key, private_key_header -- all match `_SECRET_PATTERNS` dict (lines 56-64).
- Code examples use valid API. No gaps.

---

### 5. reference/security/compliance.md -> corteX/security/compliance.py
**Status**: [A] (3 gaps found)

#### MISSING_API
1. **`consent_manager` constructor parameter undocumented**: The code's `ComplianceEngine.__init__` (line 64-68) accepts an optional `consent_manager: Optional[Any] = None` parameter. When attached, the engine auto-resolves `has_consent` from live consent records via `_resolve_consent()` (lines 122-147). This feature is entirely undocumented in the doc's constructor section (doc lines 82-89). The doc only shows:
   ```python
   ComplianceEngine(frameworks: Optional[List[ComplianceFramework]] = None)
   ```
   But the actual signature is:
   ```python
   ComplianceEngine(frameworks: Optional[List[ComplianceFramework]] = None, consent_manager: Optional[Any] = None)
   ```

#### MISSING_API
2. **Consent auto-resolution behavior undocumented**: The `pre_check()` method (code lines 77-120) auto-resolves `has_consent` when a `ConsentManager` is attached and context contains `tenant_id`, `user_id`, and `consent_purpose`. This behavior is mentioned in the code docstring but completely absent from the doc page. The doc's `pre_check` section (doc lines 93-110) does not mention consent auto-resolution or the `tenant_id`/`user_id`/`consent_purpose` context keys.

#### MISSING_API
3. **`_enforce_iso27001` method undocumented**: The code has a private `_enforce_iso27001()` method (lines 215-230) that handles ISO 27001 framework enforcement. While the doc lists `ISO27001` as a supported framework (doc line 20), there is no `enforce_iso27001` section like the other three frameworks have (`enforce_gdpr`, `enforce_hipaa`, `enforce_soc2`). The ISO 27001 handler checks for `risk_assessed` context key and generates LOG + optional WARN actions, but this behavior is not documented.

---

### 6. reference/security/audit-logger.md -> corteX/security/audit_logger.py + audit_types.py
**Status**: [OK]

All classes, methods, attributes, and examples match the combined source across `audit_logger.py`, `audit_types.py`, and `audit_storage.py`.

- `AuditEventType` enum: all 16 values match `audit_types.py` lines 21-48.
- `AuditSeverity` enum: all 5 values match `audit_types.py` lines 51-57.
- `AuditEntry` dataclass: all 12 attributes match `audit_types.py` lines 63-93.
- `AuditEntry` methods: `to_dict()`, `to_json()`, `from_dict()`, `from_json()`, `compute_chain_hash()` -- all match.
- `AuditLogger.__init__` signature: `config`, `tenant_id`, `callback`, `max_file_bytes` -- match (lines 30-36).
- Properties: `tenant_id`, `enabled`, `sequence`, `last_hash`, `entry_count` -- all match (lines 66-84).
- `log_event()` async method signature and behavior -- match (lines 86-147).
- `query_logs()` async method with all filter params -- match (lines 149-177).
- `export_logs()` with json/csv formats -- match (lines 179-189).
- `verify_integrity()` and `verify_integrity_detailed()` -- match (lines 191-226).
- `load_from_files()` and `rotate_and_archive()` -- match (lines 228-242).
- `get_stats()` -- match (lines 255-273).
- Hash chain design description -- matches `audit_types.py` `compute_chain_hash` and `hashable_content` (lines 124-145).
- Code examples use valid API including `AuditConfig` import from `corteX.enterprise.config`. No gaps.

---

### 7. reference/security/pii-tokenizer.md -> corteX/security/pii_tokenizer.py
**Status**: [OK]

All classes, methods, attributes, PII patterns, and examples match.

- `TokenizedResult` dataclass: `text`, `token_map`, `pii_count` -- match (lines 67-78).
- `PIITokenizer.__init__(classifier: Optional[DataClassifier] = None)` -- match (line 107).
- `tokenize(text, tenant_id) -> TokenizedResult` -- match (lines 116-157). Behavior: consistent tokenization, priority ordering, empty text handling, ValueError on empty tenant_id -- all match.
- `detokenize(text, tenant_id) -> str` -- match (lines 159-191).
- `clear_tenant(tenant_id) -> None` -- match (lines 193-202).
- `get_token_map(tenant_id) -> Dict[str, str]` -- match (lines 204-210).
- `get_pii_count(tenant_id) -> int` -- match (lines 212-218).
- PII types table: SSN, credit_card, email, phone, ip_address, name, address -- all match `_PII_PRIORITY` list (line 59-61) and `_EXTENDED_PII_PATTERNS` dict (lines 34-56).
- Priority order: ssn > credit_card > email > phone > ip_address > name > address -- matches `_PII_PRIORITY` (line 59-61).
- Thread safety via per-tenant locks and global lock -- matches `_TenantTokenState.lock` and `PIITokenizer._global_lock`.
- Security design table accurate. Code examples valid. No gaps.

---

### 8. reference/security/tenant-encryption.md -> corteX/security/tenant_encryption.py
**Status**: [A] (7 gaps found)

#### WRONG_DESCRIPTION
1. **See Also link text for vault.md**: Doc line 220 says:
   > Key Vault -- API key storage with XOR obfuscation

   But vault.py now uses Fernet (AES-256-CBC + HMAC-SHA256), not XOR obfuscation. Should say "API key storage with Fernet encryption".

#### WRONG_SIGNATURE
2. **`get_tenant_key` return type description incomplete**: Doc line 58 says return is "The Fernet-compatible key (URL-safe base64 encoded, 44 bytes)". The actual key derivation at code line 187 produces `base64.urlsafe_b64encode()` output of a 32-byte key, which is indeed 44 bytes of base64 (32 bytes -> 44 chars with padding). This is technically correct. No gap here after all.

   Let me recount...

#### MISSING_API
2. **`_validate_tenant_id` behavior undocumented**: The code has a `_validate_tenant_id()` static method (lines 210-213) that raises `ValueError` for empty tenant_ids. While the doc mentions `ValueError` for master_key length (line 43), it does not document that ALL public methods (`get_tenant_key`, `get_tenant_fernet`, `rotate_tenant_key`, `re_encrypt`, `encrypt`, `decrypt`, `destroy_tenant_keys`) will raise `ValueError` if `tenant_id` is empty/whitespace. The `encrypt()` and `decrypt()` methods have no Raises documented at all.

#### WRONG_DESCRIPTION
3. **`encrypt` method does not validate tenant_id directly**: Looking at code line 114-116, `encrypt()` calls `get_tenant_fernet()` which calls `_validate_tenant_id()`. The doc lists `encrypt` at line 73-83 with parameters `tenant_id` and `plaintext` but does not mention that an invalid/empty `tenant_id` raises `ValueError`.

#### WRONG_DESCRIPTION
4. **`decrypt` raises `ValueError` with different wording**: Doc line 93 says `Raises: ValueError if no key (current or retired) can decrypt the data`. Code line 205-208 does raise ValueError, but the actual message is "Cannot decrypt data for tenant '{tenant_id}': no key (current or retired) could decrypt the ciphertext." -- this is a match in spirit. However, the doc does not mention that `ValueError` is also raised for empty `tenant_id` (via `_validate_tenant_id` called at line 120).

#### MISSING_API
5. **Cache limit RuntimeError undocumented**: Code line 167-169 raises `RuntimeError` when the cache exceeds 10,000 entries (`_MAX_CACHED_KEYS`). Doc mentions "Cache limit: 10,000 tenant keys" in the design table (line 164) but never documents that exceeding this limit raises `RuntimeError("Tenant key cache full (10000 entries).")`. This could affect `get_tenant_key`, `get_tenant_fernet`, and `rotate_tenant_key`.

#### WRONG_DESCRIPTION
6. **`rotate_tenant_key` does not require existing key**: Doc line 101 says "The old key is retired (kept for decryption fallback) and a new key is derived." The code (lines 87-106) checks `if tenant_id in self._cache` and only retires the old key if it exists. If no key was previously derived, it simply derives a new one. The doc implies rotation always has an "old key" to retire, but calling `rotate_tenant_key` on a tenant that never had `get_tenant_key` called will just derive a key at version 2 with rotation_counter=1. This is not documented.

#### MISSING_API
7. **Thread safety not documented**: The code uses `threading.Lock` (`self._lock` at line 60) for all cache operations. The doc's "Key Derivation Design" table does not mention thread safety, unlike the PII tokenizer doc which explicitly documents it.

---

## Gap Summary by Category

| Category | Count | Details |
|----------|-------|---------|
| WRONG_DESCRIPTION | 6 | vault.py encryption upgrade (2 items), attenuation drift, compliance consent, tenant-encryption old key/vault ref |
| MISSING_API | 6 | compliance consent_manager param, consent auto-resolution, ISO27001 enforcement details, tenant-encryption validate/RuntimeError/thread-safety |
| WRONG_API | 0 | -- |
| WRONG_ATTR | 0 | -- |
| WRONG_SIGNATURE | 0 | -- |
| WRONG_EXAMPLE | 0 | -- |
| FEATURE_REQUEST | 0 | -- |

## Priority Fixes

### HIGH (user-facing confusion)
1. **vault.md encryption description**: The entire security design section describes XOR obfuscation but code uses Fernet/PBKDF2. This is the biggest discrepancy -- a security-conscious user reading the doc would think the vault is insecure when it is actually production-grade.
2. **compliance.md missing `consent_manager`**: Important integration feature completely undocumented.

### MEDIUM (completeness)
3. **compliance.md missing ISO 27001 enforcement details**: Framework is listed but enforcement logic is not documented.
4. **tenant-encryption.md missing error conditions**: RuntimeError on cache full, ValueError on empty tenant_id not documented.

### LOW (minor)
5. **attenuation.md drift risk non-linearity**: Table implies linear pass-through.
6. **tenant-encryption.md thread safety**: Not documented but exists in code.
7. **tenant-encryption.md rotate behavior without prior key**: Edge case not documented.
