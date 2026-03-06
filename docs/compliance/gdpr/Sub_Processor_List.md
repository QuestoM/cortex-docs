# Sub-Processor List

**Questo Ltd -- corteX AI Agent SDK**

---

| Document Control |  |
|---|---|
| **Document ID** | GDPR-SUB-001 |
| **Version** | 1.0 |
| **Effective Date** | 2026-02-16 |
| **Classification** | Public |
| **Owner** | Questo Ltd, Legal & Compliance |
| **Last Updated** | 2026-02-16 |
| **Publication URL** | https://docs.cortex-ai.com/compliance/gdpr/sub-processors |

---

## Overview

This document lists all sub-processors authorised by Questo Ltd ("Processor") to process personal data on behalf of our customers ("Controllers") in connection with the corteX AI Agent SDK platform (the "Service").

Questo Ltd acts as a data processor under the General Data Protection Regulation (EU) 2016/679 ("GDPR"). Sub-processors are engaged to perform specific processing activities as part of the Service, subject to the terms of the Data Processing Agreement ("DPA") between Questo Ltd and the Controller.

**Key Principle:** The Controller determines which sub-processors are active for its tenant through the Service's ModelPolicy configuration. Personal data is transmitted only to sub-processors that the Controller has explicitly enabled. For on-premise deployments where the Controller hosts local models, no sub-processor engagement occurs for LLM inference.

---

## Current Sub-Processor List

*Effective as of: 2026-02-16*

### LLM Inference Providers

These sub-processors provide large language model inference services. They are engaged only when the Controller enables the corresponding provider for its tenant.

| Sub-Processor | Legal Entity | Purpose | Categories of Personal Data | Processing Location | Transfer Mechanism | DPA Status |
|---|---|---|---|---|---|---|
| **OpenAI** | OpenAI, Inc. (3180 18th Street, San Francisco, CA 94110, USA) | LLM inference -- processing conversation prompts and generating AI agent responses using OpenAI models (e.g., GPT-4, GPT-4o) | Conversation content (prompts and responses), user context provided in prompts | United States | EU-US Data Privacy Framework (DPF) + Standard Contractual Clauses (SCCs) as supplementary safeguard | Active DPA in place. OpenAI is a certified participant in the EU-US Data Privacy Framework. OpenAI's Enterprise API includes a zero-data-retention policy (prompts and completions are not stored or used for model training). |
| **Google (Vertex AI)** | Google LLC (1600 Amphitheatre Parkway, Mountain View, CA 94043, USA) | LLM inference -- processing conversation prompts and generating AI agent responses using Gemini models (e.g., Gemini 3 Pro, Gemini 3 Flash) | Conversation content (prompts and responses), user context provided in prompts | EU (europe-west regions) or US (us-central regions), configurable by Controller via region selection | Adequacy Decision (when EU region selected) / EU-US Data Privacy Framework (DPF) + SCCs (when US region selected) | Google Cloud Data Processing Addendum (CDPA) in place. Google is a certified participant in the EU-US Data Privacy Framework. Vertex AI does not use customer data for model training. |
| **Anthropic** | Anthropic, PBC (548 Market St, PMB 90375, San Francisco, CA 94104, USA) | LLM inference -- processing conversation prompts and generating AI agent responses using Claude models (e.g., Claude 3.5 Sonnet, Claude 3 Opus) | Conversation content (prompts and responses), user context provided in prompts | United States | Standard Contractual Clauses (SCCs) + Transfer Impact Assessment (TIA) | Active DPA in place. Anthropic's API does not use customer data for model training. A Transfer Impact Assessment has been conducted for data transfers to Anthropic. |

### Infrastructure and Ancillary Providers

| Sub-Processor | Legal Entity | Purpose | Categories of Personal Data | Processing Location | Transfer Mechanism | DPA Status |
|---|---|---|---|---|---|---|
| *None at present* | -- | Questo Ltd does not currently engage infrastructure sub-processors for the hosted Service. The SDK is primarily deployed on-premise in the Controller's own infrastructure, or connects directly to the Controller-selected LLM providers listed above. | -- | -- | -- | -- |

**Note on Sub-Sub-Processors:** Each LLM inference provider listed above may use its own infrastructure sub-processors (e.g., cloud hosting providers such as AWS, Azure, or GCP). These sub-sub-processors are governed by the LLM provider's own data processing agreements and are not directly engaged by Questo Ltd. Details of each provider's infrastructure sub-processors are available in their respective sub-processor lists:

- OpenAI: https://openai.com/policies/sub-processors
- Google Cloud: https://cloud.google.com/terms/subprocessors
- Anthropic: https://www.anthropic.com/policies/subprocessors

---

## Sub-Processor Data Flow

```
Controller's Application
    |
    v
corteX SDK (Questo Ltd - Processor)
    |
    +--[if OpenAI enabled]--> OpenAI API (USA)
    |                            Mechanism: DPF + SCCs
    |                            Data: Prompts & responses only
    |                            Retention: Zero (API, no training)
    |
    +--[if Gemini enabled]--> Google Vertex AI (EU or US)
    |                            Mechanism: Adequacy (EU) / DPF + SCCs (US)
    |                            Data: Prompts & responses only
    |                            Retention: Zero (Vertex AI, no training)
    |
    +--[if Claude enabled]--> Anthropic API (USA)
    |                            Mechanism: SCCs + TIA
    |                            Data: Prompts & responses only
    |                            Retention: Zero (API, no training)
    |
    +--[if local model]----> Controller's Infrastructure (on-prem)
                                Mechanism: No transfer (data stays local)
                                Data: Prompts & responses only
                                Retention: Per Controller's own policies
```

---

## Notification Mechanism

### 30-Day Advance Notice

In accordance with Section 9.3 of the Data Processing Agreement, Questo Ltd provides **thirty (30) calendar days** advance written notice before:

- Engaging any new sub-processor;
- Making a material change to an existing sub-processor engagement (including changes to the sub-processor's processing location, the categories of personal data processed, or the purpose of processing); or
- Replacing an existing sub-processor with a different entity.

### How Notifications Are Delivered

Notifications of sub-processor changes are delivered through the following channels:

1. **Email notification** to the privacy contact designated in the Controller's Data Processing Agreement;
2. **Update to this public sub-processor list** at https://docs.cortex-ai.com/compliance/gdpr/sub-processors with a dated changelog entry;
3. **In-platform notification** via the corteX administration dashboard (when applicable).

Controllers may subscribe to sub-processor change notifications by sending a request to privacy@questo.io.

### Notification Content

Each notification includes:

- The identity and legal entity of the proposed sub-processor;
- A description of the processing to be performed;
- The location(s) where processing will occur;
- The applicable transfer mechanism;
- The effective date of the change (no earlier than 30 days from the notification date);
- Any relevant DPA or certification details.

---

## Objection Process

### Right to Object

Controllers have the right to object to any new or changed sub-processor within **fifteen (15) calendar days** of receiving the notification. Objections must be submitted in writing to privacy@questo.io and must include reasonable grounds related to data protection.

### Resolution Process

Upon receipt of a valid objection, Questo Ltd will:

1. **Acknowledge** the objection within two (2) business days;
2. **Engage in good-faith discussion** with the Controller to address concerns;
3. **Attempt to offer an alternative** that avoids the use of the objected-to sub-processor (e.g., configuring an alternative LLM provider, enabling an on-premise deployment option);
4. If no resolution is possible, the Controller may **terminate the Principal Agreement and DPA without penalty**, receiving a pro-rata refund of any prepaid fees for the period following termination.

### Timeline

| Step | Timeline |
|---|---|
| Sub-processor change notification sent | Day 0 |
| Controller objection deadline | Day 15 |
| Questo acknowledgement of objection | Within 2 business days of objection |
| Good-faith discussion period | Days 15-25 |
| Resolution or termination notice | Day 30 |
| Earliest effective date of sub-processor change | Day 30 (if no objection received) |

---

## Update History

| Date | Version | Change Description |
|---|---|---|
| 2026-02-16 | 1.0 | Initial publication. Sub-processors listed: OpenAI, Inc.; Google LLC (Vertex AI); Anthropic, PBC. |

---

## Contact Information

For questions about this sub-processor list, to subscribe to change notifications, or to submit an objection to a sub-processor change:

| | |
|---|---|
| **Email** | privacy@questo.io |
| **Entity** | Questo Ltd |
| **Address** | Israel |
| **DPA Inquiries** | dpa@questo.io |
| **Response Time** | Two (2) business days for acknowledgement; five (5) business days for substantive response |

---

*This document is publicly available and is updated whenever sub-processor changes occur. The authoritative version is maintained at https://docs.cortex-ai.com/compliance/gdpr/sub-processors.*
