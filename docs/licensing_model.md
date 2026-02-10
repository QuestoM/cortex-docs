# corteX SDK: Licensing Model & Update Delivery for On-Prem Deployment

> Research date: 2026-02-09
> Context: corteX is an enterprise AI Agent SDK that SaaS developers install to build AI agents in their products. The SDK must work 100% on-prem.

---

## 1. Industry Landscape: How Enterprise SDKs Handle Licensing

### 1.1 The Shift Away from Pure Per-Seat Pricing

As of 2026, the enterprise software industry is actively moving away from pure per-seat licensing. Major vendors are shifting toward consumption-based, usage-based, and hybrid pricing models. Key trends:

- **Datadog** uses per-host pricing ($23-$34/host/month for Enterprise), not per-developer. Their SDK usage is metered via custom metrics (100-200 custom metrics per monitored host).
- **HashiCorp** adopted the Business Source License (BSL) in 2023. Their model allows free internal/on-prem use of tools like Terraform and Vault, but requires a commercial license if the product is embedded in a competitive SaaS offering. This is relevant to corteX since our customers embed our SDK in their products.
- **JetBrains** uses License Vault for on-premises license management -- a floating license server that distributes licenses across an organization without requiring internet access for each developer. Licenses can float between team members.
- **Snyk** charges per-developer seat (starting $25/month) with a Team plan capped at 10 licenses per org, and an Enterprise plan for larger orgs.
- **Thales (Sentinel)** documents that SDK licensing has two distinct license types: (1) a Development License per programmer using the SDK, and (2) a Deployment License for distributing applications containing the SDK's IP.

### 1.2 The Dual-User Problem in SDK Licensing

SDK licensing is fundamentally different from application licensing because SDKs have two users:

1. **The developer** who builds with the SDK (the direct customer)
2. **The end-user** of the application built with the SDK (the indirect user)

corteX must account for both. Our developers are the SaaS companies installing corteX; their end-users are the people interacting with the AI agents those companies build.

---

## 2. Per-Seat Licensing Models

### 2.1 Token-Based Licensing

**How it works:** Each licensed developer receives a cryptographically signed token (JWT or similar) that is embedded in their development environment or CI/CD pipeline. The token encodes:
- Organization ID
- Developer seat count
- Feature entitlements
- Expiration date
- Cryptographic signature (Ed25519 or RSA)

**Pros:**
- Lightweight, no license server required
- Works fully offline after initial issuance
- Easy to rotate and revoke

**Cons:**
- Requires a token refresh mechanism (manual or automated)
- Cannot enforce concurrent seat limits without a license server

**Best for:** Small-to-medium deployments where trust is high.

### 2.2 License Key with Cryptographic Signing

**How it works:** A license key is a signed payload containing entitlement data. The SDK embeds the vendor's public key and validates the license at startup or import time. Keygen.sh (used by Spotify, Sennheiser, Synopsys) is the leading platform for this approach.

**Signing schemes (ranked by recommendation):**
1. **Ed25519** -- Best overall. Fast, small keys, strong security.
2. **ECDSA P256** -- Required for NIST FIPS compliance environments.
3. **RSA-2048 PSS** -- Widely supported fallback.

**License file format options:**
- `base64+ed25519` -- Signed, not encrypted. Tamper-proof but readable.
- `aes-256-gcm+ed25519` -- Signed AND encrypted. Prevents inspection of entitlement data.

**Validation flow:**
1. Developer places license file in a known path (e.g., `~/.cortex/license.key` or env var `CORTEX_LICENSE_KEY`)
2. SDK reads the file at import/init time
3. SDK verifies the signature using the embedded public key
4. SDK checks expiration, seat count, feature flags
5. If valid, SDK initializes normally; if invalid, SDK raises a clear error

**Pros:**
- Works 100% offline with zero network calls
- Tamper-proof via cryptographic signing
- Can encode rich entitlement data (features, limits, expiration)

**Cons:**
- License files must be manually distributed or refreshed
- Revocation requires issuing new keys (no real-time revocation without phone-home)

**Best for:** Air-gapped and high-security on-prem environments.

### 2.3 Hardware Fingerprinting (Node-Locked)

**How it works:** The license is bound to specific machine characteristics:
- Device GUID
- MAC address
- CPU serial number
- HDD/SSD serial number

A SHA256 hash of these identifiers creates a fingerprint. The license server (or file) is bound to that fingerprint.

**Pros:**
- Prevents license sharing across machines
- Strong enforcement even fully offline

**Cons:**
- Painful for developers who change machines, use VMs, or work in containers
- Cloud/Kubernetes environments make hardware fingerprinting unreliable
- Significant support burden

**Best for:** Restricted environments where license sharing is a serious concern (e.g., defense, regulated industries). NOT recommended as the default for corteX.

### 2.4 Floating License Server (On-Prem)

**How it works:** An on-premises license server (similar to JetBrains License Vault or FlexNet) manages a pool of concurrent licenses. When a developer starts the SDK, it checks out a license from the server. When they stop, the license is returned.

**Pros:**
- Efficient use of licenses (10 licenses can serve 20 developers if not all work simultaneously)
- Real-time enforcement of seat limits
- Centralized management for IT admins

**Cons:**
- Requires deploying and maintaining a license server on-prem
- Single point of failure if the server goes down
- Network dependency within the customer's infrastructure

**Best for:** Large enterprise deployments (50+ developers) where license utilization optimization matters.

---

## 3. Update Delivery Mechanisms for On-Prem Customers

### 3.1 Private Package Registry (Recommended Primary Channel)

**How it works:** Host corteX packages on a private PyPI-compatible registry or artifact repository that customers can pull from.

**Options:**
| Registry | Type | Air-Gap Support | Auth |
|---|---|---|---|
| **JFrog Artifactory** | Self-hosted or cloud | Full (mirror mode) | Token, API key, SSO |
| **AWS CodeArtifact** | Cloud | Partial (VPC endpoint) | IAM, token |
| **GCP Artifact Registry** | Cloud | Partial (VPC-SC) | IAM, token |
| **devpi** | Self-hosted | Full | Basic auth, token |
| **Azure Artifacts** | Cloud | Partial | Azure AD |

**Implementation for corteX:**
1. Publish releases to a private PyPI registry (e.g., Artifactory or devpi)
2. Gate access with license-validated API tokens
3. Customer configures `pip install --index-url https://registry.cortex-sdk.com/simple/ cortex-sdk`
4. For air-gapped customers, provide a mirroring guide or pre-built wheel archives

### 3.2 Signed Package Archives (Air-Gapped)

**How it works:** For fully air-gapped environments, distribute signed `.whl` or `.tar.gz` archives via:
- Secure file transfer (SFTP, S3 presigned URLs)
- USB drive (for classified/defense environments)
- Customer portal download (gated by license key)

**Package signing:**
- Sign packages with GPG or Sigstore/cosign
- Include a signature verification script in the SDK
- Customer verifies package integrity before installation: `pip install cortex_sdk-2.1.0-py3-none-any.whl` after running `gpg --verify cortex_sdk-2.1.0-py3-none-any.whl.sig`

### 3.3 Container Image Registry (For Docker/K8s Deployments)

**How it works:** If corteX includes any server-side components (e.g., an orchestration service), distribute as signed Docker images.

- Push to a private container registry (Harbor, Artifactory, ECR)
- Sign images with cosign/Notary
- Customers pull images behind their firewall

### 3.4 Update Notification Mechanism

Since on-prem customers control their update cycle:
1. **In-SDK version check** -- At init time, the SDK can optionally check for newer versions (configurable, off by default for air-gapped)
2. **Email/webhook notifications** -- Notify admins when new versions are available
3. **Changelog API** -- Expose a versioned changelog endpoint that customers can poll
4. **License-file-embedded version hints** -- When issuing a new license file, include the latest SDK version number so admins see it during license refresh

---

## 4. License Enforcement Approaches

### 4.1 Online Validation (Phone-Home)

**How it works:** SDK calls a license validation API on startup or periodically.

**When to use:** Only for customers with internet-connected development environments.

**Implementation:**
- HTTPS POST to `api.cortex-sdk.com/v1/validate` with license key
- Response includes entitlements, latest version info, and a signed validation receipt
- Cache the receipt locally for offline grace period

**Frequency:** Once per day or once per session (not per-function-call -- too noisy).

### 4.2 Offline Validation with Grace Period

**How it works:** The license file contains an expiration date and an offline grace period. The SDK validates the cryptographic signature locally without any network call.

**Grace period logic:**
```
if license.signature_valid AND license.expiry > now():
    # Fully valid -- all features enabled
    allow()
elif license.signature_valid AND license.expiry < now() AND (now() - license.expiry) < grace_period:
    # Expired but within grace period -- warn but allow
    warn("License expired. Renew within {days_remaining} days.")
    allow()
else:
    # Expired beyond grace period or invalid signature
    deny("Invalid or expired license. Contact sales@cortex-sdk.com")
```

**Recommended grace period:** 30 days after expiration. This gives enterprise procurement cycles time to process renewals.

### 4.3 Usage Metering (Async Reporting)

**How it works:** The SDK locally records usage metrics (agent invocations, tool calls, tokens processed) and periodically reports them to a metering endpoint. This supports usage-based pricing tiers.

**On-prem considerations:**
- Store metering data locally in a SQLite file or append-only log
- Provide a CLI tool to export metering reports (`cortex-admin export-usage --format csv`)
- Customer uploads usage reports during license renewal
- For connected environments, auto-submit via HTTPS

### 4.4 Feature-Gated Entitlements

**How it works:** The license encodes which features/tiers are enabled:

```json
{
  "org_id": "acme-corp",
  "plan": "enterprise",
  "seats": 50,
  "features": {
    "multi_agent_orchestration": true,
    "custom_tool_registry": true,
    "advanced_memory": true,
    "audit_logging": true,
    "max_agents_per_app": -1
  },
  "issued_at": "2026-01-15T00:00:00Z",
  "expires_at": "2027-01-15T00:00:00Z",
  "grace_days": 30,
  "signature": "..."
}
```

The SDK checks these flags at runtime and either enables or disables functionality accordingly.

---

## 5. Recommendations for corteX

### 5.1 Licensing Model: Hybrid Token + Signed License File

**Primary mechanism:** Cryptographically signed license files (Ed25519).

**Rationale:**
- Works 100% offline for air-gapped on-prem (hard requirement)
- Rich entitlement encoding (features, seats, expiration)
- No license server infrastructure needed
- Ed25519 is fast, secure, and has small key sizes

**Pricing structure:**
- **Per-developer seat** for the SDK (the developer building with corteX)
- **Per-deployment tier** for production (based on number of deployed agents or applications)
- This dual structure mirrors the SDK dual-user reality and aligns with industry practice (Thales, LEAD Technologies)

**Recommended tiers:**

| Tier | Seats | Features | Price Model |
|---|---|---|---|
| **Starter** | Up to 5 developers | Core agent framework, basic tools | Flat annual fee |
| **Team** | Up to 25 developers | + Multi-agent, custom tools, memory | Per-seat annual |
| **Enterprise** | Unlimited developers | + Audit logging, SSO, priority support, SLA | Custom contract |

### 5.2 Update Delivery: Private Registry + Signed Archives

**Connected customers:**
- Private PyPI registry (Artifactory or self-hosted devpi)
- Access gated by license-derived API token
- Automatic update notifications in SDK init output

**Air-gapped customers:**
- GPG-signed wheel archives downloadable from a customer portal
- SHA256 checksums published alongside every release
- Quarterly release bundles for customers who prefer batched updates

### 5.3 Enforcement Strategy: Trust but Verify

**At SDK init time (every import/startup):**
1. Verify license file signature (Ed25519, ~0.1ms)
2. Check expiration + grace period
3. Validate feature entitlements for the current operation
4. Log license status locally (never phone home without explicit opt-in)

**Periodic (optional, connected only):**
- Weekly heartbeat to validation API (if customer opts in)
- Usage metering upload (if customer opts in for usage-based billing)

**Grace period:** 30 days post-expiration with warnings. After 30 days, SDK enters read-only mode (existing agents continue to run but new agent creation is blocked).

### 5.4 Implementation Roadmap

| Phase | Deliverable | Timeline |
|---|---|---|
| **Phase 1** | License file validation (Ed25519 signing + verification) | Sprint 1-2 |
| **Phase 2** | Feature-gated entitlements in license payload | Sprint 2-3 |
| **Phase 3** | Private PyPI registry setup + gated access | Sprint 3-4 |
| **Phase 4** | Usage metering (local storage + export CLI) | Sprint 4-5 |
| **Phase 5** | Optional phone-home validation + auto-update check | Sprint 5-6 |
| **Phase 6** | Floating license server (for Enterprise tier) | Sprint 7-8 |

### 5.5 Key Technical Decisions

1. **Use Ed25519 for license signing** -- Fast, secure, small keys. Use Keygen.sh or build in-house with PyNaCl/libsodium.
2. **License file location:** Check in order: `CORTEX_LICENSE_KEY` env var > `~/.cortex/license.key` > `./cortex-license.key`
3. **Never block developers silently** -- Always provide clear error messages with remediation steps.
4. **Ship the public key embedded in the SDK** -- The private key stays on our licensing server. Obfuscate the public key in the compiled package to deter casual tampering.
5. **Version the license format** -- Include a `license_version` field so we can evolve the schema without breaking old licenses.

---

## 6. Sources

- [Keygen.sh: Offline License Implementation](https://keygen.sh/docs/choosing-a-licensing-model/offline-licenses/)
- [Keygen.sh: License Key Validation](https://keygen.sh/docs/validating-licenses/)
- [Keygen.sh: Offline Licensing API / Cryptography](https://keygen.sh/docs/api/cryptography/)
- [ModernOnPrem.org: Licensing Models for Modern On-Prem](https://onprem.org/vendor/operational/licensing/)
- [Thales: SDK Licensing](https://cpl.thalesgroup.com/software-monetization/sdk-licensing)
- [Thales: SaaS Pricing Models](https://cpl.thalesgroup.com/software-monetization/saas-pricing-models)
- [LEAD Technologies: SDK Licensing Overview](https://www.leadtools.com/help/sdk/licensing/sdk-licensing.html)
- [Revenera: FlexNet Licensing](https://www.revenera.com/software-monetization/products/software-licensing/flexnet-licensing)
- [LicenseSpring: SDK Documentation](https://docs.licensespring.com/sdk)
- [Datadog Pricing](https://www.datadoghq.com/pricing/)
- [HashiCorp: BSL License FAQ](https://www.hashicorp.com/en/license-faq)
- [JetBrains License Vault](https://www.jetbrains.com/ide-services/license-vault/)
- [Snyk Pricing](https://snyk.io/plans/)
- [Private Python Package Repositories Guide](https://inventivehq.com/blog/private-python-package-repositories-guide)
- [CIO: New Software Pricing Metrics 2026](https://www.cio.com/article/4097012/new-software-pricing-metrics-will-force-cios-to-change-negotiating-tactics.html)
- [SAMexpert: Visual Studio 2026 Licensing Update](https://samexpert.com/visual-studio-2026-licensing-update/)
