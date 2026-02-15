# Gemini Provider API Reference

## Module: `corteX.core.llm.gemini_adapter`

Gemini provider adapter using the `google-genai` SDK. Supports Gemini 3 (latest, Feb 2026) and Gemini 2.5 (stable fallback) model families with tool calling, extended thinking, streaming, and code execution.

## Classes

### `GeminiAdapter`

**Extends**: `BaseLLMProvider`

#### Constructor

```python
GeminiAdapter(
    api_key: Optional[str] = None,
    default_model: str = "gemini-3-pro-preview",
)
```

**Parameters**:

- `api_key` (`Optional[str]`): Gemini API key. Falls back to `GEMINI_API_KEY` environment variable if not provided (with a warning logged)
- `default_model` (str, default=`"gemini-3-pro-preview"`): Default model for generation calls

#### Registered Models

| Model ID | Context | Speed | Intelligence | Cost (input/output per 1K) |
|----------|---------|-------|--------------|----------------------------|
| `gemini-3-pro-preview` | 1M | medium | frontier | $0.002 / $0.012 |
| `gemini-3-flash-preview` | 1M | fast | high | $0.0005 / $0.003 |
| `gemini-2.5-pro` | 1M | medium | frontier | $0.00125 / $0.010 |
| `gemini-2.5-flash` | 1M | fast | high | $0.0003 / $0.0025 |

All models support: TEXT, VISION, TOOL_CALLING, THINKING, STREAMING, CODE_EXECUTION.

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

Generate a completion using the Gemini API. Includes automatic retry with backoff for transient errors.

**Gemini 3 Temperature Override**: For Gemini 3 models, any temperature other than `1.0` is automatically overridden to `1.0` per Google's recommendation, with a warning logged.

**System Message Handling**: System messages in the message list are extracted and combined with the explicit `system_instruction` parameter into the Gemini `system_instruction` config.

**Tool Call IDs**: Gemini does not provide tool call IDs. The adapter synthesizes unique IDs (`call_<uuid8>`) for each function call to prevent downstream ID collisions.

**Parameters**:

- `messages` (`List[LLMMessage]`): Conversation messages. System messages are extracted automatically
- `model` (`Optional[str]`): Model ID override
- `temperature` (float): Sampling temperature (overridden to 1.0 for Gemini 3)
- `thinking` (bool): Enable thinking mode (uses `ThinkingConfig`)
- `thinking_budget` (`Optional[int]`): Token budget for thinking (default 4096)
- `top_p` (`Optional[float]`): Nucleus sampling threshold
- `top_k` (`Optional[int]`): Top-k sampling (native Gemini support)
- `frequency_penalty` (`Optional[float]`): Supported since genai v1
- `presence_penalty` (`Optional[float]`): Supported since genai v1

**Returns**: `LLMResponse` with content, thinking text, tool calls, and usage metadata.

**Example**:

```python
from corteX.core.llm.gemini_adapter import GeminiAdapter
from corteX.core.llm.base import LLMMessage

provider = GeminiAdapter(api_key="AIza...")

response = await provider.generate(
    messages=[LLMMessage(role="user", content="Design a microservices architecture")],
    thinking=True,
    thinking_budget=8192,
)
print(response.content)
print(response.thinking)  # Extended thinking output
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

Stream response chunks. The Gemini SDK returns a synchronous iterator, so the adapter wraps each `__next__()` call with `asyncio.to_thread()` to avoid blocking the event loop.

Gemini streams function calls as complete parts (not incrementally like OpenAI), so each `StreamChunk` with `tool_call_delta` contains the full function name and arguments.

**Example**:

```python
async for chunk in provider.generate_stream(
    messages=[LLMMessage(role="user", content="Explain gravity")],
):
    print(chunk.content, end="", flush=True)
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

Generate a structured response. Appends the JSON schema to the last message and parses the response, handling code-fenced JSON blocks.

##### `health_check`

```python
async def health_check(self) -> bool
```

Verify connectivity by generating a minimal response with `gemini-3-flash-preview`.

---

## Error Classification

The module maps Gemini SDK exceptions to the corteX error hierarchy via `_classify_gemini_error()`:

| Gemini Exception | corteX Error | Retry? |
|-----------------|-------------|--------|
| `ResourceExhausted` / 429 / quota | `RateLimitError` | Yes |
| `Unauthenticated` / 401/403 | `AuthenticationError` | No |
| `ServiceUnavailable` / 503 | `ServiceUnavailableError` | Yes |
| `InvalidArgument` / 400 (context) | `ContextLengthExceededError` | No |
| `InvalidArgument` / 400 (other) | `InvalidRequestError` | No |

---

## Helper Functions

### `_build_system_instruction`

```python
def _build_system_instruction(
    explicit_instruction: Optional[str],
    messages: List[LLMMessage],
) -> Optional[str]
```

Combines system messages from the message list with the explicit `system_instruction` parameter. This prevents system messages from being silently dropped during message conversion (the Gemini API handles system instructions separately from conversation content).

### `_is_gemini3_model`

```python
def _is_gemini3_model(model: str) -> bool
```

Returns `True` if the model ID contains `"gemini-3"`. Used for temperature override logic.

---

## See Also

- [BaseLLMProvider API](./base.md)
- [Gemini Provider Guide](../../guides/providers/gemini.md)
- [LLM Router API](./router.md)
