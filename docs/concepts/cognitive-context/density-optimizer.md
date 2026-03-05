# Density Optimizer

The `DensityOptimizer` packs maximum semantic content per token by applying structured compression techniques. It converts verbose prose into compact representations - achieving 3-5x density improvement without losing actionable information. Every token in the context window should carry maximum meaning.

## How It Works

The optimizer applies four type-specific compression strategies in sequence, followed by domain abbreviations and redundancy removal.

### Strategy 1: Preference Compression

Converts natural language preference statements into compact key-value pairs:

| Before | After |
|--------|-------|
| "The user prefers PostgreSQL as their database" | `[PREF] database:PostgreSQL` |
| "Prefer Docker over VM deployment" | `[PREF] use:Docker (not VM)` |
| "The service should use port 8080" | `[REQ] service:port 8080` |

### Strategy 2: Tool Result Compression

Converts narrative tool output into structured pipe-delimited format. Lines matching the pattern "X is/was/contains Y" are transformed into `x:Y` pairs. Short results (under 15 lines) become pipe-separated; longer ones remain newline-separated.

### Strategy 3: Error Compression

Strips Python tracebacks down to the exception line, removing all intermediate frame lines. The result is prefixed with `[ERR]`. For non-traceback errors, it keeps only lines containing error-relevant keywords (`error`, `exception`, `failed`, `denied`, `not found`, `timeout`, `invalid`), limited to 5 lines.

### Strategy 4: Conversation Compression

Strips filler from conversation history, keeping only decision-relevant lines. Removes acknowledgments (`ok`, `sure`, `got it`, `sounds good`, etc.) and keeps lines containing action verbs (`decided`, `chose`, `created`, `deployed`, `configured`, etc.) or file path references. Short conversations (3 lines or fewer) are preserved as-is.

### Domain Abbreviations

After type-specific compression, common terms are replaced with standard abbreviations:

| Long Form | Short Form |
|-----------|------------|
| authentication | auth |
| configuration | config |
| environment | env |
| database | db |
| repository | repo |
| documentation | docs |
| implementation | impl |
| dependencies | deps |
| infrastructure | infra |

The full abbreviation table covers 25+ common software development terms.

### Redundancy Removal

Finally, duplicate sentences are detected via MD5 hashing and removed, keeping only the first occurrence.

## Key Features

- **Four compression strategies** applied in sequence for maximum density
- **25+ domain abbreviations** for common software development terms
- **Sentence-level deduplication** via content hashing
- **Custom rules support** - Pass additional `DensityRule` objects for domain-specific compression
- **Token savings tracking** - `get_stats()` reports total tokens saved and average savings per item
- **Non-destructive** - Works on `ScoredItem` copies, preserving original content
- **Zero external dependencies** - All compression is regex-based

## Integration

The `DensityOptimizer` is invoked by the `CognitiveContextPipeline` after items are selected and before final assembly. It operates on the list of `ScoredItem` objects, compressing each item's content while preserving its priority and metadata. The resulting token savings directly increase the number of items that can fit within the context window budget.

The `ContextQualityEngine`'s Information Density Index (IDI) measures the effectiveness of optimization - higher unique entity count per token indicates better density. The `ContextPyramid` handles resolution-level compression separately; the density optimizer provides within-resolution optimization.

## Usage Example

```python
from corteX.engine.cognitive.density_optimizer import (
    DensityOptimizer, ScoredItem
)

optimizer = DensityOptimizer()

items = [
    ScoredItem(
        item_id="traceback",
        content="""Traceback (most recent call last):
  File "main.py", line 42, in deploy
    service.start()
  File "service.py", line 15, in start
    self.connect()
ConnectionRefusedError: Connection refused on port 8080""",
        priority=0.8,
        tokens=50,
    ),
    ScoredItem(
        item_id="prefs",
        content="The user prefers PostgreSQL as their database. "
                "The application should use Docker for deployment.",
        priority=0.6,
        tokens=30,
    ),
    ScoredItem(
        item_id="chat",
        content="Ok, got it.\nSure, I'll do that.\n"
                "I decided to use blue-green deployment.\n"
                "Created the staging environment.\n"
                "Sounds good, thanks.",
        priority=0.5,
        tokens=40,
    ),
]

optimized = optimizer.optimize(items)

# Traceback compressed to: "[ERR] ConnectionRefusedError: conn refused on port 8080"
# Preferences compressed to: "[PREF] db:PostgreSQL. [REQ] app:Docker for deployment"
# Conversation compressed to: "decided to use blue-green deployment.\nCreated the staging env"

# Check compression stats
stats = optimizer.get_stats()
print(f"Tokens saved: {stats['tokens_saved']}")
print(f"Items optimized: {stats['items_optimized']}")
print(f"Avg savings per item: {stats['avg_savings']:.0f} tokens")

# Measure compression gain
gain = optimizer.estimate_gain(
    original_tokens=120, optimized_tokens=35
)  # 3.4x denser
```

## See Also

- [Context Pyramid](pyramid.md) - Handles resolution-level compression (R0 to R3)
- [Context Quality Engine](quality-engine.md) - IDI dimension measures information density
- [Entanglement Graph](entanglement.md) - Entity extraction identifies what must be preserved during compression
- [Active Forgetting](active-forgetting.md) - Removes items entirely vs density optimizer which compresses them
