# Model Registry API Reference

## Module: `corteX.core.llm.registry`

External YAML-based model registry with hot-reload. Maps model IDs to metadata (pricing, capabilities, quality ratings) and maps roles to models with tenant-level overrides.

Brain analogy: The sensory cortex's map of available processing regions.

## Enums

### `ModelRole`

```python
class ModelRole(str, Enum)
```

Eight specialized roles for model routing.

| Value | Description |
|-------|-------------|
| `ORCHESTRATOR` | Primary reasoning/planning model |
| `WORKER` | Fast execution model for subtasks |
| `SUMMARIZER` | Text condensation and distillation |
| `JUDGE` | Evaluation and quality assessment |
| `EMBEDDER` | Text embedding generation |
| `CLASSIFIER` | Classification and categorization |
| `CREATIVE` | Creative writing and brainstorming |
| `FAST` | Lowest-latency model for simple tasks |

---

## Data Classes

### `ModelFeatures`

**Type**: `@dataclass(frozen=True)`

Capability flags for a registered model.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `tools` | `bool` | `False` | Supports function/tool calling |
| `vision` | `bool` | `False` | Supports image/multimodal input |
| `structured_output` | `bool` | `False` | Supports JSON schema output |
| `thinking` | `bool` | `False` | Supports extended thinking |
| `streaming` | `bool` | `True` | Supports streaming responses |
| `code_execution` | `bool` | `False` | Supports native code execution |

### `ModelEntry`

**Type**: `@dataclass`

Complete metadata for a single model in the registry.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | -- | Model identifier (e.g., `"gpt-4o"`) |
| `provider` | `str` | -- | Provider name |
| `context_window` | `int` | `128_000` | Maximum context length |
| `input_price_per_1k` | `float` | `0.0` | Cost per 1K input tokens (USD) |
| `output_price_per_1k` | `float` | `0.0` | Cost per 1K output tokens (USD) |
| `features` | `ModelFeatures` | `ModelFeatures()` | Capability flags |
| `quality_ratings` | `Dict[str, float]` | `{}` | Quality scores by task type |
| `speed_rating` | `float` | `0.5` | Speed score [0, 1] |
| `reliability_score` | `float` | `1.0` | Reliability score [0, 1] |
| `deprecated` | `bool` | `False` | Whether the model is deprecated |
| `deprecation_notice` | `str` | `""` | Deprecation message |

### `RoleMapping`

**Type**: `@dataclass`

Maps a role to a primary model and ordered fallbacks.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `primary` | `str` | -- | Primary model ID for this role |
| `fallbacks` | `List[str]` | `[]` | Ordered fallback model IDs |

---

## Classes

### `ModelRegistry`

External model registry loaded from YAML with hot-reload support.

**Resolution order**:

1. `CORTEX_MODELS_PATH` environment variable
2. `./cortex_models.yaml` (project root)
3. `~/.cortex/models.yaml` (user home)
4. Built-in empty defaults

Hot-reload: Checks file mtime every 60 seconds, reloads if changed.

#### Constructor

```python
ModelRegistry(
    config_paths: Optional[List[str]] = None,
    auto_reload: bool = True,
)
```

**Parameters**:

- `config_paths` (`Optional[List[str]]`): Custom YAML search paths. If not provided, uses the resolution order above
- `auto_reload` (bool, default=`True`): Enable automatic hot-reload on file changes

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `model_count` | `int` | Number of registered models |
| `loaded_path` | `Optional[str]` | Path of the currently loaded YAML file |

#### Methods

##### `get_model`

```python
def get_model(self, model_id: str) -> Optional[ModelEntry]
```

Get a single model by ID. Triggers hot-reload check.

##### `get_all_models`

```python
def get_all_models(self) -> Dict[str, ModelEntry]
```

Get all registered models as a dictionary.

##### `get_models_for_role`

```python
def get_models_for_role(
    self, role: str, tenant_id: Optional[str] = None,
) -> List[ModelEntry]
```

Get primary + fallback models for a role, sorted by preference. Tenant overrides take precedence over global mappings. Deprecated models are excluded.

##### `estimate_cost`

```python
def estimate_cost(self, model_id: str, input_tokens: int, output_tokens: int) -> float
```

Estimate cost in USD for a given token count. Returns `0.0` if the model is not in the registry.

##### `register_model`

```python
def register_model(self, entry: ModelEntry) -> None
```

Programmatically register a model (e.g., for local models discovered at runtime).

##### `set_role_mapping`

```python
def set_role_mapping(
    self, role: str, primary: str, fallbacks: Optional[List[str]] = None,
    tenant_id: Optional[str] = None,
) -> None
```

Set or override a role mapping programmatically. If `tenant_id` is provided, creates a tenant-specific override.

##### `reload_if_changed`

```python
def reload_if_changed(self) -> bool
```

Check file mtime and reload if changed. Returns `True` if reloaded. Only checks once per 60 seconds.

##### `force_reload`

```python
def force_reload(self) -> None
```

Force immediate reload of the registry file.

---

## YAML Configuration Format

```yaml
# cortex_models.yaml
models:
  gpt-4o:
    provider: openai
    context_window: 128000
    input_price_per_1k: 0.0025
    output_price_per_1k: 0.01
    features:
      tools: true
      vision: true
      structured_output: true
      thinking: false
    quality_ratings:
      coding: 0.9
      reasoning: 0.85
    speed_rating: 0.6

  gemini-3-pro-preview:
    provider: gemini
    context_window: 1000000
    input_price_per_1k: 0.002
    output_price_per_1k: 0.012
    features:
      tools: true
      vision: true
      thinking: true
      code_execution: true

roles:
  orchestrator:
    primary: gemini-3-pro-preview
    fallbacks: [gpt-4o, gemini-2.5-pro]
  worker:
    primary: gemini-3-flash-preview
    fallbacks: [gpt-4o-mini]

tenant_overrides:
  acme_corp:
    orchestrator:
      primary: gpt-4o
      fallbacks: [gemini-3-pro-preview]
```

---

## See Also

- [LLM Router API](./router.md) - Uses registry for role-based model selection
- [Cost Tracker API](./cost-tracker.md) - Uses registry for cost estimation
