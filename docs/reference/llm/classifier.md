# Cognitive Classifier API Reference

## Module: `corteX.core.llm.classifier`

Pure-heuristic cognitive task classifier with zero LLM calls. Classifies every incoming task along two axes: cognitive type (WHAT kind of thinking) and complexity tier (HOW hard). Guaranteed under 1ms per classification on modern hardware.

Brain analogy: The thalamic reticular nucleus that classifies and routes incoming sensory signals to the appropriate cortical region before conscious processing begins.

## Enums

### `CognitiveType`

```python
class CognitiveType(str, Enum)
```

Ten categories of cognitive processing.

| Value | Description |
|-------|-------------|
| `REASONING` | Logical analysis, explanation, deduction |
| `PLANNING` | Strategy, architecture, decomposition |
| `CODING` | Code generation, debugging, implementation |
| `CREATIVE` | Writing, brainstorming, composition |
| `FACTUAL_RECALL` | Lookup, definitions, factual answers |
| `SUMMARIZATION` | Condensing, distilling, overview |
| `VALIDATION` | Review, verification, correctness |
| `DECISION` | Choice, recommendation, tradeoff analysis |
| `TOOL_USE` | Function calling, tool invocation |
| `CLASSIFICATION` | Categorization, labeling, sorting |

### `ComplexityTier`

```python
class ComplexityTier(str, Enum)
```

Five-tier complexity scale mapped to a 0.0-1.0 range.

| Value | Range | Description |
|-------|-------|-------------|
| `TRIVIAL` | 0.0 - 0.2 | Simple lookups, greetings |
| `SIMPLE` | 0.2 - 0.4 | Basic questions, short tasks |
| `MODERATE` | 0.4 - 0.6 | Multi-step tasks, moderate context |
| `COMPLEX` | 0.6 - 0.8 | Architecture, multi-tool tasks |
| `CRITICAL` | 0.8 - 1.0 | Enterprise-critical, security-sensitive |

#### Static Methods

##### `from_score`

```python
@staticmethod
def from_score(score: float) -> ComplexityTier
```

Map a 0-1 complexity score to the corresponding tier.

---

## Data Classes

### `CognitiveProfile`

**Type**: `@dataclass(frozen=True)`

Complete cognitive classification result for a task.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `cognitive_type` | `CognitiveType` | Primary cognitive type detected |
| `complexity` | `float` | Complexity score [0.0, 1.0] |
| `complexity_tier` | `ComplexityTier` | Tier derived from complexity score |
| `requires_thinking` | `bool` | Whether extended thinking is recommended |
| `requires_vision` | `bool` | Whether vision/multimodal input is present |
| `requires_tools` | `bool` | Whether tools are available |
| `estimated_tokens` | `int` | Estimated output tokens for budget planning |
| `latency_sensitivity` | `float` | 0.0 = background, 1.0 = interactive |
| `confidence` | `float` | Classification confidence [0.0, 1.0] |
| `signals` | `Dict[str, float]` | Raw signal values for observability/debugging |

---

## Classes

### `CognitiveClassifier`

Zero-LLM cognitive classifier using weighted heuristic signals: keyword frequency, message length, tool count, conversation depth, code block presence, and complexity amplifier/reducer words.

#### Methods

##### `classify`

```python
def classify(
    self,
    message: str,
    tools: Optional[List[Any]] = None,
    conversation_depth: int = 0,
    has_vision_content: bool = False,
) -> CognitiveProfile
```

Classify a task by cognitive type and complexity. Pure string operations, guaranteed under 1ms.

**Parameters**:

- `message` (str): The input message text to classify
- `tools` (`Optional[List[Any]]`): Available tools (presence biases toward TOOL_USE)
- `conversation_depth` (int): Number of turns in the conversation (higher = more complex)
- `has_vision_content` (bool): Whether multimodal/image content is present

**Returns**: `CognitiveProfile` with type, complexity, flags, and signals.

**Example**:

```python
from corteX.core.llm.classifier import CognitiveClassifier

classifier = CognitiveClassifier()

# Coding task
profile = classifier.classify("Implement a REST API with authentication")
print(profile.cognitive_type)  # CognitiveType.CODING
print(profile.complexity)      # 0.42
print(profile.requires_thinking)  # False

# Complex planning task
profile = classifier.classify(
    "Design a distributed microservices architecture for enterprise deployment",
    conversation_depth=5,
)
print(profile.cognitive_type)  # CognitiveType.PLANNING
print(profile.complexity)      # 0.71
print(profile.requires_thinking)  # True
```

---

## Classification Signals

The classifier uses six weighted signals to compute complexity:

| Signal | Weight | Source |
|--------|--------|--------|
| `length` | 0.20 | `min(msg_length / 2000, 1.0)` |
| `word_count` | 0.15 | `min(unique_words / 150, 1.0)` |
| `depth` | 0.15 | `min(conversation_depth / 20, 1.0)` |
| `tools` | 0.15 | `min(tool_count / 10, 1.0)` |
| `modifiers` | 0.20 | Balance of amplifier vs. reducer keywords |
| `type_base` | 0.15 | Inherent complexity of the cognitive type |

**Thinking is recommended** when complexity >= 0.6 or the type is REASONING or PLANNING.

---

## Keyword Lexicons

The classifier maintains frozen sets of keywords for each cognitive type. Each set contains 11-26 keywords ordered by specificity:

- `_CODING_KW`: 26 keywords (code, implement, function, class, debug, python, etc.)
- `_PLANNING_KW`: 17 keywords (plan, strategy, design, architect, roadmap, etc.)
- `_REASONING_KW`: 19 keywords (reason, analyze, why, explain, logic, etc.)
- `_CREATIVE_KW`: 18 keywords (write, story, brainstorm, compose, etc.)

Type scoring: `hits / lexicon_size`, with the highest-scoring type selected.

---

## See Also

- [LLM Router API](./router.md) - Uses classifier for routing decisions
- [Intelligent Model Routing Concept](../../concepts/model-routing.md)
