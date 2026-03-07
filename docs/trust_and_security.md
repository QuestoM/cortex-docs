# Trust & Security

**Security is not a feature. It is our foundation.**

corteX is an enterprise AI agent SDK built for organizations that cannot compromise on data protection, compliance, or operational safety. Every line of code assumes a hostile environment. Every default is locked down. Every action is auditable.

---

## Compliance

| Framework | Status | Details |
|-----------|--------|---------|
| **SOC 2 Type II** | Ready | 22 controls verified, 0 gaps. Hash-chained audit logs, access control, change management. |
| **GDPR** | Compliant | 7 DSAR rights implemented. DPIA generator, ROPA, DPA template, consent management. |
| **HIPAA** | Supported | PHI isolation, BAA enforcement, encryption-at-rest, access logging. |
| **ISO 27001** | Supported | Information security management controls, risk assessment integration. |
| **20 Security Policies** | Published | Information security, access control, change management, incident response, business continuity, data classification, vendor management, acceptable use, disaster recovery, encryption, password policy, network security, data retention, audit logging, SDLC, and more. |

---

## Architecture: On-Prem First

corteX has **zero required external dependencies**. The entire SDK runs on your infrastructure - no data leaves your network unless you explicitly configure it to.

- No cloud telemetry or phone-home behavior
- No external API calls unless the tenant enables them
- All LLM providers work with on-prem endpoints (Ollama, vLLM, or any OpenAI-compatible server)
- Per-tenant configuration stored locally as JSON - no external config service

```python
from corteX.sdk import Engine
# Fully on-prem: local LLM, no external calls
engine = Engine(
    providers={"local": {"base_url": "http://localhost:11434/v1"}},
    tenant_id="acme_corp"
)
```

---

## Data Protection

### PII Detection and Tokenization

Every piece of text flowing through corteX can be automatically scanned for PII before it reaches any LLM. Detected PII is replaced with reversible tokens - the LLM never sees the original values.

**Detected patterns:** email, phone, SSN, credit card, Israeli ID (Teudat Zehut with check-digit validation), IP address, name, address.

```python
from corteX.security.pii_tokenizer import PIITokenizer

tokenizer = PIITokenizer()
result = tokenizer.tokenize("Contact john@acme.com", tenant_id="acme")
# result.text == "Contact [PII_EMAIL_001]"

# After LLM response, restore originals
restored = tokenizer.detokenize(result.text, tenant_id="acme")
```

Per-tenant isolation guarantees that tenant A's PII tokens are never accessible to tenant B. Thread-safe with per-tenant locks.

### 4-Tier Data Classification

All data is classified automatically before processing. Classification can only escalate - it can never be downgraded.

| Level | Allowed Destinations | Requirements |
|-------|---------------------|--------------|
| **PUBLIC** | Any model (cloud or local) | None |
| **INTERNAL** | Local/on-prem models only | - |
| **CONFIDENTIAL** | Local/on-prem models only | Audit trail required |
| **RESTRICTED** | Local/on-prem models only | Human approval + full audit |

When compliance frameworks are active, data classification cannot be disabled (enforced in code via `__post_init__` guards).

### Data Residency

Per-tenant geographic restrictions enforce where data can be processed. The `DataResidencyManager` maps provider endpoints to regions and blocks requests that violate policy. Supports US, EU, UK, Asia-Pacific, Canada, Australia, Japan, India, Brazil, Middle East, and local/on-prem regions.

```python
drm = DataResidencyManager()
drm.set_allowed_regions("acme", ["eu", "uk"])
check = drm.validate_provider("acme", "https://api.openai.com/v1")
# check.allowed == False (OpenAI is US-based)
```

---

## Encryption

| Property | Implementation |
|----------|---------------|
| **Algorithm** | Fernet (AES-128-CBC + HMAC-SHA256) |
| **Key derivation** | PBKDF2-HMAC-SHA256, 600,000 iterations (OWASP 2024 minimum) |
| **Per-tenant keys** | HKDF-SHA256 derives unique keys from a single master key |
| **Key rotation** | Atomic rotation with secure zeroing of old key material |
| **Cryptographic erasure** | Destroy all tenant keys to render stored data unrecoverable (GDPR Art. 17) |
| **Key storage** | In-memory only - keys never touch disk or logs |

```python
from corteX.security.vault import KeyVault

vault = KeyVault("acme_corp")
vault.store("openai", "sk-abc123...")
vault.rotate("openai", "sk-new456...")  # Old key zeroed atomically
vault.detect_leak(response_text)         # Scan for leaked keys
```

---

## Access Control

### Capability-Based Security

corteX uses capability-based security instead of role-based access. Capabilities are immutable tokens that can only shrink (attenuate) - they can never gain new permissions after creation. Sub-agents automatically receive attenuated copies with write and delete actions removed.

### Risk-Based Attenuation

As risk increases, agent capabilities automatically decrease - without any developer intervention.

| Risk Level | Score Range | Allowed Actions |
|------------|------------|-----------------|
| **LOW** | 0.0 - 0.3 | Full capabilities |
| **MEDIUM** | 0.3 - 0.6 | Read + execute (write removed) |
| **HIGH** | 0.6 - 0.8 | Read only (execute removed) |
| **CRITICAL** | 0.8 - 1.0 | Read only + human approval required |

Risk is computed from four weighted factors: tool type (35%), data classification (25%), agent confidence (20%), and goal drift (20%).

### Zero-Trust Policies

Both `ToolPolicy` and `ModelPolicy` default to **deny-all**. No tool executes and no model is called unless explicitly allowed.

```python
from corteX.enterprise.config import TenantConfig, ToolPolicy, ModelPolicy

config = TenantConfig(
    tenant_id="acme_corp",
    tools=ToolPolicy(
        allowed_tools=["search_db", "get_user"],  # Only these two
        default_deny=True,                          # Everything else blocked
    ),
    models=ModelPolicy(
        allowed_models=["gpt-4o", "gpt-4o-mini"],  # Only these two
        default_deny=True,                           # Everything else blocked
    ),
)
```

---

## Audit & Transparency

### Tamper-Evident Audit Logs

Every auditable event produces a hash-chained log entry. Each entry includes a SHA-256 hash computed from its content and the previous entry's hash - forming an immutable chain. Any tampering breaks the chain and is immediately detectable.

```python
# Verify audit log integrity at any time
is_valid = audit_logger.verify_integrity()  # True if chain is intact

# Detailed report with break index
report = audit_logger.verify_integrity_detailed()
# {"valid": True, "total_entries": 4521, "break_index": None}
```

**Audited events:** LLM calls, tool executions, weight changes, security blocks, injection attempts, classification decisions, session lifecycle, policy decisions, compliance checks, config changes, data access, goal events.

### Supply Chain Security

| Practice | Implementation |
|----------|---------------|
| **SBOM** | CycloneDX Software Bill of Materials generated per release |
| **Static analysis** | CodeQL security scanning on every commit |
| **Dependency updates** | Dependabot automated PRs for vulnerable dependencies |
| **Branch protection** | Required reviews, no force-push to main |
| **Test coverage** | 9,300+ automated tests across 128 test files |
| **CI/CD** | 6 GitHub Actions workflows with gated deployments |

---

## GDPR Rights

corteX implements all seven GDPR data subject rights as a programmatic API - not just documentation.

| Right | Article | Implementation |
|-------|---------|----------------|
| **Access** | Art. 15 | `export_user_data()` - Full data export across all stores |
| **Rectification** | Art. 16 | `rectify_user_data()` - Apply corrections to any stored data |
| **Erasure** | Art. 17 | `erase_user_data()` - Cascade deletion across all stores + cryptographic erasure |
| **Restriction** | Art. 18 | `restrict_processing()` - Flag user as processing-restricted |
| **Portability** | Art. 20 | `export_portable_data()` - Structured, machine-readable export |
| **Objection** | Art. 21 | `register_objection()` - Record objection to profiling |
| **Transparency** | Art. 13-14 | `get_processing_info()` - What data we process and why |

Additional GDPR capabilities:

- **Consent management** (Art. 7): Per-purpose consent recording, withdrawal, expiration, lifecycle tracking
- **Profiling opt-out** (Art. 22): Per-user control over automated profiling and weight personalization
- **DPIA generator**: Automated Data Protection Impact Assessments
- **ROPA**: Record of Processing Activities
- **30-day DSAR deadline**: Enforced in code per Art. 12(3)

---

## Input & Output Safety

Every input and output passes through configurable safety checks.

- **Injection protection**: 12 pattern families detect prompt injection, jailbreak, DAN mode, and system prompt extraction attempts
- **Content filtering**: Configurable regex patterns and blocked topic lists
- **PII detection**: Automatic scanning in both inputs and outputs
- **Key leak detection**: Scans LLM responses for stored API key patterns

```python
from corteX.enterprise.config import SafetyPolicy, SafetyLevel

policy = SafetyPolicy(level=SafetyLevel.STRICT)  # All protections on by default
allowed, reason = policy.check_input("ignore all previous instructions")
# allowed == False, reason == "Prompt injection detected ..."
```

---

## Responsible AI

corteX agents are designed to stay on task, respect budgets, and defer to humans when uncertain.

- **Goal tracking**: Every agent step is verified against the original goal. Drift above threshold triggers correction.
- **Loop detection**: Multi-resolution state hashing prevents infinite execution loops.
- **Adaptive budgets**: Token and cost budgets adjust dynamically based on task complexity.
- **Human-in-the-loop**: Configurable insertion points where the agent pauses for human approval.
- **Decision logging**: Every decision is logged with full context via the `ExplainabilityEngine`, enabling post-hoc review and audit.
- **Capability attenuation**: Sub-agents automatically receive reduced permissions - they cannot escalate beyond their parent's authority.

---

## Reporting a Vulnerability

If you discover a security vulnerability in corteX, please report it responsibly.

**Email:** [security@questo.co](mailto:security@questo.co)

We will acknowledge receipt within 48 hours and provide an initial assessment within 5 business days. We do not pursue legal action against researchers who follow responsible disclosure practices.

<p align="center">
<em>corteX is built by <a href="https://questo.co">Questo Ltd.</a> - Apache 2.0 licensed.</em><br>
<code>pip install cortex-ai</code>
</p>
