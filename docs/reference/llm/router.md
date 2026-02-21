# LLM Router API Reference

## Module: `corteX.core.llm.router`

The LLM Router is the central request dispatcher for all LLM calls in corteX. It selects the best provider and model based on task classification, weight-based scoring, provider health, cost constraints, and required capabilities.

Brain analogy: The thalamus routes sensory signals to the right cortical area.

## Classes

### `TaskType`

String constants for task classification used in model routing.

#### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `PLANNING` | `"planning"` | Strategic planning and architecture tasks |
| `CODING` | `"coding"` | Code generation, debugging, implementation |
| `SUMMARIZATION` | `"summarization"` | Text summarization and distillation |
| `VALIDATION` | `"validation"` | Verification, review, correctness checking |
| `CONVERSATION` | `"conversation"` | General chat and creative tasks |
| `TOOL_USE` | `"tool_use"` | Function/tool calling tasks |
| `REASONING` | `"reasoning"` | Analysis, explanation, logical reasoning |

---

### `RoutingDecision`

**Type**: `@dataclass`

Explains why a particular model was selected by the router.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `provider` | `ProviderType` | The selected provider (openai, gemini, anthropic, local) |
| `model` | `str` | The selected model ID |
| `reason` | `str` | Human-readable reason for selection (e.g., `"weight_based_coding"`) |
| `confidence` | `float` | Confidence score for this routing decision [0.0, 1.0] |
| `alternatives` | `List[str]` | Up to 2 alternative model IDs for fallback |

---

### `ProviderConfig`

**Type**: `@dataclass`

Configuration for registering a single LLM provider with the router.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `provider_type` | `ProviderType` | Provider type enum (OPENAI, GEMINI, ANTHROPIC, LOCAL) |
| `api_key` | `str` | API key for authentication |
| `base_url` | `Optional[str]` | Custom API endpoint (for local/proxy setups) |
| `default_model` | `Optional[str]` | Default model ID for this provider |
| `organization` | `Optional[str]` | Organization ID (OpenAI only) |

---

### `LLMRouter`

Intelligent multi-provider router with circuit breaking, rate limiting, cost tracking, and cognitive task classification.

#### Constructor

```python
LLMRouter(
    circuit_failure_threshold: int = 3,
    circuit_recovery_timeout: float = 30.0,
    default_rpm: int = 60,
    model_registry: Optional[ModelRegistry] = None,
    cost_tracker: Optional[CostTracker] = None,
    data_classification_enabled: bool = True,
)
```

**Parameters**:

- `circuit_failure_threshold` (int, default=3): Number of consecutive failures before circuit breaker opens
- `circuit_recovery_timeout` (float, default=30.0): Seconds before circuit breaker allows a probe request
- `default_rpm` (int, default=60): Default requests-per-minute limit per provider
- `model_registry` (`Optional[ModelRegistry]`): External YAML model registry for role mappings and cost estimation
- `cost_tracker` (`Optional[CostTracker]`): Cost tracker instance for budget enforcement
- `data_classification_enabled` (bool, default=True): Enable data classification enforcement to block sensitive data from being sent to cloud models

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `registry` | `Optional[ModelRegistry]` | Access the model registry (if configured) |
| `classifier` | `CognitiveClassifier` | Access the cognitive classifier |
| `cost_tracker` | `CostTracker` | Access the cost tracker |
| `data_classification_enabled` | `bool` | Whether data classification enforcement is active |
| `pii_protection_enabled` | `bool` | Whether PII tokenization is active |
| `pii_tokenizer` | `PIITokenizer` | Access the PII tokenizer instance |

#### Methods

##### `register_provider`

```python
def register_provider(self, config: ProviderConfig) -> None
```

Register an LLM provider. Lazily instantiates the appropriate provider class based on `config.provider_type`.

**Parameters**:

- `config` (`ProviderConfig`): Provider configuration including type, API key, and optional settings

**Example**:

```python
from corteX.core.llm.router import LLMRouter, ProviderConfig
from corteX.core.llm.base import ProviderType

router = LLMRouter()
router.register_provider(ProviderConfig(
    provider_type=ProviderType.OPENAI,
    api_key="sk-...",
    default_model="gpt-4o",
))
router.register_provider(ProviderConfig(
    provider_type=ProviderType.GEMINI,
    api_key="AIza...",
    default_model="gemini-3-pro-preview",
))
```

##### `set_provider_rpm`

```python
def set_provider_rpm(self, provider_type: ProviderType, rpm: int) -> None
```

Configure the requests-per-minute limit for a specific provider.

##### `set_model_weights`

```python
def set_model_weights(self, weights: Dict[str, Dict[str, float]]) -> None
```

Set task-type to model scoring weights for routing decisions.

**Example**:

```python
router.set_model_weights({
    "planning": {"gpt-4o": 0.9, "gemini-3-pro-preview": 0.85},
    "coding": {"gpt-4o": 0.95, "gemini-3-flash-preview": 0.7},
})
```

##### `set_role_model`

```python
def set_role_model(self, role: str, model: str) -> None
```

Set the default model for any of the 8 roles: `orchestrator`, `worker`, `summarizer`, `judge`, `embedder`, `classifier`, `creative`, `fast`.

##### `get_role_model`

```python
def get_role_model(self, role: str) -> Optional[str]
```

Get the default model for a role. Returns `None` if not configured.

##### `set_session_context`

```python
def set_session_context(self, session_id: str, tenant_id: Optional[str] = None) -> None
```

Set session and tenant context for cost tracking and budget enforcement.

##### `set_orchestrator_model`

```python
def set_orchestrator_model(self, model: str) -> None
```

Set the smartest model for orchestration tasks. This is a convenience shortcut for `set_role_model("orchestrator", model)`.

##### `get_orchestrator_model`

```python
def get_orchestrator_model(self) -> Optional[str]
```

Get the current default orchestrator model name. Returns `None` if not configured.

##### `set_worker_model`

```python
def set_worker_model(self, model: str) -> None
```

Set the fastest model for worker/background tasks. This is a convenience shortcut for `set_role_model("worker", model)`.

##### `get_worker_model`

```python
def get_worker_model(self) -> Optional[str]
```

Get the current default worker model name. Returns `None` if not configured.

##### `set_data_classification_enabled`

```python
def set_data_classification_enabled(self, enabled: bool) -> None
```

Enable or disable pre-call data classification enforcement. When enabled, the router classifies message content before each LLM call and blocks requests if the data sensitivity level (e.g., CONFIDENTIAL, RESTRICTED) is too high for the target model's deployment type (e.g., cloud).

**Example**:

```python
router.set_data_classification_enabled(False)  # Disable for local-only deployments
```

##### `set_pii_protection_enabled`

```python
def set_pii_protection_enabled(self, enabled: bool) -> None
```

Enable or disable PII tokenization before LLM calls. When enabled, personally identifiable information (emails, phone numbers, etc.) in messages is replaced with opaque tokens before being sent to the LLM, and restored in the response. Requires a tenant context to be set via `set_session_context()`.

**Example**:

```python
router.set_pii_protection_enabled(True)
router.set_session_context("session-1", tenant_id="acme-corp")
# PII in messages will now be tokenized before LLM calls
```

##### `generate`

```python
async def generate(
    self,
    messages: List[LLMMessage],
    model: Optional[str] = None,
    temperature: Optional[float] = None,
    max_tokens: Optional[int] = None,
    tools: Optional[List[ToolDefinition]] = None,
    response_format: Optional[Dict[str, Any]] = None,
    thinking: bool = False,
    thinking_budget: Optional[int] = None,
    system_instruction: Optional[str] = None,
    prefer_speed: bool = False,
    role: str = "worker",
    top_p: Optional[float] = None,
    top_k: Optional[int] = None,
    frequency_penalty: Optional[float] = None,
    presence_penalty: Optional[float] = None,
) -> LLMResponse
```

Main entry point. Routes the request to the best model and generates a response. Temperature is auto-selected per task type if not explicitly provided. Includes automatic fallback on failure and proactive rate limit avoidance.

**Parameters**:

- `messages` (`List[LLMMessage]`): Conversation messages
- `model` (`Optional[str]`): Explicit model override (bypasses routing)
- `temperature` (`Optional[float]`): Sampling temperature. `None` = auto-select based on task type and provider
- `max_tokens` (`Optional[int]`): Maximum output tokens
- `tools` (`Optional[List[ToolDefinition]]`): Available tools for function calling
- `response_format` (`Optional[Dict[str, Any]]`): Structured output format specification
- `thinking` (bool): Enable extended thinking mode
- `thinking_budget` (`Optional[int]`): Token budget for thinking
- `system_instruction` (`Optional[str]`): System prompt
- `prefer_speed` (bool): Bias routing toward faster models
- `role` (str): Role for default model selection (`"orchestrator"`, `"worker"`, etc.)
- `top_p` (`Optional[float]`): Nucleus sampling threshold
- `top_k` (`Optional[int]`): Top-k sampling (Gemini only)
- `frequency_penalty` (`Optional[float]`): Frequency penalty (OpenAI only)
- `presence_penalty` (`Optional[float]`): Presence penalty (OpenAI only)

**Returns**: `LLMResponse` with content, tool calls, usage, and metadata.

**Raises**: `RuntimeError` if budget is exceeded or no provider is available.

**Example**:

```python
from corteX.core.llm.base import LLMMessage

response = await router.generate(
    messages=[LLMMessage(role="user", content="Write a Python REST API")],
    role="orchestrator",
    thinking=True,
)
print(response.content)
print(response.usage)  # {"input_tokens": 15, "output_tokens": 450}
```

##### `generate_stream`

```python
async def generate_stream(
    self,
    messages: List[LLMMessage],
    model: Optional[str] = None,
    temperature: Optional[float] = None,
    tools: Optional[List[ToolDefinition]] = None,
    system_instruction: Optional[str] = None,
    prefer_speed: bool = False,
    role: str = "worker",
    top_p: Optional[float] = None,
    top_k: Optional[int] = None,
) -> AsyncIterator[StreamChunk]
```

Route and stream response chunks. Includes fallback on stream failure.

**Returns**: `AsyncIterator[StreamChunk]` yielding content deltas, tool call deltas, and finish reason.

##### `generate_structured`

```python
async def generate_structured(
    self,
    messages: List[LLMMessage],
    response_model: type[BaseModel],
    model: Optional[str] = None,
    system_instruction: Optional[str] = None,
    role: str = "worker",
) -> BaseModel
```

Route and generate a Pydantic-validated structured response.

**Returns**: An instance of the provided `response_model` class.

---

## Constants

### `TASK_TEMPERATURE`

```python
TASK_TEMPERATURE: dict[str, float]
```

Default temperature per task type for OpenAI and generic providers.

| Task Type | Temperature | Rationale |
|-----------|-------------|-----------|
| planning | 0.7 | Structured but creative |
| coding | 0.3 | Deterministic, correct code |
| summarization | 0.3 | Faithful to source |
| validation | 0.1 | Highly deterministic |
| conversation | 0.8 | Natural, varied |
| tool_use | 0.2 | Precise function calls |
| reasoning | 0.5 | Balanced |

### `GEMINI3_TEMPERATURE`

Gemini 3 models default to `1.0` for most tasks per Google's recommendation. Only `validation` uses `0.5`.

### `CLAUDE_TEMPERATURE`

Claude-specific temperature defaults. Extended thinking forces `1.0` regardless.

---

## Routing Logic

The router selects models using this scoring formula:

```
final_score = weight_score + speed_bonus - failure_penalty + latency_bonus
```

Where:
- `weight_score`: From `set_model_weights()` configuration (default 0.5)
- `speed_bonus`: +0.3 for fast tier when `prefer_speed=True`, -0.3 for slow tier
- `failure_penalty`: `min(failures * 0.15, 0.6)` based on recent failure count
- `latency_bonus`: +0.1 for avg latency < 1s, -0.1 for avg > 5s

---

## See Also

- [LLM Routing Concept Guide](../../concepts/llm-routing.md)
- [Multi-Provider Failover Tutorial](../../tutorials/multi-provider.md)
- [CognitiveClassifier API](./classifier.md)
- [CostTracker API](./cost-tracker.md)
- [CircuitBreaker / RateLimiter API](./resilience.md)
