# Brain State Injector API Reference

## Module: `corteX.engine.brain_state_injector`

This module compiles brain state (behavioral weights, column mode, goal progress, attention changes, calibration warnings, prediction context, active concepts) into additions to the LLM system prompt.

Without this module, the brain operates in a parallel universe from the LLM. With it, every brain computation becomes actionable LLM context.

## Classes

### `BrainSnapshot`

**Type**: `@dataclass`

Stateless input containing all brain state needed to compile LLM context.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `behavioral_weights` | `Dict[str, float]` | Behavioral weight values (verbosity, formality, creativity, etc.) |
| `active_column` | `Optional[Dict[str, Any]]` | Active functional column info (name, specialization, overrides, competence) |
| `goal_state` | `Optional[Dict[str, Any]]` | Goal tracking state (goal, progress, drift, loop_detected, stall_turns) |
| `change_highlights` | `Optional[str]` | Attention change highlights text from `ChangeDetector` |
| `calibration_alerts` | `Optional[List[Dict[str, Any]]]` | Recent calibration alerts from `MetaCognitionMonitor` |
| `calibration_ece` | `Optional[Dict[str, float]]` | ECE scores by domain (tool_success, model_quality, etc.) |
| `prediction_surprise` | `Optional[float]` | Average surprise from `PredictionEngine` [0.0, 1.0] |
| `prediction_context` | `Optional[Dict[str, Any]]` | Predicted outcome and confidence |
| `active_concepts` | `Optional[List[Tuple[str, float]]]` | Top-activated concepts from `ConceptGraph` (name, activation_level) |
| `proactive_prediction` | `Optional[Dict[str, Any]]` | Proactive next-turn prediction (predicted_task_type, confidence) |
| `user_insights` | `Optional[Dict[str, Any]]` | User preference insights (reserved for future use) |

#### Example

```python
from corteX.engine.brain_state_injector import BrainSnapshot

snapshot = BrainSnapshot(
    behavioral_weights={
        "verbosity": 0.5,
        "formality": -0.3,
        "creativity": 0.7,
    },
    active_column={
        "name": "coding",
        "specialization": "python_implementation",
        "weight_overrides": {"code_density": 0.9},
        "competence": 0.82,
    },
    goal_state={
        "goal": "Build a REST API",
        "progress": 0.45,
        "drift": 0.15,
        "loop_detected": False,
        "stall_turns": 0,
    },
    calibration_ece={"tool_success": 0.18},
    prediction_surprise=0.35,
)
```

---

### `BrainStateInjector`

Compiles brain state into a structured context block for the LLM prompt.

#### Constructor

```python
BrainStateInjector(max_tokens: int = 500)
```

**Parameters**:

- `max_tokens` (int, default=500): Maximum token budget for compiled brain context. Context exceeding this budget will be gracefully truncated.

#### Methods

##### `compile_brain_context`

```python
def compile_brain_context(self, snapshot: BrainSnapshot) -> str
```

Compile brain state into structured text. Returns empty string if nothing meaningful to inject.

**Parameters**:

- `snapshot` (`BrainSnapshot`): Brain state snapshot to compile

**Returns**: `str` - Structured markdown text for injection into system prompt (e.g., `"## Brain Context\n### Style\n- Be concise...\n### Goal: Build API\nProgress: 45%..."`), or empty string if no significant state.

**Example**:

```python
from corteX.engine.brain_state_injector import BrainStateInjector, BrainSnapshot

injector = BrainStateInjector(max_tokens=500)

snapshot = BrainSnapshot(
    behavioral_weights={"verbosity": 0.6, "creativity": 0.4},
)

context = injector.compile_brain_context(snapshot)
print(context)
```

Output:

```
## Brain Context

### Style
- Provide detailed, thorough responses.
- Be creative and explore novel approaches.
```

##### Internal Methods

The following methods are used internally by `compile_brain_context()` and are documented for reference:

- `_compile_behavioral_weights(weights: Dict[str, float]) -> str`
  - Translates significant behavioral weights into natural language instructions
  - Only weights with `abs(value) > 0.2` are included
  - Uses `_WEIGHT_LANGUAGE_MAP` for translations (e.g., verbosity=0.5 → "Provide detailed, thorough responses.")

- `_compile_column_mode(column: Optional[Dict[str, Any]]) -> str`
  - Describes the active functional column specialization
  - Includes weight overrides and competence level

- `_compile_goal_state(goal_state: Optional[Dict[str, Any]]) -> str`
  - Describes goal progress, drift, loops, and stalls
  - Emits warnings for drift > 0.3 or loops detected

- `_compile_attention_changes(highlights: Optional[str]) -> str`
  - Includes attention change highlights if present and meaningful

- `_compile_calibration_warnings(alerts, ece_scores) -> str`
  - Compiles calibration ECE warnings (ECE > 0.15) and metacognition alerts

- `_compile_prediction_context(avg_surprise, prediction) -> str`
  - Compiles prediction surprise (if > 0.3) and predicted outcomes

- `_compile_active_concepts(concepts: Optional[List[Tuple[str, float]]]) -> str`
  - Lists top 8 active concepts from ConceptGraph

- `_compile_proactive_prediction(prediction: Optional[Dict[str, Any]]) -> str`
  - Includes proactive next-turn prediction if confidence > 0.4

- `_truncate_to_budget(text: str) -> str`
  - Truncates text to stay within token budget (max_tokens * 4 chars/token)
  - Truncates at section boundaries when possible

---

## Constants

### `_WEIGHT_LANGUAGE_MAP`

```python
_WEIGHT_LANGUAGE_MAP: Dict[str, Tuple[float, str, float, str]]
```

Maps weight keys to natural language instructions.

**Structure**: `{weight_key: (high_thresh, high_text, low_thresh, low_text)}`

**Example entries**:

```python
{
    "verbosity": (0.3, "Provide detailed, thorough responses.",
                  -0.3, "Be concise and direct."),
    "formality": (0.3, "Use formal, professional language.",
                  -0.3, "Use a casual, conversational tone."),
    "creativity": (0.3, "Be creative and explore novel approaches.",
                   -0.3, "Stick to conventional, well-established approaches."),
    "autonomy": (0.7, "Act independently; make decisions without asking.",
                 -0.3, "Ask for confirmation before taking actions."),
    # ... 10 total mappings
}
```

### `_WEIGHT_SIGNIFICANCE_THRESHOLD`

```python
_WEIGHT_SIGNIFICANCE_THRESHOLD: float = 0.2
```

Minimum absolute weight value to include in compiled context. Weights with `abs(value) < 0.2` are considered insignificant and omitted.

### `_CHARS_PER_TOKEN`

```python
_CHARS_PER_TOKEN: int = 4
```

Approximation of characters per token for budget estimation. Used to convert `max_tokens` budget into character count for truncation.

---

## Integration in Session

The `BrainStateInjector` is automatically instantiated and used in `Session.__init__()`:

```python
class Session:
    def __init__(self, agent, user_id, session_id=None):
        # ...
        self._brain_injector = BrainStateInjector(max_tokens=500)
        # ...

    async def run(self, message: str) -> Response:
        # ...
        # Collect brain snapshot
        snapshot = self._collect_brain_snapshot(
            attention_result, active_column, active_concepts, proactive_pred
        )

        # Compile brain context
        brain_context = self._brain_injector.compile_brain_context(snapshot)

        # Augment system prompt
        system_with_brain = f"{base_prompt}\n\n{brain_context}" if brain_context else base_prompt

        # Generate with augmented prompt
        response = await router.generate(
            messages=messages,
            system_instruction=system_with_brain,
            # ...
        )
```

---

## Customization

### Adjusting Token Budget

```python
# Default: 500 tokens (≈2000 characters)
injector = BrainStateInjector(max_tokens=500)

# For terse contexts (mobile, low-token models)
injector = BrainStateInjector(max_tokens=200)

# For extensive contexts (high-end models, detailed instructions)
injector = BrainStateInjector(max_tokens=1000)
```

### Adding Custom Weight Mappings

To add custom behavioral weight translations, modify `_WEIGHT_LANGUAGE_MAP` at module initialization:

```python
from corteX.engine import brain_state_injector

# Add custom mapping
brain_state_injector._WEIGHT_LANGUAGE_MAP["risk_tolerance"] = (
    0.5, "You can suggest experimental or cutting-edge solutions.",
    -0.5, "Prefer safe, well-tested solutions only."
)
```

---

## Performance Notes

- The injector is stateless and performs no I/O - all methods are synchronous and fast
- Typical compilation time: <1ms for normal brain state
- Token budget truncation is designed to preserve complete sections rather than cutting mid-sentence
- Weights starting with `_` (internal state like `_lr_multiplier`) are automatically excluded from compilation

---

## Error Handling

The injector handles missing or malformed brain state gracefully:

```python
# Empty snapshot → empty string
snapshot = BrainSnapshot(behavioral_weights={})
context = injector.compile_brain_context(snapshot)
# → ""

# Partial snapshot → only non-empty sections
snapshot = BrainSnapshot(
    behavioral_weights={"verbosity": 0.1},  # Below significance threshold
    goal_state=None,
)
context = injector.compile_brain_context(snapshot)
# → "" (no significant weights, no goal)

# Malformed column info → gracefully skipped
snapshot = BrainSnapshot(
    behavioral_weights={},
    active_column={"invalid": "structure"},  # Missing 'name'
)
context = injector.compile_brain_context(snapshot)
# → "" (column section skipped)
```

No exceptions are raised - the injector always returns a valid string (possibly empty).

---

## See Also

- [Brain-LLM Bridge Concept Guide](../../concepts/brain/brain-llm-bridge.md) - Conceptual overview
- [Brain Parameter Configuration](../../guides/config/brain-parameters.md) - How-to guide for customization
- [Brain Parameter Resolver API](./parameter-resolver.md) - API parameter mapping
