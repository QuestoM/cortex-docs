# Vendor Risk Assessment - LLM Sub-Processors

> **Document ID**: VRA-001 | **Version**: 1.0 | **Classification**: CONFIDENTIAL
> **Owner**: Questo Ltd. Security & Compliance | **Last Updated**: 2026-02-23
> **Review Schedule**: Semi-annual (next: 2026-08-23) | **Approved By**: CISO / DPO

---

## 1. Purpose and Scope

This document assesses the risk posture of all third-party LLM providers (sub-processors)
that may receive data through the corteX SDK. As corteX is a **PII processor** (customers
are the data controllers), we must evaluate each vendor that customer data may flow to.

**In scope**: OpenAI, Google (Gemini/Vertex AI), Anthropic, Microsoft Azure (Azure OpenAI),
Amazon Web Services (Bedrock). **Out of scope**: Local/on-prem providers (vLLM, Ollama,
OpenRouter on customer infra) - zero vendor risk as data never leaves customer environment.

---

## 2. Vendor Assessments

### 2.1 OpenAI, Inc.

| Field | Details |
|-------|---------|
| **Legal Entity** | OpenAI, Inc. (Delaware corporation, for-profit) |
| **HQ** | San Francisco, CA, USA |
| **Trust Portal** | [trust.openai.com](https://trust.openai.com/) |
| **Security Page** | [openai.com/security-and-privacy](https://openai.com/security-and-privacy/) |

**Certifications**: SOC 2 Type II (Security, Availability, Confidentiality, Privacy - report
period Jan-Jun 2025), ISO/IEC 27001:2022, ISO/IEC 27017:2015, ISO/IEC 27018:2019,
ISO/IEC 27701:2019.

**Data Processing**: API inputs/outputs **not used for training** (zero retention by default).
DPA at [openai.com/policies/data-processing-addendum](https://openai.com/policies/data-processing-addendum/)
with EU SCCs. Supplier breach SLA: 48 hours.

**Data Residency**: EU residency since Feb 2025 for API (new projects only). In-region GPU
inference in US or Europe since Jan 2026. 10 regions: US, Europe, UK, CA, JP, KR, SG, IN,
AU, UAE. Limitation: existing projects cannot migrate to EU after creation.

**Breach Notification**: "Without undue delay" per DPA. Sub-processor list via Trust Portal.

**Risk Rating: LOW** - Comprehensive ISO stack, EU residency with in-region inference,
zero-retention API, mature DPA with SCCs.

---

### 2.2 Google LLC (Gemini / Vertex AI)

| Field | Details |
|-------|---------|
| **Legal Entity** | Google LLC (Alphabet); Google Ireland Ltd. for EEA/UK |
| **HQ** | Mountain View, CA, USA |
| **Compliance** | [cloud.google.com/security/compliance](https://cloud.google.com/security/compliance) |
| **DPA** | [cloud.google.com/terms/data-processing-addendum](https://cloud.google.com/terms/data-processing-addendum) |

**Certifications**: SOC 1/2/3 Type II, ISO/IEC 27001/27017/27018/27701, FedRAMP High
(Vertex AI), HIPAA BAA, CSA STAR Level 2, PCI DSS.

**Data Processing**: Vertex AI - customer data **not used to train** foundation models.
Gemini API (AI Studio) may use data unless opted out. Model Armor runtime defense.
Breach notification: "Promptly and without undue delay" with incident details.

**Data Residency**: Vertex AI has full regional deployment (europe-west1, europe-west4, etc.).
Google Ireland Ltd. as EEA contracting entity. VPC Service Controls for network-level
data boundaries. Sub-processors at cloud.google.com/terms/subprocessors.

**Risk Rating: LOW** - Industry-leading compliance stack, mature Vertex AI with EU
residency, Google Ireland Ltd. as EEA entity.

---

### 2.3 Anthropic, PBC

| Field | Details |
|-------|---------|
| **Legal Entity** | Anthropic, PBC (Delaware public benefit corporation) |
| **HQ** | San Francisco, CA, USA |
| **EEA Entity** | Anthropic Ireland, Limited |
| **Trust Portal** | [trust.anthropic.com](https://trust.anthropic.com/updates) |
| **Privacy Center** | [privacy.claude.com](https://privacy.claude.com/) |

**Certifications**: SOC 2 Type II, ISO/IEC 27001:2022, ISO/IEC 42001:2023 (AI management),
CSA STAR Level 2, HIPAA configurable.

**Data Processing**: API data not used for training (zero-retention option). DPA auto-
incorporated into Commercial Terms with EU SCCs (Module 2 + Module 3). BYOK encryption
planned H1 2026. Breach notification: written, within 48 hours.

**Data Residency**: **Direct API stores data in US by default** - key limitation. Traffic may
route to US, Europe, Asia, Australia. EU residency available via AWS Bedrock (eu-west-1,
eu-central-1, eu-north-1, etc.) or Google Vertex AI regional endpoints.

**Risk Rating: MEDIUM** - Strong certs including ISO 42001 (AI-specific). Rated MEDIUM
solely due to lack of native EU data residency on direct API. Via Bedrock/Vertex AI,
risk reduces to LOW.

---

### 2.4 Microsoft Corporation (Azure OpenAI Service)

| Field | Details |
|-------|---------|
| **Legal Entity** | Microsoft Corporation |
| **HQ** | Redmond, WA, USA |
| **Trust Center** | [microsoft.com/trust-center](https://www.microsoft.com/trust-center) |
| **Compliance** | [learn.microsoft.com/azure/compliance](https://learn.microsoft.com/en-us/azure/compliance/) |

**Certifications**: SOC 1/2/3 Type II, ISO/IEC 27001/27017/27018/27701, ISO/IEC 42001:2023,
FedRAMP High, HIPAA BAA, PCI DSS, CSA STAR Level 2, C5 (Germany), MTCS Level 3 (Singapore),
ENS High (Spain).

**Data Processing**: Prompts/completions **not used to train** Microsoft/OpenAI models.
Fine-tuning may involve "temporary data relocation" outside geography. Private Link,
RBAC, audit logging, 99.9% uptime SLA. Breach: 72 hours, aligns with GDPR Art. 33.

**Data Residency**: 60+ Azure regions. EU: Switzerland North, West Europe, North Europe,
France Central. AI Studio Auto-Governance (Jan 2026) automates GDPR monitoring. In-country
processing expanding to 15 countries by end 2026. Caution: fine-tuning may relocate data.

**Risk Rating: LOW** - Most comprehensive compliance stack. Broadest data residency.
72-hour breach SLA. Minor concern: fine-tuning relocation caveat.

---

### 2.5 Amazon Web Services, Inc. (Amazon Bedrock)

| Field | Details |
|-------|---------|
| **Legal Entity** | Amazon Web Services, Inc. (Amazon.com subsidiary) |
| **HQ** | Seattle, WA, USA |
| **Security** | [aws.amazon.com/bedrock/security-compliance](https://aws.amazon.com/bedrock/security-compliance/) |
| **Compliance** | [aws.amazon.com/compliance](https://aws.amazon.com/compliance/) |

**Certifications**: SOC 1/2/3 Type II, ISO 9001/27001/27017/27018/27701/22301/20000,
FedRAMP High (GovCloud), HIPAA eligible, CSA STAR Level 2, PCI DSS Level 1, C5 (Germany),
MTCS Level 3 (Singapore).

**Data Processing**: Customer data **not shared with model providers**, not used to improve
base models. Fine-tuning creates private copies. AWS Global DPA applies automatically.
PrivateLink for VPC-to-Bedrock (no public internet). CloudTrail + CloudWatch.

**Data Residency**: EU regions: eu-west-1 (Ireland), eu-central-1 (Frankfurt), eu-north-1
(Stockholm), eu-west-3 (Paris), eu-south-1 (Milan), eu-south-2 (Spain). Geographic
Cross-Region Inference guarantees data stays within EU set. All cross-region encrypted.

**Breach Notification**: "Without undue delay" per AWS DPA. AWS Security Bulletins.

**Risk Rating: LOW** - Broadest ISO set (9 standards), extensive EU coverage with
geographic inference profiles, automatic DPA, PrivateLink, FedRAMP High.

---

## 3. Risk Comparison Matrix

| Vendor | SOC 2 | ISO 27001 | ISO 42001 | GDPR DPA | EU Residency | Breach SLA | FedRAMP | Risk |
|--------|:-----:|:---------:|:---------:|:--------:|:------------:|:----------:|:-------:|:----:|
| **OpenAI** | Yes | Yes | No | Yes (SCCs) | Yes (new projects) | Undue delay | No | **LOW** |
| **Google Vertex** | Yes | Yes | No | Yes (SCCs) | Yes (regional) | Undue delay | High | **LOW** |
| **Anthropic** | Yes | Yes | Yes | Yes (SCCs) | No (US only) | 48 hours | No | **MED** |
| **Azure OpenAI** | Yes | Yes | Yes | Yes (SCCs) | Yes (60+ regions) | 72 hours | High | **LOW** |
| **AWS Bedrock** | Yes | Yes | No | Yes (DPA) | Yes (6+ EU) | Undue delay | High* | **LOW** |

*FedRAMP High in GovCloud only. Anthropic rated MEDIUM for direct API only; via Bedrock
or Vertex AI the hosting provider's residency controls apply, reducing to LOW.

---

## 4. Mitigations Provided by corteX SDK

The corteX SDK provides five architectural controls that reduce vendor risk:

**4.1 PII Tokenization** (`corteX/security/pii_tokenizer.py`)
Redacts personal data (names, emails, phones, addresses, IDs) before any LLM call.
Replaces PII with reversible tokens; re-hydrates on response. Sub-processors never
receive raw personal data.

**4.2 Data Classification Gate** (`corteX/security/classification.py`)
Classifies data as PUBLIC, INTERNAL, CONFIDENTIAL, or RESTRICTED. Blocks CONFIDENTIAL
and RESTRICTED from cloud providers. Only PUBLIC/INTERNAL reaches external LLM APIs.

**4.3 Data Residency Enforcement** (`corteX/enterprise/data_residency.py`)
Per-tenant config restricts which providers and regions are allowed. Enforces geographic
boundaries at the SDK routing layer. Prevents accidental leakage to non-approved regions.

**4.4 On-Premises Alternative** (`LOCAL` type in `corteX/core/llm/router.py`)
Supports any OpenAI-compatible local endpoint (vLLM, Ollama, OpenRouter). Data never
leaves customer infrastructure. **Eliminates all vendor risk** - no sub-processor exists.

**4.5 Audit Logging** (`corteX/observability/audit_stream.py`)
Logs every LLM call with provider, model, timestamp, token count, data classification.
Enables compliance audits and breach forensics. Integrates with enterprise SIEM.

---

## 5. Recommendations

### 5.1 For GDPR-Regulated Workloads (EU/EEA)

| Priority | Recommendation |
|----------|---------------|
| **Best** | LOCAL provider (on-prem) - zero sub-processor risk |
| **Good** | Azure OpenAI (Switzerland North / EU) or AWS Bedrock (eu-central-1) |
| **Good** | Google Vertex AI (europe-west1/4) with VPC Service Controls |
| **Acceptable** | OpenAI API with EU data residency (new projects only) |
| **Caution** | Anthropic direct API - route through Bedrock or Vertex AI instead |

### 5.2 For Maximum Security (Regulated Industries)

| Priority | Recommendation |
|----------|---------------|
| **Best** | LOCAL provider + PII tokenizer + RESTRICTED classification gate |
| **Good** | AWS Bedrock with PrivateLink + Geographic Cross-Region Inference |
| **Good** | Azure OpenAI with Private Link + VNet integration |
| **Acceptable** | Google Vertex AI with VPC Service Controls |

### 5.3 When to Mandate On-Prem

- Processing RESTRICTED/CONFIDENTIAL data that cannot be tokenized
- Strict data sovereignty laws (government, defense, healthcare)
- Security policy prohibits external sub-processors for AI workloads
- Zero-trust environments where no external API calls are permitted

---

## 6. DPA Tracking Table

| Vendor | DPA Available | DPA Signed | Review Date | Notes |
|--------|:------------:|:----------:|:-----------:|-------|
| OpenAI | Yes | Pending | - | [DPA](https://openai.com/policies/data-processing-addendum/). EU SCCs. |
| Google Cloud | Yes | Pending | - | [Cloud DPA](https://cloud.google.com/terms/data-processing-addendum). Auto-applies. |
| Anthropic | Yes | Pending | - | Auto-incorporated into Terms. SCCs Module 2+3. |
| Microsoft | Yes | Pending | - | Part of Product Terms. 72-hour breach SLA. |
| AWS | Yes | Pending | - | Global DPA applies automatically to all services. |

"Pending" = Questo Ltd. must execute before production use. LOCAL requires no DPA.

---

## 7. Review and Approval

| Role | Name | Date | Signature |
|------|------|------|-----------|
| Author | Security & Compliance Team | 2026-02-23 | - |
| Reviewer | CISO | Pending | - |
| Approver | DPO | Pending | - |

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2026-02-23 | Initial vendor risk assessment |

*Review semi-annually or upon: (a) adding a new LLM provider, (b) vendor certification
change, (c) data breach at any vendor, (d) changes to data protection laws.*
