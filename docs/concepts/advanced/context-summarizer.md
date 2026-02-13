# Context Summarization

The Context Summarization system provides progressive compression for long-running agent sessions. It generates prompts for LLM-based summarization at two levels -- L2 (narrative summaries) and L3 (structured digests) -- but never makes LLM calls itself. The SDK pipeline handles inference.

## What It Does

Two compression levels beyond the CCE's built-in L0 (raw) and L1 (keyword extraction):

| Level | Type | Compression | Output |
|-------|------|-------------|--------|
| **L2 Summary** | LLM narrative | 10:1 to 20:1 | Concise paragraph preserving key decisions, errors, tools, progress |
| **L3 Digest** | Structured JSON | 50:1 to 100:1 | `key_decisions`, `tools_used`, `errors_encountered`, `progress_toward_goal`, `open_questions` |

## How It Works

### L2 Summarization

The `L2Summarizer` batches context entries and generates prompts asking the LLM to compress them:

```python
from corteX.engine.context_summarizer import L2Summarizer

l2 = L2Summarizer()

# Build a prompt for summarizing 5 context entries
entries = [
    "Step 1: User asked to analyze sales data",
    "Step 2: Used search tool to find dataset",
    "Step 3: Code interpreter loaded CSV, got parse error",
    "Step 4: Fixed CSV encoding, loaded successfully",
    "Step 5: Generated summary statistics",
]
prompt = l2.build_summary_prompt(entries, max_tokens=200)

# After LLM responds, parse the summary
summary = l2.parse_summary_response(llm_response)
# "Analyzed sales data: searched for and loaded CSV dataset
#  (recovered from encoding error), generated summary statistics."
```

Batch mode splits large entry sets into groups:

```python
prompts = l2.build_batch_summary_prompt(entries, batch_size=5)
# One prompt per batch of 5 entries
```

### L3 Structured Digests

The `L3DigestBuilder` extracts structured information from L2 summaries:

```python
from corteX.engine.context_summarizer import L3DigestBuilder

l3 = L3DigestBuilder()

prompt = l3.build_digest_prompt(
    summaries=["Summary of steps 1-5...", "Summary of steps 6-10..."],
    task_context="Quarterly sales analysis for Q4 2025",
)

digest = l3.parse_digest_response(llm_response)
# {
#   "key_decisions": ["Used pandas for analysis", "Switched to UTF-8 encoding"],
#   "tools_used": ["search", "code_interpreter"],
#   "errors_encountered": ["CSV parse error due to encoding"],
#   "progress_toward_goal": "75% -- analysis complete, report pending",
#   "open_questions": ["Should we include YoY comparison?"],
# }
```

Digests merge incrementally -- new digests update old ones with deduplication and capped list sizes:

```python
merged = l3.merge_digests(old_digest, new_digest)
# Lists are deduped, progress takes the newer value
```

### SummarizationPipeline

The pipeline orchestrates L2/L3 together with trigger thresholds:

```python
from corteX.engine.context_summarizer import SummarizationPipeline

pipeline = SummarizationPipeline()

# Check if summarization should trigger
if pipeline.should_summarize_l2(l1_entries_count=25):
    prompts = pipeline.build_l2_prompts(entries)
    # Send prompts to LLM, collect responses
    summaries = pipeline.process_l2_responses(prompts_and_responses)

if pipeline.should_summarize_l3(l2_summaries_count=12):
    prompt = pipeline.build_l3_prompt(summaries, context="Sales analysis")
    digest = pipeline.process_l3_response(llm_response)

# Track compression statistics
stats = pipeline.get_stats()
# {"l2_summaries_generated": 5, "average_compression_ratio": 12.3, ...}
```

## Design Principles

- **Pure prompt-builder**: No LLM calls inside the module -- fully testable without API keys
- **Preserves recent context**: The most recent N entries are never summarized (configurable via `preserve_recent_n`)
- **Incremental**: Each summarization pass processes only new entries, not the full history
- **Configurable thresholds**: L2 triggers at 20 L1 entries, L3 triggers at 10 L2 summaries (both configurable)

## When It Activates

- **L2 trigger**: When L1 keyword entries exceed the threshold (default: 20)
- **L3 trigger**: When L2 summaries exceed the threshold (default: 10)
- **Merge**: When a new L3 digest is created, it merges with the previous one
- **Pipeline stats**: Continuously tracked for observability

## API Reference

```python
from corteX.engine.context_summarizer import (
    SummarizationPipeline,
    SummarizationConfig,
    L2Summarizer,
    L3DigestBuilder,
    SummarizationLevel,
)
```
