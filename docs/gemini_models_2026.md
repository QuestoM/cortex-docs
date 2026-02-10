# Google Gemini API Models -- February 2026

Research date: 2026-02-09

Sources:
- https://ai.google.dev/gemini-api/docs/models
- https://ai.google.dev/gemini-api/docs/pricing
- https://ai.google.dev/gemini-api/docs/gemini-3
- https://blog.google/products/gemini/gemini-3/
- https://developers.googleblog.com/new-gemini-api-updates-for-gemini-3/

---

## Model Overview (Recommended for corteX)

| Model ID | Generation | Status | Best For |
|---|---|---|---|
| `gemini-3-pro-preview` | 3.0 | Preview | Orchestration, complex reasoning, agentic tasks |
| `gemini-3-flash-preview` | 3.0 | Preview | Fast worker tasks, high-throughput, cost-sensitive |
| `gemini-3-pro-image-preview` | 3.0 | Preview | Image generation and understanding |
| `gemini-2.5-pro` | 2.5 | Stable/GA | Complex reasoning fallback (production-stable) |
| `gemini-2.5-flash` | 2.5 | Stable/GA | Fast general-purpose (production-stable) |
| `gemini-2.5-flash-lite` | 2.5 | Stable/GA | Cheapest option, high throughput |
| `gemini-2.0-flash` | 2.0 | Deprecated Mar 31 2026 | DO NOT USE -- retiring soon |
| `gemini-2.0-flash-lite` | 2.0 | Deprecated Mar 31 2026 | DO NOT USE -- retiring soon |

---

## corteX SDK Recommendations

### Orchestrator / Reasoning Model: `gemini-3-pro-preview`

- Tops the LMArena leaderboard with 1501 Elo, outperforms 2.5 Pro on every major benchmark.
- 1M token context window -- can ingest entire codebases, long documents, video, audio.
- Full tool calling, code execution, search grounding, structured outputs, and thinking.
- Supports `thinking_level: "high"` (default) for maximum reasoning depth.
- Thought signatures must be preserved across multi-turn function calling conversations.

### Worker / Fast Model: `gemini-3-flash-preview`

- Pro-level intelligence at significantly lower cost and latency.
- Same 1M token context window as Pro.
- Use `thinking_level: "low"` or `"minimal"` for maximum throughput on simple tasks.
- Free tier available for development and testing.

### Stable Fallback (if preview models are unavailable):

- Orchestrator: `gemini-2.5-pro`
- Worker: `gemini-2.5-flash`

---

## Context Windows

| Model ID | Input Tokens | Output Tokens |
|---|---|---|
| `gemini-3-pro-preview` | 1,000,000 | 65,536 |
| `gemini-3-flash-preview` | 1,000,000 | 65,536 |
| `gemini-3-pro-image-preview` | 65,536 | 32,768 |
| `gemini-2.5-pro` | 1,000,000 | 65,536 |
| `gemini-2.5-flash` | 1,000,000 | 65,536 |
| `gemini-2.5-flash-lite` | 1,000,000 | 65,536 |

---

## Pricing (per 1M tokens, USD, Pay-as-you-go tier)

### Gemini 3 Series

| Model | Input (<=200K ctx) | Input (>200K ctx) | Output (<=200K) | Output (>200K) | Free Tier |
|---|---|---|---|---|---|
| `gemini-3-pro-preview` | $2.00 | $4.00 | $12.00 | $18.00 | No |
| `gemini-3-flash-preview` | $0.50 | $1.00 (audio) | $3.00 | $3.00 | Yes |
| `gemini-3-pro-image-preview` | $2.00 (text) | -- | $0.134/image | -- | No |

### Gemini 2.5 Series

| Model | Input (<=200K) | Input (>200K) | Output (<=200K) | Output (>200K) | Free Tier |
|---|---|---|---|---|---|
| `gemini-2.5-pro` | $1.25 | $2.50 | $10.00 | $15.00 | Yes |
| `gemini-2.5-flash` | $0.30 | $1.00 (audio) | $2.50 | $2.50 | Yes |
| `gemini-2.5-flash-lite` | $0.10 | $0.30 | $0.40 | $0.40 | Yes |

### Batch API (50% discount on all models)

Batch pricing is exactly half the standard pricing for all models.

### Cost Comparison (typical orchestrator call, 10K in / 2K out)

| Model | Estimated Cost |
|---|---|
| `gemini-3-pro-preview` | ~$0.044 |
| `gemini-2.5-pro` | ~$0.033 |
| `gemini-3-flash-preview` | ~$0.011 |
| `gemini-2.5-flash` | ~$0.008 |
| `gemini-2.5-flash-lite` | ~$0.002 |

---

## Capabilities Matrix

| Capability | 3 Pro | 3 Flash | 3 Pro Image | 2.5 Pro | 2.5 Flash | 2.5 Flash-Lite |
|---|---|---|---|---|---|---|
| Text generation | Yes | Yes | Yes | Yes | Yes | Yes |
| Vision (image input) | Yes | Yes | Yes | Yes | Yes | Yes |
| Video input | Yes | Yes | No | Yes | Yes | Yes |
| Audio input | Yes | Yes | No | Yes | Yes | Yes |
| PDF input | Yes | Yes | No | Yes | Yes | Yes |
| Image generation | No | No | Yes | No | Yes* | No |
| Thinking / reasoning | Yes | Yes | Yes | Yes | Yes | No |
| Function calling | Yes | Yes | Yes | Yes | Yes | Yes |
| Code execution | Yes | Yes | No | Yes | Yes | No |
| Google Search grounding | Yes | Yes | No | Yes | Yes | No |
| File search (RAG) | Yes | Yes | No | Yes | Yes | No |
| URL context | Yes | Yes | No | Yes | Yes | No |
| Structured outputs (JSON) | Yes | Yes | Yes | Yes | Yes | Yes |
| Context caching | Yes | Yes | No | Yes | Yes | Yes |
| Batch API | Yes | Yes | Yes | Yes | Yes | Yes |
| Maps grounding | No | No | No | Yes | Yes | No |

*via `gemini-2.5-flash-image` variant

---

## Key Parameters for Gemini 3

### thinking_level (controls reasoning depth)

| Value | Available On | Use Case |
|---|---|---|
| `"high"` (default) | Pro, Flash | Maximum reasoning -- orchestration, complex tasks |
| `"medium"` | Flash only | Balanced reasoning and speed |
| `"low"` | Pro, Flash | Minimal reasoning -- simple tasks, high throughput |
| `"minimal"` | Flash only | Near-zero thinking -- fastest possible |

### media_resolution (controls vision token usage)

| Value | Tokens/Image | Best For |
|---|---|---|
| `media_resolution_high` | ~1120 | Detailed image analysis, PDFs with small text |
| `media_resolution_medium` | ~560 | PDFs (recommended default) |
| `media_resolution_low` | ~70/frame | Video processing, cost optimization |

### Important Configuration Notes

- **Temperature**: Keep at default 1.0 for Gemini 3. Lowering it may cause output looping.
- **Thought signatures**: Gemini 3 uses encrypted thought signatures for multi-turn reasoning. SDKs handle this automatically, but custom implementations must preserve and return `thought_signature` fields exactly as received in function calling flows.
- **Prompting style**: Gemini 3 responds best to direct, clear instructions. Unlike 2.5, chain-of-thought prompt engineering is unnecessary and may hurt performance since the model handles reasoning internally.

---

## Specialized Models

| Model ID | Purpose |
|---|---|
| `gemini-2.5-flash-image` | Image generation via 2.5 Flash |
| `gemini-2.5-flash-native-audio-preview-12-2025` | Live bidirectional audio (Live API) |
| `gemini-2.5-flash-preview-tts` | Text-to-speech |
| `gemini-2.5-pro-preview-tts` | Text-to-speech (higher quality) |
| `gemini-embedding-001` | Text embeddings ($0.15/1M tokens) |
| `imagen-4` | Standalone image generation ($0.02-$0.06/image) |
| `veo-3.1` | Video generation ($0.15-$0.60/video) |
| `gemma-3` | Open-weight model (free) |
| `gemma-3n` | Open-weight nano model (free) |

---

## Migration Notes for corteX SDK

1. **Replace any `gemini-2.0-*` references immediately** -- these models shut down March 31, 2026.
2. **Default orchestrator**: Switch from `gemini-2.5-pro` to `gemini-3-pro-preview` for best reasoning.
3. **Default worker**: Switch from `gemini-2.5-flash` to `gemini-3-flash-preview` for best speed/quality ratio.
4. **Keep 2.5 models as fallbacks** in case preview availability is limited.
5. **Update prompt patterns**: Remove chain-of-thought scaffolding in prompts for Gemini 3 -- the model handles reasoning internally via thinking tokens.
6. **Implement thought signature handling**: If using raw API calls (not SDK), ensure `thought_signature` fields are preserved in multi-turn conversations.
7. **Budget impact**: Gemini 3 Pro is ~60% more expensive than 2.5 Pro. Consider using 3 Flash (`thinking_level: "low"`) for simpler worker tasks to offset costs.
