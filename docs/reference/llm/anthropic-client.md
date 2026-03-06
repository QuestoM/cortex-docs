# Anthropic Provider API Reference

## Module: `corteX.core.llm.anthropic_client`

Anthropic/Claude LLM provider supporting Opus, Sonnet, and Haiku model families with tool use, extended thinking, and streaming.

## Classes

### `AnthropicProvider`

**Extends**: `BaseLLMProvider`

#### Constructor

```python
AnthropicProvider(
    api_key: str = "",
    default_model: str = "claude-sonnet-4-5",
    base_url: Optional[str] = None,
    bedrock: bool = False,
    aws_region: Optional[str] = None,
    vertex_ai: bool = False,
    gcp_project: Optional[str] = None,
    gcp_location: Optional[str] = None,
)
```

**Parameters**:

- `api_key` (str, default=`""`): Anthropic API key. Not needed for Bedrock or Vertex AI modes
- `default_model` (str, default=`"claude-sonnet-4-5"`): Default model for generation calls
- `base_url` (`Optional[str]`): Custom API endpoint (for proxies or enterprise deployments)
- `bedrock` (bool, default=`False`): Use Amazon Bedrock with AWS IAM authentication
- `aws_region` (`Optional[str]`): AWS region for Bedrock (default `"us-east-1"`)
- `vertex_ai` (bool, default=`False`): Use Google Vertex AI with Application Default Credentials (ADC)
- `gcp_project` (`Optional[str]`): GCP project ID (required when `vertex_ai=True`)
- `gcp_location` (`Optional[str]`): GCP region for Vertex AI (default `"us-east5"`)

**Authentication Modes**:

- **API Key mode** (default): `AnthropicProvider(api_key="sk-ant-...")`
- **Amazon Bedrock mode**: `AnthropicProvider(bedrock=True, aws_region="us-east-1")`. Uses AWS IAM credentials
- **Google Vertex AI mode**: `AnthropicProvider(vertex_ai=True, gcp_project="my-project")`. Uses Application Default Credentials (ADC)

#### Registered Models

| Model ID | Context | Speed | Intelligence | Cost (input/output per 1K) | Thinking |
|----------|---------|-------|--------------|----------------------------|----------|
| `claude-opus-4-6` | 200K | slow | frontier | $0.005 / $0.025 | Yes |
| `claude-sonnet-4-5` | 200K | medium | high | $0.003 / $0.015 | Yes |
| `claude-haiku-4-5` | 200K | fast | medium | $0.001 / $0.005 | No |

All models support: TEXT, VISION, TOOL_CALLING, STREAMING.

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

Generate a completion using the Anthropic Messages API. Includes automatic retry with backoff for transient errors.

**Extended Thinking**: When `thinking=True` with a `thinking_budget`, the provider sends `thinking: {"type": "enabled", "budget_tokens": N}` and forces `temperature=1.0` (required by Anthropic's API). Thinking content is returned in `LLMResponse.thinking`.

**Message Conversion**: System messages are extracted from the message list and combined into the top-level `system` parameter. Tool results are converted to Anthropic's `tool_result` content blocks. Assistant messages with tool calls are converted to `tool_use` blocks.

**Parameters**:

- `messages` (`List[LLMMessage]`): Conversation messages
- `model` (`Optional[str]`): Model override (defaults to `default_model`)
- `temperature` (float): Sampling temperature (forced to 1.0 when thinking is enabled)
- `max_tokens` (`Optional[int]`): Max output tokens (defaults to 4096)
- `tools` (`Optional[List[ToolDefinition]]`): Tool definitions (converted to Anthropic's `input_schema` format)
- `thinking` (bool): Enable extended thinking
- `thinking_budget` (`Optional[int]`): Token budget for thinking chain
- `top_p` (`Optional[float]`): Nucleus sampling
- `top_k` (`Optional[int]`): Top-k sampling (natively supported)

**Returns**: `LLMResponse` with content, thinking, tool calls, and usage.

**Finish Reason Mapping**:

| Anthropic `stop_reason` | corteX `finish_reason` |
|------------------------|----------------------|
| `end_turn` | `stop` |
| `max_tokens` | `length` |
| `tool_use` | `tool_calls` |
| `stop_sequence` | `stop` |

**Example**:

```python
from corteX.core.llm.anthropic_client import AnthropicProvider
from corteX.core.llm.base import LLMMessage

provider = AnthropicProvider(api_key="sk-ant-...")

response = await provider.generate(
    messages=[LLMMessage(role="user", content="Analyze this codebase")],
    thinking=True,
    thinking_budget=10000,
)
print(response.thinking)  # Reasoning chain
print(response.content)   # Final answer
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

Stream response chunks using Anthropic's `messages.stream()` context manager. Emits `StreamChunk` for text deltas, thinking deltas, tool call JSON deltas, and message stop events.

**Note**: When `thinking=True`, the streaming implementation uses a hardcoded `thinking_budget=10000` tokens. This value is not configurable in streaming mode (unlike `generate()` where you can pass a custom `thinking_budget`).

**Stream Event Types**:

| Event Type | Delta Type | StreamChunk Field |
|-----------|-----------|-------------------|
| `content_block_delta` | `text_delta` | `content` |
| `content_block_delta` | `thinking_delta` | `thinking_delta` |
| `content_block_delta` | `input_json_delta` | `tool_call_delta` |
| `message_stop` | -- | `finish_reason="stop"` |

**Example**:

```python
async for chunk in provider.generate_stream(
    messages=[LLMMessage(role="user", content="Explain recursion")],
):
    if chunk.content:
        print(chunk.content, end="")
    if chunk.thinking_delta:
        print(f"[thinking] {chunk.thinking_delta}")
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

Generate a structured response by injecting the JSON schema into the system instruction and parsing the JSON output. Uses `temperature=0.2` for deterministic output.

##### `health_check`

```python
async def health_check(self) -> bool
```

Verify connectivity by sending a minimal "ping" message. Returns `True` on success.

---

## Helper Functions

### `_build_kwargs`

```python
def _build_kwargs(
    model: str, msgs: List[Dict], max_tokens: Optional[int],
    system: Optional[str], tools: Optional[List[ToolDefinition]],
    temperature: float, thinking: bool, thinking_budget: Optional[int] = None,
    top_p: Optional[float] = None, top_k: Optional[int] = None,
) -> Dict[str, Any]
```

Build the kwargs dict for both `messages.create` and `messages.stream`. Handles thinking configuration (forces `temperature=1.0` when thinking is enabled with a budget), tool conversion to Anthropic's `input_schema` format, and sampling parameters. Default `max_tokens` is 4096 when not explicitly provided.

---

## Error Classification

| Anthropic Exception | corteX Error | Retry? |
|--------------------|-------------|--------|
| `RateLimitError` / 429 | `RateLimitError` | Yes |
| `AuthenticationError` / 401 | `AuthenticationError` | No |
| `PermissionDeniedError` / 403 | `AuthenticationError` | No |
| `InternalServerError` / `OverloadedError` / 503 | `ServiceUnavailableError` | Yes |
| `BadRequestError` / 400 (context) | `ContextLengthExceededError` | No |
| `BadRequestError` / 400 (other) | `InvalidRequestError` | No |

---

## See Also

- [BaseLLMProvider API](./base.md)
- [Switching Providers Guide](../../guides/providers/switching-providers.md)
- [LLM Router API](./router.md)
