# Structured Output

The Structured Output system extracts self-assessment signals from LLM responses -- confidence, difficulty, escalation needs, and per-tool confidence -- without making additional LLM calls. It uses the existing response, keeping latency and cost at zero.

## What It Does

Three components work together:

1. **StructuredOutputInjector**: Appends instructions to the system prompt asking the LLM to include a `cortex-signals` JSON block in its response
2. **StructuredOutputParser**: Extracts signals using multi-strategy fallback (cortex block -> generic JSON -> keyword scan -> defaults)
3. **SignalAggregator**: Combines LLM self-assessed signals with brain signals (surprise, calibration ECE, population quality) into a unified quality assessment

## Why It Matters

LLMs can self-assess their own output quality if asked correctly. This is cheaper and faster than running a separate evaluation call. The signals feed directly into the brain engine's decision-making:

- **Low confidence + high difficulty** triggers System 2 escalation
- **High confidence + low agreement with brain signals** triggers output verification
- **Escalation requests** from the LLM itself get routed to human review

## How It Works

### Signal Injection

The injector generates instruction text appended to the system prompt:

```python
from corteX.engine.structured_output import StructuredOutputInjector

injector = StructuredOutputInjector(verbosity="full")

# Generate instruction to append to system prompt
instruction = injector.generate_instruction(
    tool_names=["code_interpreter", "browser", "search"]
)
# Tells the LLM to include a cortex-signals JSON block
```

Two verbosity modes: `"full"` (detailed guidelines) and `"compact"` (one-line instruction).

### Signal Extraction

The parser uses three fallback strategies:

```python
from corteX.engine.structured_output import StructuredOutputParser

parser = StructuredOutputParser()

# Parse from LLM response text
signals = parser.parse(response_text)
# StructuredSignals(confidence=0.85, difficulty=MEDIUM, ...)

# Strip the signal block from user-facing content
clean_response = parser.strip_signal_block(response_text)
```

Extraction strategies (tried in order):
1. **cortex-signals block**: Fenced code block with `cortex-signals` language tag
2. **Generic JSON**: Any JSON object containing a `confidence` key
3. **Keyword scan**: Regex patterns for confidence, difficulty, escalation keywords

### Signal Aggregation

The aggregator combines LLM signals with brain-internal signals:

```python
from corteX.engine.structured_output import SignalAggregator

aggregator = SignalAggregator(
    llm_weight=0.30,
    population_weight=0.30,
    calibration_weight=0.25,
    surprise_weight=0.15,
)

result = aggregator.aggregate(
    signals=signals,
    prediction_surprise=0.3,
    calibration_ece=0.12,
    population_quality=0.8,
)
# AggregatedQuality(
#   composite_confidence=0.72,
#   recommended_action="proceed",
#   escalation_urgency=0.0,
# )
```

### Recommended Actions

The aggregator outputs one of six actions based on composite signals:

| Action | Trigger |
|--------|---------|
| `proceed_confident` | High confidence + high agreement |
| `proceed` | Normal confidence |
| `verify_output` | Low agreement between LLM and brain signals |
| `retry_with_stronger_model` | Very low composite confidence |
| `escalate_to_system2` | Medium urgency or hard task with low confidence |
| `escalate_to_human` | High urgency |

## When It Activates

- **Before every LLM call**: Injector appends signal instructions to the system prompt
- **After every LLM response**: Parser extracts signals from the response text
- **After extraction**: Aggregator combines with brain signals and recommends an action
- **Zero extra latency**: Everything happens within the existing request/response cycle

## API Reference

```python
from corteX.engine.structured_output import (
    StructuredOutputInjector,
    StructuredOutputParser,
    SignalAggregator,
    StructuredSignals,
    AggregatedQuality,
    DifficultyLevel,
)
```
