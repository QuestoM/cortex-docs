# LLM Base API Reference

## Module: `corteX.core.llm.base`

This module defines the abstract LLM provider interface, universal message/response types, error hierarchy, and retry logic. All provider implementations (OpenAI, Gemini, Anthropic, local) must implement the `BaseLLMProvider` abstract class.

## Enums

### `ProviderType`

```python
class ProviderType(str, Enum)
```

Supported LLM provider types.

| Value | Description |
|-------|-------------|
| `OPENAI` | OpenAI API (GPT-4o, o3, etc.) |
| `GEMINI` | Google Gemini API (Gemini 3, 2.5) |
| `ANTHROPIC` | Anthropic Claude API (Opus, Sonnet, Haiku) |
| `LOCAL` | Any OpenAI-compatible local endpoint (vLLM, Ollama, etc.) |

### `ModelCapability`

```python
class ModelCapability(str, Enum)
```

Capability flags for model metadata.

| Value | Description |
|-------|-------------|
| `TEXT` | Text generation |
| `VISION` | Image/multimodal input |
| `TOOL_CALLING` | Function/tool calling |
| `STRUCTURED_OUTPUT` | JSON schema output |
| `STREAMING` | Streaming response chunks |
| `CODE_EXECUTION` | Native code execution |
| `THINKING` | Extended thinking / chain-of-thought |

---

## Data Classes

### `ModelInfo`

**Type**: `@dataclass`

Metadata about a specific model, used by the router for scoring decisions.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `model_id` | `str` | -- | Unique model identifier (e.g., `"gpt-4o"`) |
| `provider` | `ProviderType` | -- | Which provider serves this model |
| `capabilities` | `List[ModelCapability]` | `[]` | Supported capabilities |
| `max_tokens` | `int` | `128000` | Context window size in tokens |
| `cost_per_1k_input` | `float` | `0.0` | Cost per 1K input tokens (USD) |
| `cost_per_1k_output` | `float` | `0.0` | Cost per 1K output tokens (USD) |
| `speed_tier` | `str` | `"medium"` | Speed classification: `"fast"`, `"medium"`, `"slow"` |
| `intelligence_tier` | `str` | `"medium"` | Intelligence level: `"low"`, `"medium"`, `"high"`, `"frontier"` |

---

### `LLMMessage`

**Type**: `@dataclass`

Universal message format used across all providers.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `role` | `str` | Message role: `"system"`, `"user"`, `"assistant"`, `"tool"` |
| `content` | `Union[str, List[Dict[str, Any]]]` | Text content or multimodal parts list |
| `name` | `Optional[str]` | Optional sender name |
| `tool_calls` | `Optional[List[Dict[str, Any]]]` | Tool/function calls from the assistant |
| `tool_call_id` | `Optional[str]` | ID linking a tool result to its call |

#### Example

```python
from corteX.core.llm.base import LLMMessage

# Simple text message
user_msg = LLMMessage(role="user", content="Explain quantum computing")

# System instruction
system_msg = LLMMessage(role="system", content="You are a physics professor")

# Tool result message
tool_msg = LLMMessage(
    role="tool",
    content='{"temperature": 72}',
    tool_call_id="call_abc123",
)
```

---

### `ToolDefinition`

**Type**: `@dataclass`

Universal tool/function definition passed to LLM providers.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `name` | `str` | Tool function name |
| `description` | `str` | Human-readable description for the LLM |
| `parameters` | `Dict[str, Any]` | JSON Schema defining the parameters |

---

### `LLMResponse`

**Type**: `@dataclass`

Universal response format returned by all providers.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `content` | `str` | -- | Generated text content |
| `model` | `str` | -- | Model ID that generated the response |
| `provider` | `ProviderType` | -- | Provider that served the request |
| `tool_calls` | `List[Dict[str, Any]]` | `[]` | Tool/function calls requested by the model |
| `thinking` | `Optional[str]` | `None` | Extended thinking content (if enabled) |
| `usage` | `Optional[Dict[str, int]]` | `None` | Token usage: `{"input_tokens": N, "output_tokens": M}` |
| `finish_reason` | `str` | `"stop"` | Why generation stopped: `"stop"`, `"length"`, `"tool_calls"` |
| `raw_response` | `Any` | `None` | Original provider SDK response object |

---

### `StreamChunk`

**Type**: `@dataclass`

A single chunk from a streaming response.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `content` | `str` | `""` | Text content delta |
| `tool_call_delta` | `Optional[Dict[str, Any]]` | `None` | Partial tool call data |
| `thinking_delta` | `Optional[str]` | `None` | Thinking content delta |
| `finish_reason` | `Optional[str]` | `None` | Set on final chunk |
| `model` | `Optional[str]` | `None` | Model ID |
| `chunk_type` | `str` | `"content"` | `"content"`, `"tool_execution"`, `"tool_result"` |

---

## Abstract Class

### `BaseLLMProvider`

```python
class BaseLLMProvider(ABC)
```

Abstract base for all LLM providers. Implementations handle their own auth, retries, and error mapping.

#### Class Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `provider_type` | `ProviderType` | Provider identifier |
| `available_models` | `List[ModelInfo]` | Models available from this provider |

#### Abstract Methods

##### `generate`

```python
async def generate(
    self,
    messages: List[LLMMessage],
    model: Optional[str] = None,
    temperature: float = 0.7,
    max_tokens: Optional[int] = None,
    tools: Optional[List[ToolDefinition]] = None,
    response_format: Optional[Dict[str, Any]] = None,
    thinking: bool = False,
    thinking_budget: Optional[int] = None,
    system_instruction: Optional[str] = None,
    top_p: Optional[float] = None,
    top_k: Optional[int] = None,
    frequency_penalty: Optional[float] = None,
    presence_penalty: Optional[float] = None,
) -> LLMResponse
```

Generate a completion. All providers must implement this.

##### `generate_stream`

```python
async def generate_stream(...) -> AsyncIterator[StreamChunk]
```

Generate a streaming completion. Returns an async iterator of `StreamChunk` objects.

##### `generate_structured`

```python
async def generate_structured(
    self,
    messages: List[LLMMessage],
    response_model: type[BaseModel],
    model: Optional[str] = None,
    system_instruction: Optional[str] = None,
) -> BaseModel
```

Generate a structured (Pydantic-validated) response.

##### `health_check`

```python
async def health_check(self) -> bool
```

Check if the provider is reachable and authenticated.

#### Concrete Methods

##### `get_model_info`

```python
def get_model_info(self, model_id: str) -> Optional[ModelInfo]
```

Look up model metadata by ID from `available_models`. Returns `None` if not found.

---

## Error Hierarchy

All LLM provider errors inherit from `LLMError`. The hierarchy determines retry behavior.

```
LLMError (base)
  |-- RateLimitError        (429 - retryable with backoff)
  |-- ServiceUnavailableError (503 - retryable with backoff)
  |-- AuthenticationError   (401/403 - fail fast, no retry)
  |-- ContextLengthExceededError (input too long - fail fast)
  |-- InvalidRequestError   (400 - fail fast, no retry)
```

### `RateLimitError`

```python
RateLimitError(message: str = "Rate limit exceeded", retry_after: Optional[float] = None)
```

Has an optional `retry_after` attribute (seconds) extracted from provider response headers.

---

## Functions

### `retry_with_backoff`

```python
async def retry_with_backoff(
    fn: Callable,
    max_retries: int = 3,
    base_delay: float = 1.0,
) -> Any
```

Retry an async callable with exponential backoff for transient errors.

**Retries on**: `RateLimitError`, `ServiceUnavailableError`, `TimeoutError`

**Fails fast on**: `AuthenticationError`, `ContextLengthExceededError`, `InvalidRequestError`

**Backoff formula**: `base_delay * (2 ^ attempt) + random(0, 0.5)` seconds. Respects `retry_after` from `RateLimitError`.

**Example**:

```python
from corteX.core.llm.base import retry_with_backoff

async def call_api():
    return await some_provider.generate(messages)

response = await retry_with_backoff(call_api, max_retries=3, base_delay=1.0)
```

---

## See Also

- [LLM Router API](./router.md)
- [OpenAI Provider](./openai-client.md)
- [Gemini Provider](./gemini-client.md)
- [Anthropic Provider](./anthropic-client.md)
