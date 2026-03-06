# Vendor Management Policy

**Document ID**: POL-009
**Version**: 1.0
**Effective Date**: 2026-02-16
**Last Reviewed**: 2026-02-16
**Next Review**: 2027-02-16
**Owner**: CTO
**Approved By**: CEO, Questo
**Classification**: INTERNAL

---

## 1. Purpose

This Vendor Management Policy establishes the framework for evaluating, onboarding, monitoring, and managing third-party vendors whose products or services are used by Questo Ltd or integrated with the corteX AI Agent SDK platform. It defines vendor risk classification, due diligence requirements, contractual security obligations, and ongoing monitoring procedures.

Given that the corteX SDK integrates with multiple LLM providers (OpenAI, Google Gemini, Anthropic, and local models), vendor management is critical to ensuring that the security and privacy commitments made to corteX customers are upheld throughout the supply chain. This policy also addresses the unique risks of AI model providers, including data training policies, prompt confidentiality, and model output reliability.

**SOC 2 Mapping**: CC9.2 (Vendor and Business Partner Risk), CC2.3 (Communication with External Parties)

## 2. Scope

This policy applies to:

- **Vendors**: All third-party organizations that provide products, services, or platforms used by Questo Ltd in the development, distribution, operation, or support of the corteX SDK.
- **LLM Providers**: AI model providers whose APIs are integrated with the corteX SDK, including OpenAI, Google (Gemini/Vertex AI), Anthropic (Claude), and any future providers.
- **Infrastructure Providers**: Cloud hosting, CI/CD platforms, DNS providers, CDN providers, and monitoring services.
- **Software Vendors**: Development tools, communication platforms, and business applications.
- **Sub-Processors**: Vendors that process personal data on behalf of Questo or Questo's customers, subject to GDPR requirements.
- **Open-Source Dependencies**: Third-party Python packages included in the corteX SDK dependency tree.

## 3. Definitions

| Term | Definition |
|------|-----------|
| **Vendor** | Any third-party organization providing products or services to Questo Ltd |
| **LLM Provider** | A vendor providing large language model API services |
| **Sub-Processor** | A vendor that processes personal data on behalf of Questo Ltd or its customers |
| **Vendor Risk Classification** | The assigned risk level (Critical, High, Medium, Low) based on the vendor's access to data and operational impact |
| **SOC 2 Report** | Service Organization Control Type 2 report, an independent audit of a service organization's controls |
| **DPA** | Data Processing Agreement; a contractual document defining data processing obligations under GDPR |
| **BYOK** | Bring Your Own Key; the corteX model where tenants provide their own LLM API keys |
| **SLA** | Service Level Agreement; contractual commitments regarding service availability and performance |
| **Vendor Assessment** | The formal evaluation of a vendor's security posture and risk profile |
| **SBOM** | Software Bill of Materials; an inventory of all software components and dependencies |

## 4. Roles and Responsibilities

### 4.1 CTO (Policy Owner)

- Owns this policy and approves vendor onboarding for Critical and High vendors
- Reviews vendor assessments for Critical vendors annually
- Makes vendor risk acceptance decisions within the defined threshold
- Approves exceptions to vendor management requirements

### 4.2 Security Lead

- Executes vendor risk assessments and due diligence procedures
- Maintains the vendor inventory with current risk classifications
- Collects and reviews SOC 2 reports and other security attestations
- Monitors vendor security posture and incident disclosures
- Manages DPAs and security addenda with vendors

### 4.3 Engineering Team

- Reports vendor security concerns to the Security Lead
- Evaluates open-source dependencies for security, license compatibility, and maintenance status before proposing inclusion
- Monitors dependency vulnerability disclosures

### 4.4 CEO

- Approves vendor contracts exceeding the defined financial threshold
- Approves vendor risk acceptance for Critical vendors when risk exceeds CTO threshold

## 5. Policy Statements

### 5.1 Vendor Risk Classification

All vendors are classified based on their access to data and their operational impact on Questo and corteX:

| Classification | Criteria | Assessment Frequency |
|----------------|----------|---------------------|
| **Critical** | Processes customer data, provides core infrastructure, or has direct access to RESTRICTED data | Annually |
| **High** | Provides significant services with access to INTERNAL or CONFIDENTIAL data | Annually |
| **Medium** | Provides supporting services with limited data access | Every 2 years |
| **Low** | Provides commodity services with no data access | Every 3 years |

### 5.2 LLM Provider Risk Assessments

LLM providers represent a unique vendor category for the corteX platform. Each LLM provider is classified as **Critical** and is assessed against the following criteria:

#### 5.2.1 OpenAI

| Assessment Area | Requirement | Verification Method |
|----------------|-------------|-------------------|
| **SOC 2 Report** | Current SOC 2 Type II report | Annual report review |
| **Data Training** | Enterprise API tier with contractual no-training guarantee | DPA review, API tier verification |
| **Data Residency** | Documentation of data processing locations | DPA terms, privacy documentation |
| **Prompt Confidentiality** | Contractual commitment that prompts are not stored beyond processing | DPA terms |
| **Incident Notification** | Commitment to timely breach notification | DPA terms, SLA review |
| **API Security** | API key security, rate limiting, access logging | Technical assessment |
| **Business Continuity** | Published SLA with availability commitments | SLA review |

#### 5.2.2 Google (Gemini / Vertex AI)

| Assessment Area | Requirement | Verification Method |
|----------------|-------------|-------------------|
| **SOC 2 Report** | Current Google Cloud SOC 2 Type II report | Annual report review |
| **Data Training** | Gemini API no-training policy for paid tiers | Terms of service review, DPA |
| **Data Residency** | EU data residency options (Vertex AI) | Configuration verification, DPA |
| **CDPA** | Google Cloud Data Processing Addendum | Signed agreement on file |
| **Incident Notification** | Google Cloud incident notification procedures | DPA terms |
| **Model Deprecation** | Advance notice for model deprecation | Published deprecation policy review |
| **Business Continuity** | Google Cloud SLA | SLA review |

#### 5.2.3 Anthropic (Claude)

| Assessment Area | Requirement | Verification Method |
|----------------|-------------|-------------------|
| **SOC 2 Report** | Current SOC 2 Type II report | Annual report review |
| **Data Training** | Enterprise API no-training commitment | DPA review, usage policy |
| **Data Residency** | Documentation of data processing locations | DPA terms |
| **Prompt Confidentiality** | Contractual prompt confidentiality | DPA terms |
| **Incident Notification** | Breach notification commitment | DPA terms |
| **API Security** | API key management, usage logging | Technical assessment |
| **Business Continuity** | Published availability commitments | SLA review |

#### 5.2.4 Local / On-Premises LLM Providers

For customers using local OpenAI-compatible models (Ollama, vLLM, LMStudio), the vendor risk is transferred to the customer. The corteX SDK documentation provides guidance on securing local model deployments, including network isolation, access controls, and model integrity verification.

### 5.3 Infrastructure and Platform Vendors

| Vendor | Classification | Key Assessment Areas |
|--------|---------------|---------------------|
| **GitHub** | Critical | SOC 2 report, repository security, secret scanning, Actions security, incident notification |
| **Cloud Provider (Azure/AWS/GCP)** | Critical | SOC 2 report, shared responsibility model, encryption, data residency, incident notification |
| **PyPI** | High | Publishing security, package integrity (Sigstore), MFA enforcement, account security |
| **Domain Registrar / DNS** | Medium | Account security, DNSSEC support, MFA |
| **Monitoring Services** | Medium | Data handling, access controls, SOC 2 or equivalent |

### 5.4 Vendor Due Diligence Requirements

#### 5.4.1 Critical Vendors

Before onboarding and annually thereafter:

1. **SOC 2 Report Review**: Obtain and review the vendor's most recent SOC 2 Type II report. Evaluate any exceptions, qualifications, or complementary user entity controls (CUECs).
2. **Data Processing Agreement**: Execute a DPA covering: data types processed, processing purposes, security requirements, breach notification (maximum 72 hours), audit rights, data return and deletion, sub-processor management.
3. **Security Questionnaire**: Complete a vendor security questionnaire covering: encryption practices, access controls, incident response, business continuity, personnel security, and vulnerability management.
4. **Contract Review**: Verify that the contract includes: security obligations, SLA commitments, liability provisions, termination rights, and data ownership clauses.
5. **Financial Stability**: Basic assessment of the vendor's financial viability and business continuity.

#### 5.4.2 High Vendors

Before onboarding and annually thereafter:

1. **SOC 2 Report or Equivalent**: SOC 2, ISO 27001, or equivalent security attestation.
2. **DPA**: Required if the vendor processes personal data.
3. **Contract Review**: Security obligations, SLA, and termination rights.

#### 5.4.3 Medium and Low Vendors

Before onboarding and per the assessment cycle:

1. **Security Questionnaire**: For Medium vendors.
2. **Contract Review**: Standard terms review.
3. **No formal attestation required** for Low vendors, but security terms should be present in the contract.

### 5.5 Open-Source Dependency Management

Open-source Python packages in the corteX SDK dependency tree are managed as a vendor risk category:

1. **Evaluation Criteria**: Before adding a new dependency, evaluate: license compatibility (must be compatible with corteX distribution license), maintenance status (last commit within 12 months), security history (known vulnerabilities), community size and bus factor, and code quality.
2. **Vulnerability Monitoring**: All dependencies are monitored for CVE disclosures through GitHub Dependabot and the CI/CD dependency audit stage (POL-003).
3. **SBOM Generation**: A Software Bill of Materials is generated for each SDK release, listing all direct and transitive dependencies with versions.
4. **Pinning**: Dependencies are version-pinned in the lock file to prevent supply chain attacks from unreviewed updates.
5. **Critical Dependencies**: The `cryptography` library (providing Fernet/AES-128-CBC with PBKDF2-derived 256-bit keys for KeyVault) is classified as a Critical dependency and is assessed with enhanced scrutiny.

### 5.6 SOC 2 Report Collection and Review

1. The Security Lead requests SOC 2 Type II reports from all Critical and High vendors annually.
2. Reports are reviewed for:
   - Scope: Does the report cover the services Questo uses?
   - Opinion: Is the auditor's opinion unqualified?
   - Exceptions: Are there any control exceptions? If so, do they affect Questo's security?
   - CUECs: Are there complementary user entity controls that Questo must implement?
3. Review findings are documented in the vendor assessment record.
4. Material findings are escalated to the CTO and added to the risk register (POL-005).
5. Reports are retained for 3 years.

### 5.7 Ongoing Vendor Monitoring

Between formal assessments, vendors are monitored through:

1. **Security Incident Disclosures**: The Security Lead monitors vendor security advisories and public breach disclosures.
2. **Service Performance**: Track vendor uptime and SLA compliance.
3. **Terms of Service Changes**: Monitor for changes to vendor terms that affect data handling or security.
4. **Regulatory Changes**: Monitor for regulatory actions against vendors.
5. **Financial Changes**: Monitor for significant financial events (acquisition, bankruptcy) affecting Critical vendors.

Material monitoring findings trigger an out-of-cycle vendor reassessment.

### 5.8 Vendor Offboarding

When a vendor relationship is terminated:

1. Revoke all access credentials and API keys associated with the vendor.
2. Request confirmation of data return or deletion per the DPA.
3. Verify that Questo data has been removed from the vendor's systems.
4. Update the vendor inventory to reflect the termination.
5. For LLM providers: update the corteX SDK documentation if a supported provider is removed.
6. For infrastructure vendors: execute the migration plan before termination.

## 6. Procedures

### 6.1 Vendor Onboarding Procedure

1. The requesting party identifies the vendor and the service to be used.
2. The Security Lead classifies the vendor per Section 5.1.
3. The Security Lead executes due diligence per the vendor's classification (Section 5.4).
4. For Critical vendors: the CTO reviews and approves the assessment.
5. Contract negotiation includes security requirements appropriate to the classification.
6. The vendor is added to the vendor inventory with: name, classification, services, contact, contract dates, assessment date, and next assessment date.
7. Access is provisioned per the contract terms and POL-002.

### 6.2 Annual Vendor Assessment Procedure

1. The Security Lead identifies vendors due for assessment per the assessment schedule.
2. Request current SOC 2 reports (or equivalent) from Critical and High vendors.
3. Complete the vendor assessment for each vendor using the criteria for its classification.
4. Document findings in the vendor assessment record.
5. Present Critical vendor assessments to the CTO for review.
6. Update the risk register with any new or changed vendor risks.
7. Schedule the next assessment per the classification cycle.

### 6.3 Vendor Incident Response Procedure

When a vendor discloses a security incident:

1. The Security Lead assesses the impact on Questo and corteX customers.
2. For LLM provider incidents: assess whether customer prompts, API keys, or outputs were affected.
3. Coordinate with the vendor for detailed impact information.
4. Activate the Incident Response Plan (POL-004) if Questo or customer data is affected.
5. For BYOK scenarios: notify affected tenants and recommend API key rotation.
6. Document the vendor incident in both the incident log and the vendor assessment record.
7. Update the vendor's risk classification if warranted.

## 7. Exceptions

1. New Critical vendor engagements without a completed SOC 2 review may be approved by the CTO for a maximum of 90 days, with a compensating control plan documented.
2. Vendors that do not produce SOC 2 reports must provide an equivalent security attestation (ISO 27001, penetration test report, or completed security questionnaire) approved by the CTO.
3. Open-source dependencies that do not meet all evaluation criteria may be approved by the CTO with documented justification and a plan to mitigate identified risks.
4. All exceptions are documented in the Exception Register per POL-001 Section 7.

## 8. Enforcement

- Use of unapproved vendors for Questo business is a policy violation.
- Bypassing the vendor onboarding process is a serious policy violation.
- The Security Lead is accountable for maintaining the vendor assessment schedule.
- Failure to collect required SOC 2 reports is a control deficiency for SOC 2 purposes.
- Adding dependencies to the corteX SDK without the required evaluation (POL-003 Section 6.3) is a policy violation subject to the change management enforcement provisions.

## 9. Related Documents

| Document ID | Document Name |
|-------------|---------------|
| POL-001 | Information Security Policy |
| POL-003 | Change Management Policy |
| POL-004 | Incident Response Plan |
| POL-005 | Risk Assessment and Mitigation Policy |
| POL-006 | Business Continuity Plan |
| POL-008 | Data Classification Policy |
| POL-011 | Encryption Policy |

## 10. Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-16 | CTO | Initial policy creation |
