# Context Summarizer API Reference

## Module: `corteX.engine.context_summarizer`

L2/L3 LLM-based summarization for the Cortical Context Engine (CCE). L2 produces narrative summaries at 10:1-20:1 compression, L3 produces structured JSON digests at 50:1-100:1 compression. This is a pure prompt-builder module -- it generates prompts and parses responses but never makes LLM calls itself. Rate-limit aware via `SummarizationRateLimiter` (P2-16).

---

## Enums

### `SummarizationLevel`

**Type**: `IntEnum`

Progressive summarization levels matching the CCE `CompressionLevel`.

| Value | Name | Description |
|-------|------|-------------|
| `0` | `L0_RAW` | Raw entries, no compression |
| `1` | `L1_KEYWORDS` | Keyword extraction only |
| `2` | `L2_SUMMARY` | Narrative summary (10:1-20:1 compression) |
| `3` | `L3_DIGEST` | Structured JSON digest (50:1-100:1 compression) |

---

## Configuration

### `SummarizationConfig`

**Type**: `@dataclass`

Configuration for the summarization pipeline, including trigger thresholds, batching, and rate limiting.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `l2_trigger_entries` | `int` | `20` | Number of L1 entries required to trigger L2 summarization |
| `l3_trigger_summaries` | `int` | `10` | Number of L2 summaries required to trigger L3 digest |
| `l2_max_summary_tokens` | `int` | `200` | Maximum token length for each L2 summary |
| `l3_max_digest_tokens` | `int` | `500` | Maximum token length for the L3 digest |
| `l2_batch_size` | `int` | `5` | Number of entries per L2 summarization batch |
| `preserve_recent_n` | `int` | `5` | Number of recent entries to exclude from summarization (keep in hot context) |
| `max_calls_per_minute` | `int` | `10` | Rate limit: max LLM summarization calls per time window |
| `max_calls_per_session` | `int` | `200` | Rate limit: max LLM summarization calls per session |
| `rate_limit_window_seconds` | `float` | `60.0` | Duration of the rate limit sliding window in seconds |

#### Example

```python
from corteX.engine.context_summarizer import SummarizationConfig

config = SummarizationConfig(
    l2_trigger_entries=15,
    l3_trigger_summaries=8,
    l2_batch_size=3,
    preserve_recent_n=3,
    max_calls_per_minute=5,
)
```

---

## Classes

### `L2Summarizer`

Generates LLM prompts for L2 (narrative) summarization and parses LLM responses into clean summaries.

#### Constructor

```python
L2Summarizer()
```

No parameters. Stateless prompt builder.

#### Methods

##### `build_summary_prompt`

```python
def build_summary_prompt(self, entries: List[str], max_tokens: int = 200) -> str
```

Create a prompt asking the LLM to summarize context entries into a concise narrative.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `entries` | `List[str]` | (required) | Context entries to summarize |
| `max_tokens` | `int` | `200` | Target maximum token length for the summary |

**Returns**: `str` -- Formatted prompt string, or empty string if `entries` is empty.

##### `parse_summary_response`

```python
def parse_summary_response(self, response: str) -> str
```

Extract a clean summary from an LLM response by stripping common preamble patterns (e.g., "Here is a summary:", "Summary:", "The summary follows:").

| Parameter | Type | Description |
|-----------|------|-------------|
| `response` | `str` | Raw LLM response text |

**Returns**: `str` -- Cleaned summary text, or empty string if response is empty.

##### `build_batch_summary_prompt`

```python
def build_batch_summary_prompt(self, entries: List[str], batch_size: int = 5) -> List[str]
```

Split entries into batches and generate one prompt per batch.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `entries` | `List[str]` | (required) | Context entries to batch |
| `batch_size` | `int` | `5` | Number of entries per batch |

**Returns**: `List[str]` -- List of prompt strings, one per batch.

##### `estimate_compression_ratio`

```python
def estimate_compression_ratio(self, original_tokens: int, summary_tokens: int) -> float
```

Calculate the compression ratio (original / summary).

| Parameter | Type | Description |
|-----------|------|-------------|
| `original_tokens` | `int` | Token count of the original entries |
| `summary_tokens` | `int` | Token count of the summary |

**Returns**: `float` -- Compression ratio. Returns `0.0` if `summary_tokens <= 0`.

#### Example

```python
from corteX.engine.context_summarizer import L2Summarizer

summarizer = L2Summarizer()

entries = [
    "User asked to build a REST API for inventory management.",
    "Agent chose Flask framework, created app.py with /items endpoint.",
    "Error: ModuleNotFoundError for flask. Installed with pip.",
    "Tests passed. Added /items/{id} GET endpoint.",
    "User requested pagination support.",
]

# Single prompt
prompt = summarizer.build_summary_prompt(entries, max_tokens=150)

# Batched prompts (2 entries per batch)
prompts = summarizer.build_batch_summary_prompt(entries, batch_size=2)
# len(prompts) == 3

# After LLM call
raw_response = "Here is a concise summary: The agent built a REST API..."
clean = summarizer.parse_summary_response(raw_response)
# "The agent built a REST API..."
```

---

### `L3DigestBuilder`

Generates LLM prompts for L3 structured digest extraction and parses JSON responses.

#### Constructor

```python
L3DigestBuilder()
```

No parameters. Stateless prompt builder.

#### Methods

##### `build_digest_prompt`

```python
def build_digest_prompt(self, summaries: List[str], task_context: str) -> str
```

Create a prompt requesting a structured JSON digest from L2 summaries.

| Parameter | Type | Description |
|-----------|------|-------------|
| `summaries` | `List[str]` | L2 summary strings to digest |
| `task_context` | `str` | Description of the overall task for context |

**Returns**: `str` -- Formatted digest prompt, or empty string if `summaries` is empty.

##### `parse_digest_response`

```python
def parse_digest_response(self, response: str) -> Dict[str, Any]
```

Parse an LLM JSON response into a structured digest dictionary. Handles markdown code fences, extracts JSON, validates required keys, and normalizes list fields.

| Parameter | Type | Description |
|-----------|------|-------------|
| `response` | `str` | Raw LLM response containing JSON |

**Returns**: `Dict[str, Any]` -- Structured digest with keys:

| Key | Type | Description |
|-----|------|-------------|
| `key_decisions` | `List[str]` | Important decisions made (capped at 20) |
| `tools_used` | `List[str]` | Tools invoked during the session (capped at 30) |
| `errors_encountered` | `List[str]` | Errors and how they were resolved (capped at 15) |
| `progress_toward_goal` | `str` | Percentage or qualitative progress description |
| `open_questions` | `List[str]` | Unresolved questions (capped at 10) |

Returns `_EMPTY_DIGEST` (all empty) if parsing fails.

##### `merge_digests`

```python
def merge_digests(self, old_digest: Dict[str, Any], new_digest: Dict[str, Any]) -> Dict[str, Any]
```

Merge two digests. New digest takes precedence for progress. Lists are deduplicated (case-insensitive) and capped at per-field limits.

| Parameter | Type | Description |
|-----------|------|-------------|
| `old_digest` | `Dict[str, Any]` | Previous digest |
| `new_digest` | `Dict[str, Any]` | Newer digest to merge in |

**Returns**: `Dict[str, Any]` -- Merged digest with deduplicated, capped lists.

#### Example

```python
from corteX.engine.context_summarizer import L3DigestBuilder

builder = L3DigestBuilder()

summaries = [
    "Built REST API with Flask. Fixed import error. Added pagination.",
    "Implemented authentication with JWT. Added rate limiting middleware.",
]

prompt = builder.build_digest_prompt(summaries, task_context="Build a production API")

# After LLM call returns JSON
response = '{"key_decisions": ["Flask framework", "JWT auth"], "tools_used": ["pip"], ...}'
digest = builder.parse_digest_response(response)

# Incremental merging across sessions
old = {"key_decisions": ["Flask framework"], "tools_used": ["pip"], ...}
new = {"key_decisions": ["Added Redis cache"], "tools_used": ["docker"], ...}
merged = builder.merge_digests(old, new)
```

---

### `SummarizationPipeline`

Orchestrates the full L2/L3 summarization lifecycle: trigger detection, prompt generation, response processing, compression statistics, and rate-limit-aware fallback to truncation.

#### Constructor

```python
SummarizationPipeline(config: Optional[SummarizationConfig] = None)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `config` | `Optional[SummarizationConfig]` | `None` | Pipeline configuration. Uses `SummarizationConfig()` defaults if not provided. |

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `config` | `SummarizationConfig` | The active configuration |
| `rate_limiter` | `SummarizationRateLimiter` | The rate limiter instance (read-write) |

#### Methods

##### `should_summarize_l2`

```python
def should_summarize_l2(self, l1_entries_count: int) -> bool
```

Check whether the number of L1 entries has reached the L2 trigger threshold.

| Parameter | Type | Description |
|-----------|------|-------------|
| `l1_entries_count` | `int` | Current number of L1 (raw/keyword) entries |

**Returns**: `bool` -- `True` if `l1_entries_count >= config.l2_trigger_entries`.

##### `should_summarize_l3`

```python
def should_summarize_l3(self, l2_summaries_count: int) -> bool
```

Check whether the number of L2 summaries has reached the L3 trigger threshold.

| Parameter | Type | Description |
|-----------|------|-------------|
| `l2_summaries_count` | `int` | Current number of L2 summaries |

**Returns**: `bool` -- `True` if `l2_summaries_count >= config.l3_trigger_summaries`.

##### `build_l2_prompts`

```python
def build_l2_prompts(self, entries: List[str]) -> List[str]
```

Generate batched L2 prompts, excluding the most recent `preserve_recent_n` entries from summarization to keep them in hot context.

| Parameter | Type | Description |
|-----------|------|-------------|
| `entries` | `List[str]` | All L1 context entries |

**Returns**: `List[str]` -- List of L2 prompt strings. Empty if not enough entries after preserving recent ones.

##### `process_l2_responses`

```python
def process_l2_responses(self, prompts_and_responses: List[Tuple[str, str]]) -> List[str]
```

Process LLM responses into clean summaries and track compression statistics.

| Parameter | Type | Description |
|-----------|------|-------------|
| `prompts_and_responses` | `List[Tuple[str, str]]` | List of (prompt, LLM response) pairs |

**Returns**: `List[str]` -- Cleaned L2 summary strings.

##### `build_l3_prompt`

```python
def build_l3_prompt(self, summaries: List[str], context: str) -> str
```

Generate an L3 structured digest prompt from L2 summaries.

| Parameter | Type | Description |
|-----------|------|-------------|
| `summaries` | `List[str]` | L2 summary strings |
| `context` | `str` | Task context description |

**Returns**: `str` -- L3 digest prompt string.

##### `process_l3_response`

```python
def process_l3_response(self, response: str) -> Dict[str, Any]
```

Parse an LLM response into a structured L3 digest dictionary.

| Parameter | Type | Description |
|-----------|------|-------------|
| `response` | `str` | Raw LLM response containing JSON |

**Returns**: `Dict[str, Any]` -- Structured digest (see `L3DigestBuilder.parse_digest_response`).

##### `summarize_or_truncate`

```python
def summarize_or_truncate(self, entries: List[str], max_chars: int = 2000) -> str
```

Rate-limit-aware fallback. If the rate limiter blocks the call, returns a truncated concatenation of entries. Otherwise returns an empty string (indicating a real LLM summarization call is allowed).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `entries` | `List[str]` | (required) | Entries to potentially truncate |
| `max_chars` | `int` | `2000` | Maximum character length for truncated output |

**Returns**: `str` -- Truncated text if rate-limited, empty string if LLM call is allowed.

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Return pipeline statistics including summaries generated, compression ratios, truncation fallbacks, and rate limiter state.

**Returns**: `Dict[str, Any]` with keys: `l2_summaries_generated`, `l3_digests_generated`, `total_entries_summarized`, `average_compression_ratio`, `compression_samples`, `truncation_fallbacks`, `rate_limiter`, `config`.

#### Example

```python
from corteX.engine.context_summarizer import SummarizationPipeline, SummarizationConfig

pipeline = SummarizationPipeline(SummarizationConfig(
    l2_trigger_entries=10,
    l2_batch_size=3,
    preserve_recent_n=2,
))

entries = ["Step 1: ...", "Step 2: ...", ..., "Step 15: ..."]

# Check if summarization is needed
if pipeline.should_summarize_l2(len(entries)):
    # Rate limit check with fallback
    fallback = pipeline.summarize_or_truncate(entries)
    if fallback:
        # Rate limited -- use truncated text
        compressed = fallback
    else:
        # Generate prompts and send to LLM
        prompts = pipeline.build_l2_prompts(entries)
        # ... send prompts to LLM, collect responses ...
        summaries = pipeline.process_l2_responses(list(zip(prompts, responses)))

        # Check for L3
        if pipeline.should_summarize_l3(len(summaries)):
            l3_prompt = pipeline.build_l3_prompt(summaries, "Build REST API")
            digest = pipeline.process_l3_response(l3_response)

print(pipeline.get_stats())
```

---

## Re-exported: `SummarizationRateLimiter`

```python
from corteX.engine.summarization_limiter import SummarizationRateLimiter
```

Rate limiter for LLM summarization calls. Tracks calls per time window and per session to prevent API quota exhaustion. Provides exponential backoff for 429 errors and truncation fallback.

### Constructor

```python
SummarizationRateLimiter(
    max_calls_per_minute: int = 10,
    max_calls_per_session: int = 200,
    window_seconds: float = 60.0,
)
```

### Key Methods

| Method | Signature | Description |
|--------|-----------|-------------|
| `can_call` | `(now: Optional[float]) -> bool` | Check if a call is allowed right now |
| `record_call` | `(now: Optional[float]) -> None` | Record that a call was made |
| `record_rate_limit_error` | `(now: Optional[float]) -> float` | Record a 429 error; returns backoff seconds |
| `backoff_seconds` | `() -> float` | Current exponential backoff (2^n, capped at 60s) |
| `reset_backoff` | `() -> None` | Reset the backoff counter after success |
| `get_stats` | `() -> Dict[str, Any]` | Rate limiter statistics |

### Key Properties

| Property | Type | Description |
|----------|------|-------------|
| `calls_remaining_in_window` | `int` | Calls still allowed in the current time window |
| `calls_remaining_in_session` | `int` | Calls still allowed in the session |
| `is_rate_limited` | `bool` | Whether currently in backoff period |

---

## Re-exported: `truncate_entries`

```python
from corteX.engine.summarization_limiter import truncate_entries
```

```python
def truncate_entries(entries: List[str], max_total_chars: int = 2000) -> str
```

Fallback truncation when summarization is rate-limited. Concatenates entries (each truncated to 200 chars) with pipe separators and cuts to `max_total_chars`.

---

## Module Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `_L2_SYSTEM` | (string) | System prompt for L2 summarization LLM calls |
| `_L2_TEMPLATE` | (string) | Prompt template for L2 narrative summaries |
| `_L3_TEMPLATE` | (string) | Prompt template for L3 structured JSON digests |
| `_DIGEST_KEYS` | `frozenset` | Required keys in L3 digest output |
| `_EMPTY_DIGEST` | `Dict` | Empty digest template used as fallback |
| `_LIST_LIMITS` | `Dict[str, int]` | Per-field cap for digest list fields |

---

## Performance Notes

- All classes are stateless prompt builders -- no I/O, no LLM calls
- The `SummarizationPipeline` tracks statistics but performs no blocking operations
- Typical prompt build time: <1ms
- Rate limiter uses `time.monotonic()` for reliable timing
- Truncation fallback is the "fatigue" mechanism: when API budget is low, the system gracefully degrades to simpler compression

---

## Error Handling

- Empty inputs always return empty outputs (no exceptions)
- Malformed JSON in L3 responses falls back to `_EMPTY_DIGEST`
- Non-dict JSON responses are rejected gracefully
- List fields that contain non-list values are auto-converted to single-element lists
- `progress_toward_goal` is auto-converted to string if not already

---

## See Also

- [Cortical Context Engine API](./context.md) -- The CCE that uses this module for context compression
- [Semantic Scorer API](./semantic-scorer.md) -- TF-IDF scoring used alongside summarization
- [Content Prediction API](./content-prediction.md) -- Related prompt-builder pattern for prediction
- [Brain State Injector API](./brain-state-injector.md) -- Another prompt compilation module
