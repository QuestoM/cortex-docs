# OpenAI Provider API Reference

## Module: `corteX.core.llm.openai_client`

OpenAI-compatible LLM provider. Works with the OpenAI API, Azure OpenAI, and any OpenAI-compatible endpoint (vLLM, Ollama, LM Studio, etc.).

## Classes

### `OpenAIProvider`

**Extends**: `BaseLLMProvider`

#### Constructor

```python
OpenAIProvider(
    api_key: str,
    base_url: Optional[str] = None,
    default_model: str = "gpt-4o",
    organization: Optional[str] = None,
)
```

**Parameters**:

- `api_key` (str): OpenAI API key
- `base_url` (`Optional[str]`): Custom API endpoint URL. Set this for Azure OpenAI, vLLM, Ollama, or any OpenAI-compatible server
- `default_model` (str, default=`"gpt-4o"`): Default model when none is specified in generate calls
- `organization` (`Optional[str]`): OpenAI organization ID

#### Registered Models

| Model ID | Context | Speed | Intelligence | Cost (input/output per 1K) |
|----------|---------|-------|--------------|----------------------------|
| `gpt-4o` | 128K | medium | high | $0.0025 / $0.01 |
| `gpt-4o-mini` | 128K | fast | medium | $0.00015 / $0.0006 |
| `o3` | 200K | slow | frontier | $0.01 / $0.04 |

#### Methods

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

Generate a completion using the OpenAI API. Includes automatic retry with backoff for transient errors (429, 503, timeouts). Errors are classified into the typed `LLMError` hierarchy.

**Parameters**:

- `messages` (`List[LLMMessage]`): Conversation messages. System instruction is prepended automatically
- `model` (`Optional[str]`): Model ID override. Falls back to `default_model`
- `temperature` (float): Sampling temperature [0.0, 2.0]
- `max_tokens` (`Optional[int]`): Maximum output tokens
- `tools` (`Optional[List[ToolDefinition]]`): Function definitions for tool calling
- `response_format` (`Optional[Dict]`): JSON schema for structured output
- `system_instruction` (`Optional[str]`): System prompt (prepended as a system message)
- `top_p` (`Optional[float]`): Nucleus sampling threshold
- `frequency_penalty` (`Optional[float]`): Penalize repeated tokens [-2.0, 2.0]
- `presence_penalty` (`Optional[float]`): Penalize token presence [-2.0, 2.0]
- `top_k`: Silently ignored (not supported by OpenAI API)

**Returns**: `LLMResponse` with content, tool calls, token usage, and finish reason.

**Example**:

```python
from corteX.core.llm.openai_client import OpenAIProvider
from corteX.core.llm.base import LLMMessage

provider = OpenAIProvider(api_key="sk-...", default_model="gpt-4o")

response = await provider.generate(
    messages=[LLMMessage(role="user", content="Explain recursion")],
    temperature=0.5,
    max_tokens=1000,
)
print(response.content)
print(response.usage)  # {"input_tokens": 8, "output_tokens": 250}
```

##### `generate_stream`

```python
async def generate_stream(
    self,
    messages: List[LLMMessage],
    model: Optional[str] = None,
    temperature: float = 0.7,
    max_tokens: Optional[int] = None,
    tools: Optional[List[ToolDefinition]] = None,
    thinking: bool = False,
    system_instruction: Optional[str] = None,
    top_p: Optional[float] = None,
    top_k: Optional[int] = None,
) -> AsyncIterator[StreamChunk]
```

Stream response chunks from the OpenAI API. Emits `StreamChunk` objects for text content deltas, tool call deltas, and finish reasons.

**Example**:

```python
async for chunk in provider.generate_stream(
    messages=[LLMMessage(role="user", content="Write a haiku")],
):
    if chunk.content:
        print(chunk.content, end="", flush=True)
    if chunk.tool_call_delta:
        print(f"Tool: {chunk.tool_call_delta}")
```

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

Generate a Pydantic-validated structured response using OpenAI's `json_schema` response format.

**Example**:

```python
from pydantic import BaseModel

class Summary(BaseModel):
    title: str
    points: list[str]

result = await provider.generate_structured(
    messages=[LLMMessage(role="user", content="Summarize Python benefits")],
    response_model=Summary,
)
print(result.title)   # "Benefits of Python"
print(result.points)  # ["Easy to learn", ...]
```

##### `health_check`

```python
async def health_check(self) -> bool
```

Check if the OpenAI API is reachable by listing available models. Returns `True` on success, `False` on any error.

---

## Error Classification

The module maps OpenAI SDK exceptions to the corteX error hierarchy via `_classify_openai_error()`:

| OpenAI Exception | corteX Error | Retry? |
|-----------------|-------------|--------|
| `RateLimitError` / 429 | `RateLimitError` | Yes (with backoff) |
| `AuthenticationError` / 401/403 | `AuthenticationError` | No |
| `InternalServerError` / 503 | `ServiceUnavailableError` | Yes (with backoff) |
| `BadRequestError` / 400 (context) | `ContextLengthExceededError` | No |
| `BadRequestError` / 400 (other) | `InvalidRequestError` | No |
| `APITimeoutError` | `TimeoutError` | Yes (with backoff) |

---

## Local Model Usage

To use with OpenAI-compatible local endpoints:

```python
provider = OpenAIProvider(
    api_key="not-needed",
    base_url="http://localhost:8080/v1",  # vLLM, Ollama, etc.
    default_model="local-model-name",
)
```

---

## See Also

- [BaseLLMProvider API](./base.md)
- [OpenAI Provider Guide](../../guides/providers/openai.md)
- [Local Models Guide](../../guides/providers/local-models.md)
