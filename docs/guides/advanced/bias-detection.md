# Detect When the LLM Ignores Your Instructions

LLMs have strong pre-training defaults - they tend to produce markdown when you asked for JSON, write long responses when you asked for brevity, or inject training knowledge when you provided specific facts. The bias detector catches these instruction overrides with confidence scores.

---

## Quick example

```python
from corteX.sdk import Engine

engine = Engine()
agent = engine.create_agent(name="test", system_prompt="You are helpful.")
session = agent.start_session(user_id="user-1")

result = session.detect_bias(
    instructions=["Respond in JSON format", "Keep response under 50 words"],
    response="The answer to your question is that Python is a wonderful programming language that was created by Guido van Rossum in 1991. It has become one of the most popular languages in the world.",
)

print(f"Has bias: {result['has_bias']}")
print(f"Compliance: {result['overall_compliance']:.0%}")
for d in result["detections"]:
    print(f"  [{d['bias_type']}] confidence={d['confidence']:.0%}: {d['evidence']}")
```

## What it detects

| Bias Type | What it checks | Example |
|---|---|---|
| **Format** | JSON, bullet points, table, XML, numbered list | Asked for JSON, got prose |
| **Length** | Word count vs instructions (max/min words, brevity) | Asked for < 50 words, got 200 |
| **Style** | Formal/casual/technical tone matching | Asked for casual, got academic |
| **Factual** | Response uses provided facts vs training knowledge | Provided Q3 revenue, LLM made up numbers |

## Factual grounding check

The most powerful check verifies the LLM used your provided facts:

```python
result = session.detect_bias(
    instructions=["Use only the provided facts to answer"],
    response="The company's revenue was $10M last quarter with 500 employees.",
    context_facts=[
        "Acme Corp revenue was $5.2M in Q3 2025",
        "Acme Corp has 127 employees",
    ],
)

# Detects factual bias: response contains numbers ($10M, 500)
# not present in the context facts
for d in result["detections"]:
    if d["bias_type"] == "factual":
        print(f"Fabrication detected: {d['evidence']}")
        print(f"Suggestion: {d['suggestion']}")
```

## Use in a production pipeline

Check every LLM response before returning to the user:

```python
async def safe_run(session, message, instructions, facts=None):
    response = await session.run(message)

    bias = session.detect_bias(
        instructions=instructions,
        response=response.content,
        context_facts=facts,
    )

    if bias["has_bias"] and bias["max_confidence"] > 0.7:
        # High-confidence bias detected - retry or flag
        logger.warning(
            "Bias detected: %s",
            [d["bias_type"] for d in bias["detections"]],
        )
        # Option 1: Retry with stronger instructions
        # Option 2: Return with a warning flag
        # Option 3: Fall back to a templated response

    return response
```

!!! tip "Combine with context pinning"
    If bias detection repeatedly catches format non-compliance, pin the format instruction with `session.pin_instruction()` to reinforce it in long conversations.

---

## Next steps

- [Pin Critical Instructions](context-pinning.md) - reinforce instructions the LLM keeps ignoring
- [Optimize Context Usage](span-tracking.md) - ensure your facts are actually reaching the LLM
