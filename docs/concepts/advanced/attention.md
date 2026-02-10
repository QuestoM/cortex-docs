# Attentional Filter

The Attentional Filter routes incoming messages to the appropriate processing depth. Not every user turn needs the same computational budget -- a routine continuation gets lighter processing, while a topic shift, error spike, or safety concern demands full resources.

## What It Does

The filter classifies every incoming message into five priority levels and assigns a corresponding processing budget:

| Priority | Processing | Model | Tools | Memory | Context |
|----------|-----------|-------|-------|--------|---------|
| **CRITICAL** | Maximum | Orchestrator | 20 calls | Full | Full |
| **FOREGROUND** | Full | Orchestrator | 10 calls | Full | Full |
| **BACKGROUND** | Moderate | Worker | 5 calls | Yes | Summary |
| **SUBCONSCIOUS** | Minimal | Worker | None | No | Minimal |
| **SUPPRESSED** | Skip | Worker | None | No | Minimal |

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Thalamic Gating"
    "We are very unconscious creatures... what I describe to you are things I AM conscious of, and that's only a small part of brain activity." -- Prof. Idan Segev

    The thalamus acts as a relay station, gating sensory information before it reaches the cortex. Most sensory input is processed subconsciously. Only deviations, errors, and goal-relevant signals break through to conscious awareness.

    The brain's attentional system operates on two principles:
    - **Change detection**: The Mismatch Negativity (MMN) response fires when reality deviates from the brain's implicit model of "what's normal"
    - **Threat detection**: The amygdala's fast path bypasses cortical processing for immediate threat detection -- this maps to the CRITICAL priority

    "Attention is not about seeing more -- it's about ignoring more. The brain is a massive filtering engine."

## How It Works

### Classification Algorithm

```python
from corteX.engine.attention import AttentionalFilter

af = AttentionalFilter(
    foreground_threshold=0.35,     # Delta above this -> FOREGROUND
    subconscious_threshold=0.10,   # Delta below this for N turns -> SUBCONSCIOUS
    subconscious_streak=4,         # N consecutive low-delta turns needed
)

priority = af.classify(
    message="Now let's switch to implementing the payment system",
    context={"tools_used": ["code_interpreter"], "error_count": 0},
)
# AttentionalPriority.FOREGROUND (topic shift detected)
```

The algorithm:
1. **Check CRITICAL triggers**: safety keywords, high error count (>=3), goal drift, error spikes
2. **Check SUPPRESSED**: habituation via AdaptationFilter (>70% of signals habituated)
3. **Compute delta** from previous state across: topic, behavior, tools, errors, quality
4. **If delta > foreground_threshold**: FOREGROUND (something changed)
5. **If delta < subconscious_threshold for N turns**: SUBCONSCIOUS (routine)
6. **Otherwise**: BACKGROUND

### Change Detector

The `ChangeDetector` tracks state across turns and detects five types of changes:

```python
from corteX.engine.attention import ChangeDetector

detector = ChangeDetector()
detector.record_state({"message": "...", "tools_used": ["web_search"], "quality": 0.7})
detector.record_state({"message": "...", "tools_used": ["code_interpreter"], "quality": 0.3})

changes = detector.detect_changes()
# [ChangeEvent(type="tool_shift", magnitude=1.0),
#  ChangeEvent(type="quality_drift", magnitude=0.4)]
```

| Change Type | What Changed | Threshold |
|-------------|-------------|-----------|
| `topic_shift` | Subject/intent keywords | Jaccard distance > 0.40 |
| `behavior_shift` | Message length, vocabulary, questions | Composite > 0.30 |
| `tool_shift` | Tools being used | Jaccard distance > 0.50 |
| `error_spike` | Error rate jumped | Normalized delta > 0.60 |
| `quality_drift` | Gradual quality decline over window | Normalized > 0.20 |

### Context Delta Compression

The `ContextDeltaCompressor` optimizes context by highlighting changes and compressing stable parts:

```python
from corteX.engine.attention import ContextDeltaCompressor

compressor = ContextDeltaCompressor()
compressed = compressor.compress(current_context, previous_context)
# Stable keys get [STABLE] prefix with short summary
# Changed keys get [CHANGED] prefix with full detail + delta description

highlights = compressor.highlight_changes(compressed, changes)
# "=== ATTENTION: 2 change(s) detected ===
#   [HIGH] tool_shift: Tool usage pattern changed (delta=1.00)
#   [MEDIUM] quality_drift: Quality drifted down from 0.70 to 0.30"
```

### Attentional Gate (Spotlight Model)

The `AttentionalGate` implements Posner's spotlight model with fixed-capacity focus:

```python
from corteX.engine.attention import AttentionalGate

gate = AttentionalGate(capacity=5)

# Update spotlight with relevance scores
gate.update_spotlight([
    ("auth_handler.py", 0.9),
    ("test_auth.py", 0.7),
    ("config.yaml", 0.3),
])

# Items in spotlight: full processing (1.0x)
gate.get_processing_multiplier("auth_handler.py")  # 1.0

# Items in penumbra: reduced (0.3x)
# Items in periphery: minimal (0.1x)
```

### Unified Attention System

The `AttentionSystem` facade wires all components together:

```python
from corteX.engine.attention import AttentionSystem

attention = AttentionSystem()

result = attention.process_turn(
    message="Let's switch to the payment system",
    context={"tools_used": [], "error_count": 0},
)
# {
#   "priority": "foreground",
#   "budget": {"max_tokens": 4096, "model_tier": "orchestrator", ...},
#   "compressed_context": {...},
#   "change_highlights": "...",
#   "spotlight": ["auth_handler.py", ...],
#   "delta_magnitude": 0.62,
# }
```

## When It Activates

- **Before every LLM call**: `classify()` determines the processing budget
- **Every turn**: `ChangeDetector` tracks state and detects changes
- **Before context packing**: `ContextDeltaCompressor` highlights changes
- **Continuously**: `AttentionalGate` manages the information spotlight

## API Reference

```python
from corteX.engine.attention import (
    AttentionSystem,
    AttentionalFilter,
    AttentionalPriority,
    ChangeDetector,
    ChangeEvent,
    ContextDeltaCompressor,
    AttentionalGate,
    ProcessingBudget,
)
```
