# Optimize Context Usage with Span Tracking

Not everything you put in the context window gets used. Span tracking analyzes which context chunks the LLM actually referenced in its response - using keyword overlap, entity matching, and structural references. No extra LLM call required.

---

## Analyze a response

```python
from corteX.sdk import Engine

engine = Engine()
agent = engine.create_agent(name="test", system_prompt="You are helpful.")
session = agent.start_session(user_id="user-1")

# Define your context items (what you sent to the LLM)
context_items = [
    {
        "span_id": "kb-article-1",
        "span_type": "knowledge",
        "content": "Our refund policy allows returns within 30 days of purchase.",
        "tokens": 120,
    },
    {
        "span_id": "kb-article-2",
        "span_type": "knowledge",
        "content": "Enterprise plans include 24/7 phone support and a dedicated CSM.",
        "tokens": 85,
    },
    {
        "span_id": "user-history",
        "span_type": "memory",
        "content": "User previously asked about pricing tiers.",
        "tokens": 40,
    },
]

# Analyze which spans were used
result = session.analyze_span_usage(
    context_items,
    response="You can return your purchase within 30 days for a full refund.",
)

print(f"Utilization: {result['utilization_ratio']:.0%}")
print(f"Wasted tokens: {result['waste_tokens']}")
print(f"Used: {result['used_spans']}")
print(f"Unused: {result['unused_spans']}")
```

## Understanding the results

| Field | Meaning |
|---|---|
| `utilization_ratio` | Fraction of context tokens that were actually used (0.0 - 1.0) |
| `waste_tokens` | Total tokens in unused spans |
| `used_spans` | List of span_ids the LLM referenced |
| `unused_spans` | List of span_ids with no detected usage |
| `total_context_tokens` | Sum of all span token counts |

## How detection works

The tracker uses three heuristic signals (no extra LLM call):

1. **Keyword overlap** - shared significant words between span content and response
2. **Entity matching** - proper nouns, numbers, and domain terms that appear in both
3. **Structural references** - when the response echoes specific phrases or data points

A span is marked "used" if any signal exceeds the threshold.

## Optimize context packing

Use span tracking data to improve your context strategy:

```python
import collections

# Track usage over many turns
usage_stats = collections.Counter()

for turn in conversation_turns:
    result = session.analyze_span_usage(turn.context, turn.response)
    for span_id in result["unused_spans"]:
        usage_stats[span_id] += 1

# Find consistently unused context
for span_id, unused_count in usage_stats.most_common(10):
    print(f"{span_id}: unused {unused_count} times - consider removing")
```

!!! tip "Target 70%+ utilization"
    If your utilization ratio is consistently below 50%, you're wasting tokens on context the LLM ignores. Remove or summarize low-usage spans.

!!! tip "Pair with cost tracking"
    Each wasted token costs money. Multiply `waste_tokens` by your per-token cost to see the dollar impact.

---

## Next steps

- [Pin Critical Instructions](context-pinning.md) - ensure important context stays visible
- [Monitor Your Agent](observability.md) - track utilization trends over time
