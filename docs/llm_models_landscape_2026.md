# LLM Models Landscape -- February 2026

## Comprehensive Research Report for corteX AI Agent SDK

Research date: 2026-02-14

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Provider-by-Provider Analysis](#provider-by-provider-analysis)
3. [Master Pricing Comparison Table](#master-pricing-comparison-table)
4. [On-Premises / Self-Hosted Options](#on-premises--self-hosted-options)
5. [Agent-Specific Model Capabilities](#agent-specific-model-capabilities)
6. [Integration Recommendations for corteX](#integration-recommendations-for-cortex)
7. [Sources](#sources)

---

## Executive Summary

The LLM landscape in February 2026 is defined by several key trends:

1. **Context windows have reached 1M-10M tokens** across most major providers, eliminating context as a primary differentiator
2. **Pricing has dropped dramatically** -- GPT-5 Nano at $0.05/1M input tokens represents a 100x reduction from GPT-4 pricing two years ago
3. **Mixture-of-Experts (MoE) architecture dominates** -- Llama 4, Mistral 3, Qwen 3, and DeepSeek all use MoE for efficiency
4. **Agentic capabilities are the new battleground** -- tool calling, multi-step planning, and self-correction are now standard benchmarks
5. **Open-source models have closed the gap** -- GLM-4.7, Qwen3, and DeepSeek V3.2 compete with proprietary models on agent tasks
6. **Model routing and cascading** are essential for cost optimization -- enterprises using intelligent routing report 70-85% cost reductions

For corteX specifically: the SDK should support **at minimum** Google Gemini 3, Anthropic Claude (Opus/Sonnet/Haiku), OpenAI GPT-5.x, and open-source models via vLLM/Ollama. The provider-agnostic architecture is now a critical competitive advantage given the rapid pace of model releases.

---

## Provider-by-Provider Analysis

### 1. Google Gemini

Google's Gemini 3 family represents their most capable models to date, with strong agentic capabilities and the best price/performance ratio for Flash-tier models.

#### Current Models

| Model | Released | Parameters | Context | Max Output | Status |
|-------|----------|-----------|---------|------------|--------|
| `gemini-3-pro-preview` | Nov 2025 | Undisclosed | 1M tokens | 64K tokens | Preview |
| `gemini-3-flash-preview` | Dec 2025 | Undisclosed | 1M tokens | 64K tokens | Preview |
| `gemini-3-pro-image-preview` | Nov 2025 | Undisclosed | 1M tokens | 64K tokens | Preview |
| `gemini-2.5-pro` | 2025 | Undisclosed | 1M tokens | 64K tokens | GA (deprecated Jun 17, 2026) |
| `gemini-2.5-flash` | 2025 | Undisclosed | 1M tokens | 64K tokens | GA (deprecated Jun 17, 2026) |
| `gemini-2.5-flash-lite` | 2025 | Undisclosed | 1M tokens | 64K tokens | GA |

#### Pricing (per 1M tokens)

| Model | Input | Output | Cached Input | Audio Input |
|-------|-------|--------|-------------|-------------|
| Gemini 3 Pro | $2.00 | $12.00 | 90% discount | $2.00 |
| Gemini 3 Flash | $0.50 | $3.00 | 90% discount | $1.00 |
| Gemini 2.5 Pro | $1.25 | $10.00 | 90% discount | $1.25 |
| Gemini 2.5 Flash | $0.15 | $0.60 | 90% discount | $0.40 |

#### Key Strengths
- **Best price/performance for Flash models** -- $0.50/1M input is extremely competitive
- **1M token context across all models** -- industry-leading for the price tier
- **Native multimodal** -- text, images, audio, video as first-class inputs
- **Strong agentic coding** -- Gemini 3 Flash scores 78% on SWE-bench Verified
- **Context caching** -- up to 90% cost reduction for repeated context
- **Free tier available** for development and testing
- **91.9% on GPQA Diamond** (Gemini 3 Pro) -- best reasoning benchmark score

#### Key Weaknesses
- Preview models may have stability issues
- Gemini 2.5 models deprecated June 17, 2026 -- migration required
- Rate limits can be restrictive on lower tiers (Tier-1: 25 RPM)

#### corteX Status
Already integrated. Primary provider for orchestrator (3 Pro) and worker (3 Flash) roles.

---

### 2. Anthropic Claude

Anthropic's Claude family excels at complex reasoning, code understanding, and agentic workflows. Claude Opus 4.6 is the newest and most capable model.

#### Current Models

| Model | Released | Context | Max Output | Status |
|-------|----------|---------|------------|--------|
| `claude-opus-4-6` | Feb 5, 2026 | 1M tokens (beta) / 200K | 32K tokens | GA |
| `claude-opus-4-5` | 2025 | 200K tokens | 32K tokens | GA |
| `claude-sonnet-4-5` | 2025 | 1M tokens | 32K tokens | GA |
| `claude-sonnet-4` | 2025 | 200K tokens | 32K tokens | GA |
| `claude-haiku-4-5` | Oct 2025 | 200K tokens | 32K tokens | GA |

#### Pricing (per 1M tokens)

| Model | Input | Output | Long Context Input (>200K) | Long Context Output (>200K) |
|-------|-------|--------|---------------------------|----------------------------|
| Claude Opus 4.6 | $5.00 | $25.00 | $10.00 | $37.50 |
| Claude Opus 4.5 | $5.00 | $25.00 | N/A | N/A |
| Claude Sonnet 4.5 | $3.00 | $15.00 | $3.00 (1M included) | $15.00 |
| Claude Sonnet 4 | $3.00 | $15.00 | N/A | N/A |
| Claude Haiku 4.5 | $1.00 | $5.00 | N/A (200K max) | N/A |

#### Key Strengths
- **Best-in-class agentic performance** -- Opus 4.6 leads on Terminal-Bench 2.0 (65.4%), OSWorld (72.7%), BrowseComp (84.0%)
- **Exceptional long-context retrieval** -- 93% on MRCR v2 at 256K (vs. 10.8% for Sonnet 4.5)
- **Outstanding code understanding** -- excels at debugging, architectural reasoning, and multi-file analysis
- **Strong safety and alignment** -- best-in-class for enterprise compliance
- **Prompt caching** -- up to 90% cost savings
- **Batch processing** -- 50% cost savings for non-real-time workloads
- **Haiku 4.5** -- excellent speed (109.5 tokens/sec) at low cost for routing/triage

#### Key Weaknesses
- Most expensive provider for flagship models ($5/$25 per 1M)
- Long context pricing (>200K) has premium surcharge for Opus 4.6
- 1M context for Opus 4.6 still in beta (requires usage tier 4)
- No free tier for API access

#### corteX Status
Partially integrated (OpenAI-compatible endpoint). Should be elevated to first-class provider.

---

### 3. OpenAI

OpenAI's GPT-5.x family offers a broad range from the ultra-cheap Nano to the flagship 5.2 Pro. The model lineup is now the most diverse of any provider.

#### Current Models

| Model | Released | Context | Max Output | Status |
|-------|----------|---------|------------|--------|
| `gpt-5.2` (Instant/Thinking/Pro) | Dec 2025 | 400K tokens | 128K tokens | GA |
| `gpt-5` | 2025 | 128K tokens | 32K tokens | GA |
| `gpt-5-mini` | 2025 | 128K tokens | 32K tokens | GA |
| `gpt-5-nano` | 2025 | 128K tokens | 16K tokens | GA |
| `o3` | 2025 | 200K tokens | 100K tokens | GA |
| `o3-pro` | 2025 | 200K tokens | 100K tokens | GA |
| `o4-mini` | 2025 | 200K tokens | 100K tokens | GA |
| `gpt-4o` | 2024 | 128K tokens | 16K tokens | Legacy |
| `gpt-4.1` | 2025 | 1M tokens | 32K tokens | GA |

#### Pricing (per 1M tokens)

| Model | Input | Output | Cached Input |
|-------|-------|--------|-------------|
| GPT-5.2 | $1.75 | $14.00 | $0.175 (90% discount) |
| GPT-5.2 Pro | $21.00 | $168.00 | N/A |
| GPT-5 | $1.25 | $10.00 | $0.125 |
| GPT-5 Mini | $0.25 | $2.00 | $0.025 |
| GPT-5 Nano | $0.05 | $0.40 | $0.005 |
| o3 | $2.00 | $8.00 | N/A |
| o3-pro | $20.00 | $80.00 | N/A |
| o4-mini | $1.10 | $4.40 | N/A |
| GPT-4o | $2.50 | $10.00 | $1.25 |

#### Key Strengths
- **Broadest model range** -- from $0.05/1M (Nano) to $21/1M (5.2 Pro) covers every use case
- **GPT-5.2 leads LiveCodeBench** (89%) and many agentic benchmarks
- **400K context window** on GPT-5.2 -- largest among proprietary non-Google models
- **128K output tokens** -- largest output window in the industry
- **Adaptive reasoning** -- GPT-5.2 dynamically allocates compute based on task complexity
- **Strong tooling ecosystem** -- Assistants API, function calling, structured outputs
- **GPT-5 Nano** -- ultra-cheap for classification, routing, and simple extraction

#### Key Weaknesses
- GPT-5.2 Pro pricing is extremely expensive ($21/$168)
- o3-pro at $20/$80 is costly for reasoning-heavy workflows
- No open-source options
- Knowledge cutoff (Aug 2025 for GPT-5.2) may miss recent events

#### corteX Status
Partially integrated. Should expand support to full GPT-5.x family and reasoning models.

---

### 4. Meta (Llama 4)

Meta's Llama 4 family represents the most capable open-source models available. The MoE architecture provides excellent efficiency for on-prem deployment.

#### Current Models

| Model | Parameters (Active/Total) | Experts | Context | Status |
|-------|--------------------------|---------|---------|--------|
| Llama 4 Scout | 17B / 109B | 16 | 10M tokens | Released (Apache 2.0) |
| Llama 4 Maverick | 17B / 400B | 128 | 1M tokens | Released (Apache 2.0) |
| Llama 4 Behemoth | 288B / 2T | 16 | TBD | Training |

#### Pricing
- **Free** -- open-source under permissive license
- Inference costs depend on hosting (self-hosted or cloud providers)
- Cloud provider pricing varies: typically $0.10-$0.50/1M input for Scout, $0.30-$1.00/1M for Maverick

#### Hardware Requirements (On-Prem)

| Model | FP16 | INT8 Quantized | INT4 Quantized |
|-------|------|----------------|----------------|
| Scout (109B total) | 2x H100 80GB | 1x H100 80GB | 1x H100 80GB |
| Maverick (400B total) | 8x H100 80GB | 4x H100 80GB | 1x H100 DGX (8 GPUs) |
| Behemoth (2T total) | Not feasible | 16+ H100 | 8+ H100 |

#### Key Strengths
- **10M token context** on Scout -- largest production context window available
- **Free and open-source** -- no API costs, full control, Apache 2.0 license
- **MoE architecture** -- only 17B active parameters means fast inference despite large total size
- **Natively multimodal** -- text, images, video understanding
- **200 language pre-training** -- strongest multilingual support
- **Scout fits on single H100** with INT4 quantization -- very practical for on-prem

#### Key Weaknesses
- Behemoth still not released (still training as of Feb 2026)
- MoE models require more memory than dense models of equivalent active params
- Tool calling and function calling less mature than proprietary models
- Requires significant engineering for production deployment

#### corteX Status
Supported via Ollama integration. Should add vLLM support for production on-prem deployments.

---

### 5. Mistral AI

Mistral continues to offer strong European-hosted alternatives with competitive pricing and enterprise-friendly licensing.

#### Current Models

| Model | Parameters (Active/Total) | Architecture | Context | Released |
|-------|--------------------------|-------------|---------|----------|
| Mistral Large 3 | 41B / 675B | MoE | 128K tokens | Dec 2025 |
| Mistral 3 14B | 14B | Dense | 128K tokens | Dec 2025 |
| Mistral 3 8B | 8B | Dense | 128K tokens | Dec 2025 |
| Mistral 3 3B | 3B | Dense | 32K tokens | Dec 2025 |
| Devstral 2 | Undisclosed | Dense | 128K tokens | Dec 2025 |
| Devstral Small 2 (24B) | 24B | Dense | 128K tokens | Dec 2025 |

#### Pricing (per 1M tokens)

| Model | Input | Output |
|-------|-------|--------|
| Mistral Large 3 | $2.00 | $5.00 |
| Mistral 3 14B | $0.10 | $0.30 |
| Mistral 3 8B | $0.05 | $0.15 |
| Mistral 3 3B | $0.02 | $0.06 |
| Devstral Small 2 | $0.15 | $0.45 |

#### Key Strengths
- **European hosting** -- GDPR compliance, EU data residency
- **Devstral 2** -- purpose-built for coding tasks, strong performance
- **Competitive pricing** -- Large 3 at $2/$5 is cheaper than GPT-5 and Claude Sonnet
- **Open-weight models** -- Mistral 3 series available for self-hosting
- **Available on many platforms** -- AWS Bedrock, Azure, HuggingFace, OpenRouter
- **Small models (3B/8B)** -- excellent for edge deployment and low-resource environments

#### Key Weaknesses
- 128K context is smaller than Google/Anthropic/OpenAI offerings
- Less established ecosystem than major US providers
- Benchmarks trail behind Gemini 3 Pro and Claude Opus on complex agent tasks
- MoE Large 3 requires significant GPU memory for self-hosting

#### corteX Status
Not currently integrated. Should add as European-friendly provider option.

---

### 6. DeepSeek

DeepSeek offers extraordinary value with open-source models that rival proprietary options at a fraction of the cost.

#### Current Models

| Model | Type | Context | Released | Status |
|-------|------|---------|----------|--------|
| DeepSeek V3.2 Exp | General | 128K tokens | Sep 2025 | Experimental |
| DeepSeek V3.1 Terminus | General | 128K tokens | Sep 2025 | GA |
| DeepSeek V3.1 | General | 128K tokens | Aug 2025 | GA |
| DeepSeek V3 | General | 66K tokens | Dec 2024 | GA |
| DeepSeek R1 | Reasoning | 128K tokens | Jan 2025 | GA |

#### Pricing (per 1M tokens, DeepSeek API)

| Model | Input | Output |
|-------|-------|--------|
| DeepSeek V3 | $0.30 | $1.20 |
| DeepSeek V3.1 | ~$0.30 | ~$1.20 |
| DeepSeek R1 | $0.70 | $2.50 |
| R1 Distill Llama 70B | $0.03 | $0.10 |

#### Key Strengths
- **Astonishingly cheap** -- R1 offers o1-level reasoning at ~95% lower cost
- **Open-source and open-weight** -- full model weights available
- **R1 Distill variants** -- distilled reasoning into smaller models (Llama 70B, 8B, etc.)
- **Strong on math, coding, and logic** -- competitive with much more expensive models
- **V3.2 competitive with frontier models** on reasoning and agentic benchmarks

#### Key Weaknesses
- Chinese company -- some enterprises have data sovereignty concerns
- API reliability and rate limits less established than US providers
- Smaller context windows (66K-128K) vs. 1M from Google/Anthropic
- Model updates less frequent and less predictable
- Output limited to 8K tokens on V3

#### corteX Status
Not currently integrated. Should add support, especially for cost-sensitive deployments and R1 for reasoning tasks.

---

### 7. Alibaba Qwen 3

Qwen 3 is a powerhouse open-source family with excellent agent capabilities and native MCP support.

#### Current Models

| Model | Parameters | Architecture | Context | Released |
|-------|-----------|-------------|---------|----------|
| Qwen3-Max-Thinking | Undisclosed (hosted) | Proprietary | 128K+ | Jan 2026 |
| Qwen3-Coder-Next | 80B-A3B | MoE (ultra-sparse) | 128K | 2026 |
| Qwen3-235B-A22B | 235B (22B active) | MoE | 128K | Apr 2025 |
| Qwen3-30B-A3B | 30B (3B active) | MoE | 128K | Apr 2025 |
| Qwen3-32B | 32B | Dense | 128K | Apr 2025 |
| Qwen3-14B | 14B | Dense | 128K | Apr 2025 |
| Qwen3-8B | 8B | Dense | 128K | Apr 2025 |
| Qwen3-4B | 4B | Dense | 32K | Apr 2025 |
| Qwen3-1.7B | 1.7B | Dense | 32K | Apr 2025 |
| Qwen3-0.6B | 0.6B | Dense | 32K | Apr 2025 |
| Qwen3-VL | Various | Multimodal | 128K | 2025 |

#### Pricing (Alibaba Cloud, per 1M tokens)

| Model | Input | Output |
|-------|-------|--------|
| Qwen3-Max-Thinking | ~$1.50 | ~$6.00 |
| Qwen3-235B-A22B | Self-hosted (free) | Self-hosted |
| Qwen3-30B-A3B | Self-hosted (free) | Self-hosted |
| Smaller models | Self-hosted (free) | Self-hosted |

#### Key Strengths
- **Hybrid reasoning** -- seamless switching between thinking and non-thinking modes
- **Native MCP support** -- built-in Model Context Protocol and function calling
- **119 language support** -- broadest multilingual capability
- **Ultra-sparse MoE** -- Qwen3-Coder-Next has 10x higher throughput for coding
- **Excellent range** -- from 0.6B (runs on phones) to 235B (competes with frontier)
- **36 trillion training tokens** -- one of the largest training datasets
- **Competitive with GPT-5.2** on many benchmarks (Qwen3-Max-Thinking)

#### Key Weaknesses
- Chinese company -- same data sovereignty concerns as DeepSeek
- Alibaba Cloud API less accessible in some regions
- Largest models require significant GPU memory for self-hosting
- Less established in Western enterprise markets

#### corteX Status
Not currently integrated. Strong candidate for open-source on-prem deployment.

---

### 8. xAI (Grok)

xAI's Grok models offer competitive capabilities with the largest context window in the industry.

#### Current Models

| Model | Context | Released | Status |
|-------|---------|----------|--------|
| Grok 4 | 2M tokens | 2026 | GA |
| Grok 4.1 Fast | 2M tokens | 2026 | GA |
| Grok 3 | 2M tokens | Jun 2025 | GA |

#### Pricing (per 1M tokens)

| Model | Input | Output |
|-------|-------|--------|
| Grok 4 | $3.00 | $15.00 |
| Grok 4.1 Fast | $0.20 | $1.00 |
| Grok 3 | $3.00 | $15.00 |

#### Key Strengths
- **2M token context window** -- largest in the industry
- **$25 free credits** for new users + $150/month data sharing credits
- **Grok 4.1 Fast** -- very competitive at $0.20/$1.00 per 1M
- **Strong reasoning capabilities**

#### Key Weaknesses
- Smaller ecosystem and less enterprise adoption
- Tool invocation charges ($2.50-$5.00 per 1,000 calls) on top of token costs
- Less mature API compared to Google/OpenAI/Anthropic
- Limited enterprise support infrastructure

#### corteX Status
Not currently integrated. Could add as optional provider.

---

### 9. Cohere

Cohere focuses on enterprise RAG and search use cases with strong on-prem deployment options.

#### Current Models

| Model | Context | Best For |
|-------|---------|---------|
| Command R+ | 128K tokens | Complex reasoning, multi-step tool use |
| Command R | 128K tokens | General purpose |
| Command R7B | 128K tokens | Lightweight, fast |
| Command A | 256K tokens | Agentic tasks |

#### Pricing (per 1M tokens)

| Model | Input | Output |
|-------|-------|--------|
| Command R+ | $2.50 | $10.00 |
| Command R | $0.15 | $0.60 |
| Command A | ~$2.50 | ~$10.00 |

#### Key Strengths
- **Enterprise-focused** -- private deployment, custom models, dedicated support
- **Strong RAG capabilities** -- Rerank API, Embed API for search pipelines
- **On-prem deployment** available with enterprise licensing
- **Grounded generation** -- can cite sources in responses

#### Key Weaknesses
- Smaller model range than competitors
- Less competitive on pure coding/reasoning benchmarks
- 128K context (256K for Command A) smaller than many competitors
- Higher pricing than DeepSeek/Qwen for equivalent performance

#### corteX Status
Not currently integrated. Lower priority unless enterprise RAG is a focus.

---

## Master Pricing Comparison Table

Sorted by input price (ascending). All prices per 1M tokens.

| Model | Provider | Input | Output | Context | Max Output | Best Use Case |
|-------|----------|-------|--------|---------|------------|---------------|
| Mistral 3 3B | Mistral | $0.02 | $0.06 | 32K | 32K | Edge, mobile, routing |
| R1 Distill Llama 70B | DeepSeek | $0.03 | $0.10 | 128K | 8K | Cheap reasoning |
| GPT-5 Nano | OpenAI | $0.05 | $0.40 | 128K | 16K | Classification, routing |
| Mistral 3 8B | Mistral | $0.05 | $0.15 | 128K | 128K | Light tasks, extraction |
| Mistral 3 14B | Mistral | $0.10 | $0.30 | 128K | 128K | General purpose, cheap |
| Devstral Small 2 | Mistral | $0.15 | $0.45 | 128K | 128K | Coding, cheap |
| Gemini 2.5 Flash | Google | $0.15 | $0.60 | 1M | 64K | High-throughput worker |
| Grok 4.1 Fast | xAI | $0.20 | $1.00 | 2M | TBD | Fast general purpose |
| GPT-5 Mini | OpenAI | $0.25 | $2.00 | 128K | 32K | Balanced cost/quality |
| DeepSeek V3 | DeepSeek | $0.30 | $1.20 | 66K | 8K | General purpose, cheap |
| Gemini 3 Flash | Google | $0.50 | $3.00 | 1M | 64K | **Best worker model** |
| DeepSeek R1 | DeepSeek | $0.70 | $2.50 | 128K | 8K | Reasoning, math, logic |
| Claude Haiku 4.5 | Anthropic | $1.00 | $5.00 | 200K | 32K | Fast, triage, routing |
| o4-mini | OpenAI | $1.10 | $4.40 | 200K | 100K | Reasoning, efficient |
| GPT-5 | OpenAI | $1.25 | $10.00 | 128K | 32K | General flagship |
| Qwen3-Max-Thinking | Alibaba | ~$1.50 | ~$6.00 | 128K+ | TBD | Reasoning, multilingual |
| GPT-5.2 | OpenAI | $1.75 | $14.00 | 400K | 128K | Complex tasks, coding |
| Gemini 3 Pro | Google | $2.00 | $12.00 | 1M | 64K | **Best orchestrator** |
| o3 | OpenAI | $2.00 | $8.00 | 200K | 100K | Deep reasoning |
| Mistral Large 3 | Mistral | $2.00 | $5.00 | 128K | 128K | European enterprise |
| Command R+ | Cohere | $2.50 | $10.00 | 128K | 4K | Enterprise RAG |
| Grok 4 | xAI | $3.00 | $15.00 | 2M | TBD | Large context tasks |
| Claude Sonnet 4.5 | Anthropic | $3.00 | $15.00 | 1M | 32K | Balanced quality/cost |
| Claude Opus 4.6 | Anthropic | $5.00 | $25.00 | 1M (beta) | 32K | **Best agentic model** |
| o3-pro | OpenAI | $20.00 | $80.00 | 200K | 100K | Maximum reasoning |
| GPT-5.2 Pro | OpenAI | $21.00 | $168.00 | 400K | 128K | Maximum intelligence |

### Open-Source Models (Free, Self-Hosted)

| Model | Provider | Parameters (Active/Total) | Context | GPU Requirement |
|-------|----------|--------------------------|---------|-----------------|
| Qwen3-0.6B | Alibaba | 0.6B | 32K | CPU / Phone |
| Qwen3-1.7B | Alibaba | 1.7B | 32K | Any GPU / CPU |
| Mistral 3 3B | Mistral | 3B | 32K | 4GB VRAM |
| Qwen3-4B | Alibaba | 4B | 32K | 6GB VRAM |
| Qwen3-8B / Mistral 3 8B | Various | 8B | 128K | 8-12GB VRAM |
| Qwen3-14B / Mistral 3 14B | Various | 14B | 128K | 16-24GB VRAM |
| Devstral Small 2 | Mistral | 24B | 128K | 24-32GB VRAM |
| Qwen3-30B-A3B | Alibaba | 3B / 30B (MoE) | 128K | 24GB VRAM |
| Qwen3-32B | Alibaba | 32B | 128K | 32-48GB VRAM |
| Llama 4 Scout | Meta | 17B / 109B (MoE) | 10M | 1x H100 (INT4) |
| Qwen3-235B-A22B | Alibaba | 22B / 235B (MoE) | 128K | 2-4x H100 |
| Llama 4 Maverick | Meta | 17B / 400B (MoE) | 1M | 1x H100 DGX (8 GPUs) |
| Mistral Large 3 | Mistral | 41B / 675B (MoE) | 128K | 4-8x H100 |
| DeepSeek V3 | DeepSeek | 37B / 671B (MoE) | 66K | 8x H200 |

---

## On-Premises / Self-Hosted Options

### Critical for corteX's On-Prem-First Philosophy

corteX's core value proposition includes 100% on-prem capability. This section covers the practical landscape for self-hosted inference.

### Recommended Inference Frameworks

| Framework | Best For | Throughput | Latency | Multi-GPU | Production Ready |
|-----------|---------|-----------|---------|-----------|-----------------|
| **vLLM** | Production serving | 793 TPS | 80ms P99 | Yes | Yes |
| **SGLang** | Maximum throughput | Higher than vLLM | Similar | Yes | Yes |
| **TGI** | HuggingFace ecosystem | 100-140 req/sec | 60-90ms TTFT | Yes | Yes |
| **TensorRT-LLM** | NVIDIA optimization | Highest | Lowest | Yes | Yes |
| **Ollama** | Development, single-user | 41 TPS | 673ms P99 | Limited | Dev only |
| **llama.cpp** | CPU inference, edge | Low | Variable | No | Yes |

#### Framework Recommendations for corteX

1. **Development / Testing**: Ollama (simple setup, model management, good DX)
2. **Production Single-GPU**: vLLM (best balance of throughput and latency)
3. **Production Multi-GPU**: SGLang or vLLM (depends on model support)
4. **NVIDIA Hardware**: TensorRT-LLM (maximum performance on NVIDIA GPUs)
5. **CPU-Only / Edge**: llama.cpp (runs on consumer hardware)

### Recommended On-Prem Models by Hardware Tier

#### Tier 1: Consumer GPU (RTX 4090 / 5090, 24-32GB VRAM)
| Model | Quantization | VRAM Usage | Quality Level |
|-------|-------------|-----------|---------------|
| Qwen3-14B | INT4 | ~10GB | Good for general tasks |
| Devstral Small 2 (24B) | INT4 | ~16GB | Good for coding |
| Qwen3-30B-A3B (MoE) | FP16 | ~24GB | Very good, only 3B active |
| Qwen3-32B | INT4 | ~20GB | Good reasoning |

#### Tier 2: Workstation GPU (A6000 48GB / dual consumer GPUs)
| Model | Quantization | VRAM Usage | Quality Level |
|-------|-------------|-----------|---------------|
| Qwen3-32B | FP16 | ~64GB (2x GPU) | Excellent reasoning |
| GLM-4.7 Thinking | INT4 | ~40GB | Best self-hostable agent |
| Llama 4 Scout | INT4 | ~60-70GB | Excellent, 10M context |

#### Tier 3: Data Center (H100 80GB / A100 80GB)
| Model | Quantization | GPUs | Quality Level |
|-------|-------------|------|---------------|
| Llama 4 Scout | INT4 | 1x H100 | Frontier-class, 10M context |
| Qwen3-235B-A22B | INT4 | 2-4x H100 | Near-frontier MoE |
| Llama 4 Maverick | FP8 | 8x H100 | Frontier-class |
| DeepSeek V3 | FP8 | 8x H200 | Frontier-class |
| Mistral Large 3 | INT4 | 4-8x H100 | Frontier-class |

### On-Prem vs. Cloud API Quality Comparison

| Capability | Best Cloud API | Best Self-Hosted | Gap |
|-----------|---------------|-----------------|-----|
| Tool calling | GPT-5.2 / Claude Opus 4.6 | GLM-4.7 Thinking | Small (90%+ on both) |
| Code generation | GPT-5.2 / Claude Sonnet 4.5 | Qwen3-Coder-Next / Devstral 2 | Small |
| Reasoning | Claude Opus 4.6 | DeepSeek R1 / Qwen3-Max-Thinking | Small-Medium |
| Long context | Gemini 3 Pro (1M) | Llama 4 Scout (10M) | **Scout wins** |
| Speed/latency | All (optimized infrastructure) | vLLM on H100 | Medium (cloud faster) |
| Cost at scale | Expensive | **Self-hosted wins** at >100K req/day | Large savings |

---

## Agent-Specific Model Capabilities

### Tool Calling / Function Calling

Models ranked by Berkeley Function Calling Leaderboard (BFCL) V4 and real-world agent benchmarks:

| Rank | Model | BFCL Score | Notes |
|------|-------|-----------|-------|
| 1 | GPT-5.2 (xhigh) | ~95%+ | Best overall, preamble explains reasoning |
| 2 | Claude Opus 4.6 | ~93%+ | Best multi-tool orchestration |
| 3 | Gemini 3 Pro | ~92%+ | Strong native tool use |
| 4 | GLM-4.7 Thinking | ~90%+ | Best self-hostable for agents |
| 5 | Qwen3-235B-A22B | ~88% | Native MCP support |
| 6 | Llama 4 Maverick | ~85% | Improving rapidly |
| 7 | DeepSeek V3.2 | ~83% | Good value |
| 8 | Mistral Large 3 | ~80% | Solid European option |

### Long Context (10K+ Step Workflows)

| Model | Context | Retrieval Accuracy at 256K | Best For |
|-------|---------|---------------------------|---------|
| Llama 4 Scout | 10M | Good | Massive document processing |
| Grok 4 | 2M | Good | Large context tasks |
| Gemini 3 Pro | 1M | Very Good | Agent orchestration |
| Claude Opus 4.6 | 1M (beta) | 93% (MRCR v2) | **Best retrieval accuracy** |
| Claude Sonnet 4.5 | 1M | 10.8% (poor) | Not for deep retrieval |
| GPT-5.2 | 400K | ~100% (up to 256K) | Strong retrieval |

### Code Generation and Debugging

| Rank | Model | SWE-bench | LiveCodeBench | Best For |
|------|-------|-----------|---------------|---------|
| 1 | GPT-5.2 | High | 89% | Raw code generation |
| 2 | Gemini 3 Flash | 78% | High | Cost-effective coding |
| 3 | Claude Opus 4.6 | 65.4% (Terminal-Bench 2.0) | High | Debugging, architecture |
| 4 | Claude Sonnet 4.5 | Good | Good | Day-to-day development |
| 5 | Qwen3-Coder-Next | Good | Good | Open-source coding |
| 6 | Devstral 2 / Small 2 | Good | Good | Self-hosted coding |
| 7 | DeepSeek V3.1 | Good | Good | Budget coding |

### Reasoning and Planning

| Rank | Model | GPQA Diamond | ARC-AGI-2 | Humanity's Last Exam |
|------|-------|-------------|-----------|---------------------|
| 1 | Gemini 3 Pro | 91.9% | Near 2x GPT-5.1 | High |
| 2 | Claude Opus 4.6 | High | 68.8% | 53.1% (with tools) |
| 3 | GPT-5.2 Thinking | High | High | High |
| 4 | o3-pro | Very High | High | High |
| 5 | DeepSeek R1 | High | Good | Good |
| 6 | Qwen3-Max-Thinking | High | Good | Good |

### Self-Reflection and Error Correction

| Model | Capability | Notes |
|-------|-----------|-------|
| Claude Opus 4.6 | Excellent | Best at recognizing and correcting own errors |
| GPT-5.2 Thinking | Excellent | Adaptive reasoning allocates compute to hard parts |
| Gemini 3 Pro | Very Good | Thinking mode with configurable depth |
| DeepSeek R1 | Very Good | Shows reasoning chain, catches errors |
| o3 / o3-pro | Excellent | Purpose-built for deep reasoning |

---

## Integration Recommendations for corteX

### 1. Priority Provider Support

Based on the research, corteX should support providers in this priority order:

| Priority | Provider | Rationale |
|----------|----------|-----------|
| P0 (Must Have) | Google Gemini | Already integrated; best price/performance; free tier |
| P0 (Must Have) | Anthropic Claude | Best agentic performance; enterprise trust |
| P0 (Must Have) | OpenAI GPT-5.x | Broadest model range; industry standard |
| P0 (Must Have) | Local/On-Prem (Ollama/vLLM) | Core value prop of corteX |
| P1 (Should Have) | Meta Llama 4 (via vLLM/Ollama) | Best open-source; 10M context |
| P1 (Should Have) | DeepSeek | 95% cost reduction for reasoning; open-source |
| P1 (Should Have) | Alibaba Qwen 3 | Excellent open-source MoE; native MCP |
| P2 (Nice to Have) | Mistral | European market; good coding models |
| P2 (Nice to Have) | xAI Grok | 2M context; growing ecosystem |
| P3 (Low Priority) | Cohere | Niche RAG use cases |

### 2. Recommended Model Tiers

corteX should implement a tiered model system for intelligent routing:

#### Tier Architecture

```
ORCHESTRATOR (complex reasoning, planning, tool selection)
    |
    v
WORKER (task execution, code generation, data processing)
    |
    v
ROUTER/TRIAGE (intent classification, complexity estimation)
    |
    v
FALLBACK (when primary models are unavailable or rate-limited)
```

#### Recommended Default Configurations

**Cloud-First Configuration (Best Quality)**:
| Role | Primary | Secondary | Fallback |
|------|---------|-----------|----------|
| Orchestrator | Claude Opus 4.6 | Gemini 3 Pro | GPT-5.2 Thinking |
| Worker | Gemini 3 Flash | Claude Sonnet 4.5 | GPT-5 |
| Router/Triage | GPT-5 Nano | Claude Haiku 4.5 | Gemini 2.5 Flash |
| Reasoning | o3 / DeepSeek R1 | Claude Opus 4.6 | Gemini 3 Pro |

**Cost-Optimized Configuration**:
| Role | Primary | Secondary | Fallback |
|------|---------|-----------|----------|
| Orchestrator | Gemini 3 Pro | DeepSeek R1 | GPT-5 |
| Worker | Gemini 3 Flash | DeepSeek V3 | GPT-5 Mini |
| Router/Triage | GPT-5 Nano | Mistral 3 3B | Gemini 2.5 Flash |
| Reasoning | DeepSeek R1 | Gemini 3 Pro | o4-mini |

**On-Prem Configuration (Zero External Dependencies)**:
| Role | Primary | Secondary | Fallback |
|------|---------|-----------|----------|
| Orchestrator | Llama 4 Scout (vLLM) | Qwen3-235B-A22B | Qwen3-32B |
| Worker | Qwen3-30B-A3B | Devstral Small 2 | Qwen3-14B |
| Router/Triage | Qwen3-4B | Mistral 3 3B | Qwen3-1.7B |
| Reasoning | GLM-4.7 Thinking | DeepSeek R1 (local) | Qwen3-32B |
| Coding | Qwen3-Coder-Next | Devstral 2 | Devstral Small 2 |

### 3. Intelligent Model Routing

corteX should implement a routing system that considers:

#### Routing Criteria

```python
class ModelRouter:
    """Route requests to optimal model based on task characteristics."""

    def route(self, task: AgentTask) -> ModelSelection:
        # 1. Complexity estimation (use cheap model to classify)
        complexity = self.estimate_complexity(task)  # low/medium/high/extreme

        # 2. Task type detection
        task_type = self.classify_task(task)  # reasoning/coding/extraction/routing

        # 3. Context length requirement
        context_needed = self.estimate_context(task)

        # 4. Cost budget check
        budget = task.cost_budget or self.default_budget

        # 5. Latency requirement
        max_latency = task.max_latency or self.default_latency

        # Route to optimal model
        return self.select_model(
            complexity=complexity,
            task_type=task_type,
            context_needed=context_needed,
            budget=budget,
            max_latency=max_latency
        )
```

#### Routing Decision Matrix

| Complexity | Task Type | Recommended Model | Cost/1M Out |
|-----------|-----------|-------------------|-------------|
| Low | Classification | GPT-5 Nano | $0.40 |
| Low | Extraction | Gemini 3 Flash | $3.00 |
| Low | Simple Q&A | Claude Haiku 4.5 | $5.00 |
| Medium | Code generation | Gemini 3 Flash | $3.00 |
| Medium | Analysis | Claude Sonnet 4.5 | $15.00 |
| Medium | Summarization | GPT-5 Mini | $2.00 |
| High | Multi-step reasoning | Gemini 3 Pro | $12.00 |
| High | Complex debugging | Claude Opus 4.6 | $25.00 |
| High | Architecture planning | Claude Opus 4.6 | $25.00 |
| Extreme | Deep research | o3-pro / GPT-5.2 Pro | $80-168.00 |
| Any | Long context (>200K) | Gemini 3 Pro / Flash | $3-12.00 |
| Any | On-prem required | Llama 4 Scout / Qwen3 | Free (HW cost) |

### 4. Cost Optimization Strategies

Based on the research, these strategies can reduce LLM costs by 70-85%:

#### Strategy 1: Cascade Routing
Route simple queries to cheap models first. Only escalate to expensive models if the cheap model signals low confidence.

```
User Query -> GPT-5 Nano (classify complexity, $0.05/1M)
    -> If simple: Gemini 3 Flash ($0.50/$3.00)
    -> If medium: Claude Sonnet 4.5 ($3/$15)
    -> If complex: Claude Opus 4.6 ($5/$25)
```
**Expected savings: 60-70%** (most queries are simple)

#### Strategy 2: Context Caching
Cache repeated context (system prompts, tool definitions, conversation history) using provider caching APIs.

- Google Gemini: 90% discount on cached tokens
- OpenAI: 90% discount on cached input
- Anthropic: Up to 90% with prompt caching

**Expected savings: 40-60%** for multi-turn conversations

#### Strategy 3: Batch Processing
For non-real-time tasks (analysis, report generation, data processing), use batch APIs.

- Anthropic: 50% discount for batch processing
- OpenAI: Batch API with significant discounts

**Expected savings: 50%** on batch-eligible workloads

#### Strategy 4: Prompt Compression
Reduce token count by optimizing prompts, using structured formats, and avoiding redundancy.

**Expected savings: 20-30%** on token usage

#### Strategy 5: On-Prem for High-Volume
At >100K requests/day, self-hosted models become cheaper than API calls even with GPU costs.

**Break-even analysis:**
- 1x H100 lease: ~$2-3/hour ($1,500-$2,200/month)
- Running Llama 4 Scout on 1x H100: handles ~100-500 req/min
- Equivalent API cost: $3,000-$15,000/month for Gemini 3 Flash
- **Savings: 50-85%** at scale

### 5. Provider Abstraction Architecture

corteX should implement a clean provider abstraction that makes model switching transparent:

```python
# Recommended provider interface for corteX
class LLMProvider(Protocol):
    """Abstract provider interface supporting all major LLM providers."""

    async def complete(
        self,
        messages: list[Message],
        tools: list[ToolDefinition] | None = None,
        model: str | None = None,
        temperature: float = 0.7,
        max_tokens: int | None = None,
        response_format: ResponseFormat | None = None,
    ) -> CompletionResponse: ...

    async def stream(
        self,
        messages: list[Message],
        tools: list[ToolDefinition] | None = None,
        model: str | None = None,
    ) -> AsyncIterator[StreamChunk]: ...

    def supports_tool_calling(self) -> bool: ...
    def supports_vision(self) -> bool: ...
    def supports_caching(self) -> bool: ...
    def max_context_tokens(self) -> int: ...
    def cost_per_million(self, direction: str) -> float: ...

# Provider implementations needed:
# - GeminiProvider (existing)
# - AnthropicProvider (new - first-class)
# - OpenAIProvider (existing, needs GPT-5.x update)
# - OllamaProvider (existing, needs vLLM alternative)
# - vLLMProvider (new - for production on-prem)
# - MistralProvider (new)
# - DeepSeekProvider (new - OpenAI-compatible)
# - CohereProvider (new - low priority)
```

---

## Key Takeaways

1. **The market is fragmenting in corteX's favor** -- no single provider dominates all use cases, making provider-agnostic SDKs essential

2. **Gemini 3 Flash is the best default worker model** -- $0.50/$3.00 per 1M tokens with 1M context and 78% SWE-bench is unmatched value

3. **Claude Opus 4.6 is the best agentic orchestrator** -- leads on nearly every agent benchmark (Terminal-Bench, OSWorld, BrowseComp, ARC-AGI-2)

4. **Open-source has reached production quality** -- GLM-4.7, Qwen3-Coder-Next, and Llama 4 Scout are viable on-prem alternatives to proprietary models

5. **Cost optimization through routing is mandatory** -- enterprises using intelligent routing report 70-85% cost reductions. corteX should build this into the SDK

6. **The model zoo is growing faster than ever** -- corteX's provider abstraction layer must be designed for easy addition of new providers (most use OpenAI-compatible APIs)

7. **Llama 4 Scout (10M context, single H100) is the best on-prem model** for corteX's on-prem-first philosophy

8. **GPT-5 Nano at $0.05/1M is transformative** -- enables routing/classification at near-zero cost, making cascade routing economically viable

---

## Sources

### Google Gemini
- [Google Gemini 3 Flash Announcement](https://blog.google/products/gemini/gemini-3-flash/)
- [Gemini API Pricing 2026](https://www.metacto.com/blogs/the-true-cost-of-google-gemini-a-guide-to-api-pricing-and-integration)
- [Gemini 3 Flash on OpenRouter](https://openrouter.ai/google/gemini-3-flash-preview)
- [Gemini Models Documentation](https://ai.google.dev/gemini-api/docs/models)
- [Gemini 3 Pro on Google DeepMind](https://deepmind.google/models/gemini/pro/)
- [Gemini 3 Benchmarks](https://www.vellum.ai/blog/google-gemini-3-benchmarks)
- [Gemini Deprecation Schedule](https://ai.google.dev/gemini-api/docs/deprecations)
- [Gemini Pricing Impact Discussion](https://discuss.ai.google.dev/t/pricing-impact-following-the-deprecation-of-gemini-2-5-flash/112737)

### Anthropic Claude
- [Claude API Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- [Claude Models Overview](https://platform.claude.com/docs/en/about-claude/models/overview)
- [Introducing Claude Opus 4.6](https://www.anthropic.com/news/claude-opus-4-6)
- [Claude Opus 4.6 Benchmarks](https://www.vellum.ai/blog/claude-opus-4-6-benchmarks)
- [Claude Opus 4.6 vs 4.5](https://ssntpl.com/blog-claude-opus-4-6-vs-4-5-benchmarks-testing/)
- [Claude Opus 4.6 Features Guide](https://www.digitalapplied.com/blog/claude-opus-4-6-release-features-benchmarks-guide)
- [Claude Haiku 4.5 Announcement](https://www.anthropic.com/news/claude-haiku-4-5)
- [Claude Haiku 4.5 Deep Dive](https://caylent.com/blog/claude-haiku-4-5-deep-dive-cost-capabilities-and-the-multi-agent-opportunity)

### OpenAI
- [OpenAI API Pricing](https://platform.openai.com/docs/pricing)
- [Introducing GPT-5.2](https://openai.com/index/introducing-gpt-5-2/)
- [GPT-5.2 Model Docs](https://platform.openai.com/docs/models/gpt-5.2)
- [GPT-5 Nano Docs](https://platform.openai.com/docs/models/gpt-5-nano)
- [GPT-5 Mini Docs](https://platform.openai.com/docs/models/gpt-5-mini)
- [GPT-5.2 Benchmarks](https://www.datacamp.com/blog/gpt-5-2)
- [OpenAI Pricing 2026 Guide](https://www.finout.io/blog/openai-pricing-in-2026)

### Meta Llama
- [Llama 4 Announcement](https://ai.meta.com/blog/llama-4-multimodal-intelligence/)
- [Llama 4 on HuggingFace](https://huggingface.co/blog/llama4-release)
- [Llama 4 Models Page](https://www.llama.com/models/llama-4/)
- [Llama 4 Explained](https://www.techtarget.com/whatis/feature/Meta-Llama-4-explained-Everything-you-need-to-know)

### Mistral
- [Mistral Pricing](https://mistral.ai/pricing)
- [Introducing Mistral 3](https://mistral.ai/news/mistral-3)
- [Mistral API Pricing 2026](https://pricepertoken.com/pricing-page/provider/mistral-ai)

### DeepSeek
- [DeepSeek Pricing](https://api-docs.deepseek.com/quick_start/pricing)
- [DeepSeek Models Overview](https://pricepertoken.com/pricing-page/provider/deepseek)

### Alibaba Qwen
- [Qwen3 GitHub](https://github.com/QwenLM/Qwen3)
- [Qwen3 Announcement](https://www.alibabacloud.com/blog/alibaba-introduces-qwen3-setting-new-benchmark-in-open-source-ai-with-hybrid-reasoning_602192)
- [Qwen3-Max-Thinking Guide](https://www.digitalapplied.com/blog/qwen3-max-thinking-alibaba-reasoning-model-guide)
- [Qwen3-Coder-Next](https://venturebeat.com/technology/qwen3-coder-next-offers-vibe-coders-a-powerful-open-source-ultra-sparse)

### xAI Grok
- [xAI API](https://x.ai/api)
- [Grok Models and Pricing](https://docs.x.ai/developers/models)
- [Grok Review 2026](https://hackceleration.com/grok-review/)

### Cohere
- [Cohere Pricing](https://cohere.com/pricing)
- [Cohere Models Overview](https://docs.cohere.com/docs/models)

### Agent Benchmarks & Capabilities
- [Berkeley Function Calling Leaderboard V4](https://gorilla.cs.berkeley.edu/leaderboard.html)
- [SEAL Agentic Tool Use Leaderboard](https://scale.com/leaderboard/tool_use_enterprise)
- [Best Agentic Models January 2026](https://whatllm.org/blog/best-agentic-models-january-2026)
- [Top Agentic LLM Models for 2026](https://www.adaline.ai/blog/top-agentic-llm-models-frameworks-for-2026)
- [Best Open Source LLM for Agent Workflow](https://www.siliconflow.com/articles/en/best-open-source-LLM-for-Agent-Workflow)

### Inference & Deployment
- [vLLM vs TGI vs Ollama Comparison](https://compute.hivenet.com/post/vllm-vs-tgi-vs-tensorrt-llm-vs-ollama)
- [State of LLM Serving 2026](https://thecanteenapp.com/analysis/2026/01/03/inference-serving-landscape.html)
- [Choosing the Right Inference Framework](https://bentoml.com/llm/getting-started/choosing-the-right-inference-framework)
- [Local LLM on 24GB GPUs](https://intuitionlabs.ai/articles/local-llm-deployment-24gb-gpu-optimization)
- [Best GPU for LLM 2026](https://www.fluence.network/blog/best-gpu-for-llm/)

### Cost Optimization
- [Intelligent LLM Routing](https://www.requesty.ai/blog/intelligent-llm-routing-in-enterprise-ai-uptime-cost-efficiency-and-model)
- [10 AI Cost Optimization Strategies 2026](https://www.aipricingmaster.com/blog/10-AI-Cost-Optimization-Strategies-for-2026)
- [LLM Routing Guide](https://medium.com/@kamyashah2018/the-complete-guide-to-llm-routing-5-ai-gateways-transforming-production-ai-infrastructure-b5c68ee6d641)
- [Multi-Model AI Cost Reduction](https://www.swfte.com/blog/intelligent-llm-routing-multi-model-ai)

### Coding & Development
- [Best LLMs for Coding 2026](https://www.builder.io/blog/best-llms-for-coding)
- [Best AI Models for Coding - JetBrains](https://blog.jetbrains.com/ai/2026/02/the-best-ai-models-for-coding-accuracy-integration-and-developer-fit/)
