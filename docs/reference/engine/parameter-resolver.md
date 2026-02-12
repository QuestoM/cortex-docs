# Brain Parameter Resolver API Reference

## Module: `corteX.engine.parameter_resolver`

Translates cognitive state from brain components into concrete LLM API parameters (temperature, top_p, max_tokens, frequency_penalty, presence_penalty, thinking_budget).

This module bridges the gap between corteX's brain-inspired components and the parameters that LLM providers expose. The resolver is pure computation: no side effects, no I/O, no state mutation.

## Classes

### `LLMParameterBundle`

**Type**: `@dataclass`

All LLM parameters that can be brain-controlled. `None` values mean "use provider default."

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `temperature` | `Optional[float]` | `None` | Exploration vs. exploitation tradeoff [0.0, 2.0] |
| `top_p` | `Optional[float]` | `None` | Nucleus sampling threshold [0.0, 1.0] |
| `top_k` | `Optional[int]` | `None` | Top-k sampling (Gemini only) |
| `max_tokens` | `Optional[int]` | `None` | Maximum tokens in response |
| `frequency_penalty` | `Optional[float]` | `None` | Penalize token repetition (OpenAI only) [0.0, 2.0] |
| `presence_penalty` | `Optional[float]` | `None` | Encourage new vocabulary (OpenAI only) [0.0, 2.0] |
| `thinking_budget` | `Optional[int]` | `None` | Reasoning token budget (for o1-style models) |
| `seed` | `Optional[int]` | `None` | Random seed for deterministic generation |
| `stop_sequences` | `Optional[List[str]]` | `None` | Stop sequences |

#### Methods

##### `to_dict`

```python
def to_dict(self) -> Dict[str, Any]
```

Serialize to a plain dict, omitting `None` values.

**Returns**: `Dict[str, Any]` - Only non-None parameters included

**Example**:

```python
bundle = LLMParameterBundle(temperature=0.7, max_tokens=2000)
print(bundle.to_dict())
# → {"temperature": 0.7, "max_tokens": 2000}
```

##### `to_provider_kwargs`

```python
def to_provider_kwargs(self, provider: ProviderType) -> Dict[str, Any]
```

Filter parameters to only those supported by the given provider.

**Parameters**:

- `provider` (`ProviderType`): Target LLM provider

**Returns**: `Dict[str, Any]` - Provider-specific parameter dict suitable for `**kwargs` unpacking

**Supported Parameters by Provider**:

| Provider | Supported Parameters |
|----------|---------------------|
| **OpenAI** | temperature, top_p, max_tokens, frequency_penalty, presence_penalty, seed, stop_sequences, thinking_budget |
| **Gemini** | temperature, top_p, top_k, max_tokens, stop_sequences |
| **Local** | Same as OpenAI |

**Example**:

```python
from corteX.core.llm.base import ProviderType

bundle = LLMParameterBundle(
    temperature=0.7,
    top_p=0.9,
    top_k=40,  # Gemini-specific
    frequency_penalty=0.5,  # OpenAI-specific
)

# OpenAI: excludes top_k
openai_kwargs = bundle.to_provider_kwargs(ProviderType.OPENAI)
# → {"temperature": 0.7, "top_p": 0.9, "frequency_penalty": 0.5}

# Gemini: excludes frequency_penalty
gemini_kwargs = bundle.to_provider_kwargs(ProviderType.GEMINI)
# → {"temperature": 0.7, "top_p": 0.9, "top_k": 40}
```

---

### `BrainParameterResolver`

Translates cognitive state from brain components into concrete LLM API parameters.

#### Constructor

```python
BrainParameterResolver()
```

No parameters. The resolver is stateless except for observability stats from the last `resolve()` call.

#### Methods

##### `resolve`

```python
def resolve(
    self,
    *,
    task_type: str = "conversation",
    model: str = "",
    provider: ProviderType = ProviderType.OPENAI,
    process_type: Optional[str] = None,
    surprise: float = 0.0,
    calibration_health: str = "healthy",
    confidence: float = 0.7,
    attention_priority: str = "foreground",
    creativity: float = 0.0,
    verbosity: float = 0.0,
    resource_token_budget: Optional[float] = None,
    column_weight_overrides: Optional[Dict[str, float]] = None,
    modulator_clamps: Optional[Dict[str, float]] = None,
) -> LLMParameterBundle
```

Resolve all brain signals into a concrete `LLMParameterBundle`.

**Parameters**:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `task_type` | `str` | `"conversation"` | Current task classification (coding, planning, debugging, etc.) |
| `model` | `str` | `""` | Model identifier (e.g., "gemini-3-pro-preview") |
| `provider` | `ProviderType` | `OPENAI` | LLM provider type |
| `process_type` | `Optional[str]` | `None` | "system1" or "system2" from DualProcessRouter |
| `surprise` | `float` | `0.0` | Average surprise from PredictionEngine [0.0, 1.0] |
| `calibration_health` | `str` | `"healthy"` | "healthy", "warning", or "critical" |
| `confidence` | `float` | `0.7` | Calibration confidence level [0.0, 1.0] |
| `attention_priority` | `str` | `"foreground"` | "critical", "foreground", "background", "subconscious", "suppressed" |
| `creativity` | `float` | `0.0` | BehavioralWeight for creativity [-1.0, 1.0] |
| `verbosity` | `float` | `0.0` | BehavioralWeight for verbosity [-1.0, 1.0] |
| `resource_token_budget` | `Optional[float]` | `None` | ResourceHomunculus token_budget multiplier |
| `column_weight_overrides` | `Optional[Dict[str, float]]` | `None` | Active column's weight_overrides dict |
| `modulator_clamps` | `Optional[Dict[str, float]]` | `None` | Dict of param_name → clamped_value from TargetedModulator |

**Returns**: `LLMParameterBundle` - All resolved parameters

**Example**:

```python
from corteX.engine.parameter_resolver import BrainParameterResolver
from corteX.core.llm.base import ProviderType

resolver = BrainParameterResolver()

# System 1: Fast, confident, routine task
bundle = resolver.resolve(
    task_type="coding",
    provider=ProviderType.OPENAI,
    process_type="system1",
    surprise=0.05,
    confidence=0.85,
    attention_priority="foreground",
    creativity=0.0,
    verbosity=0.2,
)

print(bundle.to_dict())
# → {
#     "temperature": 0.2,
#     "top_p": 0.85,
#     "max_tokens": 5120,
#     "frequency_penalty": 0.3,
#     "presence_penalty": 0.04,
#     "seed": 42
# }
```

##### `get_stats`

```python
def get_stats(self) -> Dict[str, Any]
```

Return the stats from the last `resolve()` call. Shows which brain signals contributed to each parameter decision.

**Returns**: `Dict[str, Any]` - Stats with "signals" and "decisions" keys

**Example**:

```python
resolver = BrainParameterResolver()
bundle = resolver.resolve(
    task_type="coding",
    process_type="system1",
    surprise=0.15,
    confidence=0.7,
    creativity=0.2,
)

stats = resolver.get_stats()
print(stats)
```

Output:

```python
{
    "signals": {
        "dual_process_base": 0.2,
        "surprise_boost": 0.045,
        "confidence_boost": 0.06,
        "attention_adjustment": 0.0,
        "creativity_delta": 0.03,
        "combined_raw": 0.335,
        "task_ceiling": 0.5,
        "temperature_final": 0.335,
    },
    "decisions": {
        "temperature": "brain_state_computation",
        "top_p": "system1_default",
        "max_tokens": "attention_resource_verbosity",
        "frequency_penalty": "creativity_mapped",
        "presence_penalty": "surprise_mapped",
        "thinking_budget": "system1_no_thinking",
        "seed": "system1_low_surprise_deterministic",
    }
}
```

---

## Constants and Thresholds

The resolver uses class-level constants for all thresholds (no magic numbers):

### Temperature Base Values

```python
SYSTEM1_BASE_TEMP: float = 0.2  # Fast path
SYSTEM2_BASE_TEMP: float = 0.6  # Slow path
```

### Temperature Modulation Ranges

```python
SURPRISE_MAX_BOOST: float = 0.3          # High surprise → +0.3 temp
CONFIDENCE_MAX_BOOST: float = 0.2        # Low confidence → +0.2 temp
CREATIVITY_MAX_DELTA: float = 0.15       # Creativity → ±0.15 temp
CRITICAL_TEMP_ADJUSTMENT: float = -0.1   # Critical attention → -0.1 temp
SUBCONSCIOUS_TEMP_ADJUSTMENT: float = -0.15  # Subconscious → -0.15 temp
```

### Task Temperature Ceilings

```python
TASK_CEILING: Dict[str, float] = {
    "coding": 0.5,
    "validation": 0.3,
    "tool_use": 0.4,
    "planning": 0.9,
    "conversation": 1.0,
    "summarization": 0.6,
    "reasoning": 0.7,
    "debugging": 0.5,
    "testing": 0.4,
    "research": 0.8,
}
```

### Top-p Values

```python
SYSTEM1_TOP_P: float = 0.85  # Tight distribution
SYSTEM2_TOP_P: float = 0.95  # Wide distribution
```

### Attention Token Budgets

```python
ATTENTION_TOKEN_BUDGETS: Dict[str, int] = {
    "critical": 8192,
    "foreground": 4096,
    "background": 2048,
    "subconscious": 1024,
    "suppressed": 256,
}
```

### Resource Token Multiplier

```python
RESOURCE_TOKEN_MULTIPLIER: int = 4096
# resource_token_budget=1.2 → 1.2 * 4096 = 4915 tokens
```

### Frequency Penalty Mapping

```python
FREQ_PENALTY_BASE: float = 0.3
FREQ_PENALTY_SCALE: float = 0.6
# penalty = max(0, 0.3 + creativity * 0.6)
# creativity=-1 → 0.0, creativity=0 → 0.3, creativity=1 → 0.9
```

### Presence Penalty Mapping

```python
PRESENCE_PENALTY_SCALE: float = 0.8
# penalty = surprise * 0.8
# surprise=0 → 0.0, surprise=1 → 0.8
```

### Thinking Budget Thresholds

```python
THINKING_BUDGET_CRITICAL: int = 8192   # Critical calibration health
THINKING_BUDGET_WARNING: int = 4096    # Warning health
THINKING_BUDGET_HEALTHY: int = 2048    # Healthy
```

### Determinism Threshold

```python
SURPRISE_DETERMINISM_THRESHOLD: float = 0.1
DETERMINISTIC_SEED: int = 42
# If process_type="system1" AND surprise < 0.1 → seed=42
```

### Gemini 3 Forced Temperature

```python
GEMINI3_FORCED_TEMP: float = 1.0
# Gemini 3 models ignore temperature parameter, always use 1.0
```

---

## Resolution Priority Hierarchy

When multiple systems attempt to set the same parameter, the resolver follows this strict priority order:

1. **Modulator CLAMP** (highest) - `TargetedModulator` hard overrides
2. **Column override** - `FunctionalColumn.weight_overrides`
3. **Gemini 3 forced temperature** - Provider constraint (temperature only)
4. **Brain state computation** - Multi-signal fusion (default)
5. **Task ceiling** - Task-specific upper bounds (temperature only)

**Example**:

```python
# Brain computes temperature=0.7 from signals
# Task ceiling for "coding" is 0.5 → temperature=0.5

# But if column override exists:
column_weight_overrides = {"temperature": 0.3}
# → temperature=0.3 (column beats brain state and task ceiling)

# But if modulator CLAMP exists:
modulator_clamps = {"temperature": 0.9}
# → temperature=0.9 (modulator beats everything)
```

---

## Temperature Resolution Algorithm

The temperature resolution is the most complex, combining 7 signals:

```python
def _resolve_temperature(...) -> float:
    # Priority 1: Modulator CLAMP
    if "temperature" in clamps:
        return clamp(clamps["temperature"], 0.0, 2.0)

    # Priority 2: Column override
    if "temperature" in column_overrides:
        return clamp(column_overrides["temperature"], 0.0, 2.0)

    # Priority 3: Gemini 3 forced
    if is_gemini_3(model):
        return 1.0

    # Priority 4: Brain state computation
    # Step 4a: Base from DualProcess
    base = SYSTEM1_BASE_TEMP if process_type == "system1" else SYSTEM2_BASE_TEMP

    # Step 4b: Surprise modulation
    surprise_boost = surprise * SURPRISE_MAX_BOOST

    # Step 4c: Confidence modulation
    confidence_boost = (1.0 - confidence) * CONFIDENCE_MAX_BOOST

    # Step 4d: Attention modulation
    if attention_priority == "critical":
        attention_adj = CRITICAL_TEMP_ADJUSTMENT
    elif attention_priority == "subconscious":
        attention_adj = SUBCONSCIOUS_TEMP_ADJUSTMENT
    else:
        attention_adj = 0.0

    # Step 4e: Creativity
    creativity_delta = creativity * CREATIVITY_MAX_DELTA

    # Combine
    temp = base + surprise_boost + confidence_boost + attention_adj + creativity_delta

    # Step 4f: Task ceiling
    temp = min(temp, TASK_CEILING.get(task_type, 1.0))

    # Final clamp
    return clamp(temp, 0.0, 2.0)
```

---

## Integration in Session

The `BrainParameterResolver` is automatically instantiated in `Session.__init__()`:

```python
class Session:
    def __init__(self, agent, user_id, session_id=None):
        # ...
        self._param_resolver = BrainParameterResolver()
        # ...

    async def run(self, message: str) -> Response:
        # ...
        # Resolve brain parameters
        param_bundle = self._resolve_brain_parameters(
            task_type=task_type,
            process_type=process_type,
            attention_priority=attention_priority,
            active_column=active_column,
            resource_alloc=resource_alloc,
        )

        # Generate with resolved parameters
        response = await router.generate(
            messages=messages,
            tools=tool_defs,
            role=role,
            temperature=param_bundle.temperature,
            top_p=param_bundle.top_p,
            top_k=param_bundle.top_k,
            max_tokens=param_bundle.max_tokens,
            frequency_penalty=param_bundle.frequency_penalty,
            presence_penalty=param_bundle.presence_penalty,
            thinking_budget=param_bundle.thinking_budget,
        )
```

---

## Helper Functions

### `_clamp`

```python
def _clamp(value: float, min_val: float, max_val: float) -> float
```

Clamp a value to a range. Used internally by all resolution methods.

**Example**:

```python
_clamp(0.95, 0.0, 1.0)  # → 0.95
_clamp(1.5, 0.0, 1.0)   # → 1.0
_clamp(-0.2, 0.0, 1.0)  # → 0.0
```

---

## Performance Notes

- The resolver is stateless and performs no I/O - all methods are synchronous and fast
- Typical resolution time: <0.1ms
- All constants are class-level (no dictionary lookups at runtime for common paths)
- The resolver creates a new stats dict for each `resolve()` call but reuses the same instance variable for observability

---

## Error Handling

The resolver handles invalid inputs gracefully:

```python
# Invalid surprise range → clamped
bundle = resolver.resolve(surprise=1.5)  # Clamped to 1.0

# Invalid confidence → clamped
bundle = resolver.resolve(confidence=-0.2)  # Clamped to 0.0

# Invalid task_type → defaults to 1.0 ceiling
bundle = resolver.resolve(task_type="unknown")  # Uses 1.0 ceiling

# Invalid process_type → neutral base
bundle = resolver.resolve(process_type="invalid")  # Uses 0.4 base temp

# None values → ignored
bundle = resolver.resolve(
    resource_token_budget=None,  # Skips resource budget
    column_weight_overrides=None,  # Skips column overrides
)
```

No exceptions are raised - the resolver always returns a valid `LLMParameterBundle`.

---

## See Also

- [Brain-LLM Bridge Concept Guide](../../concepts/brain/brain-llm-bridge.md) - Conceptual overview
- [Brain Parameter Configuration](../../guides/config/brain-parameters.md) - How-to guide for customization
- [Brain State Injector API](./brain-state-injector.md) - Prompt context injection
