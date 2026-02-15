# Response & Configuration API Reference

## Module: `corteX.sdk_config`

This module defines all configuration classes and response types used throughout the corteX SDK. These are the data structures you interact with on every agent call.

---

## Response Classes

### `Response`

**Type**: `@dataclass`

The result of an agent run. Returned by `session.run()`.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `content` | `str` | The agent's text response. |
| `metadata` | `ResponseMetadata` | Execution metadata (latency, tokens, goal progress, tools called, etc.). |
| `artifacts` | `List[Dict[str, Any]]` | Structured artifacts produced during execution (code, documents, plans). Default: `[]`. |
| `raw_response` | `Optional[Any]` | The raw LLM response object for advanced inspection. Default: `None`. |

#### Example

```python
response = await session.run("What's the weather?")

# Access the text content
print(response.content)

# Access metadata
print(f"Model: {response.metadata.model_used}")
print(f"Tokens: {response.metadata.tokens_used}")
print(f"Latency: {response.metadata.latency_ms:.0f}ms")

# Check if tools were called
if response.metadata.tools_called:
    print(f"Tools: {', '.join(response.metadata.tools_called)}")

# Check goal progress
if response.metadata.goal_progress > 0:
    print(f"Goal: {response.metadata.goal_progress:.0%}")

# Check for drift or loops
if response.metadata.loop_detected:
    print("WARNING: Loop detected!")
if response.metadata.drift_score > 0.3:
    print(f"WARNING: Drifting from goal (drift={response.metadata.drift_score:.2f})")
```

---

### `ResponseMetadata`

**Type**: `@dataclass`

Metadata about an agent response. Provides observability into the brain-integrated execution pipeline.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `goal_progress` | `float` | `0.0` | Current goal completion progress [0.0, 1.0]. Only meaningful when goal tracking is enabled. |
| `drift_score` | `float` | `0.0` | How far the response drifted from the original goal [0.0, 1.0]. Values > 0.3 indicate significant drift. |
| `loop_detected` | `bool` | `False` | Whether a conversation loop was detected via state hashing. |
| `weights_delta` | `Dict[str, float]` | `{}` | Weight changes applied during this turn. |
| `steps_taken` | `int` | `0` | Number of tool execution rounds in this turn. |
| `model_used` | `str` | `""` | Name of the LLM model used for generation. |
| `tokens_used` | `int` | `0` | Total tokens consumed in this turn. |
| `latency_ms` | `float` | `0.0` | End-to-end latency in milliseconds. |
| `tools_called` | `List[str]` | `[]` | Names of tools called during this turn. |

#### Example

```python
response = await session.run("Analyze the sales data")

meta = response.metadata
print(f"Model: {meta.model_used}")
print(f"Tokens: {meta.tokens_used}")
print(f"Latency: {meta.latency_ms:.0f}ms")
print(f"Steps: {meta.steps_taken}")
print(f"Tools: {meta.tools_called}")
print(f"Goal: {meta.goal_progress:.0%}")
print(f"Drift: {meta.drift_score:.2f}")
print(f"Loop: {meta.loop_detected}")
```

---

## Configuration Classes

### `WeightConfig`

**Type**: `pydantic.BaseModel`

Configuration for the behavioral weight system. Controls how the agent behaves (verbosity, formality, autonomy) and how fast it learns from feedback.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `verbosity` | `float` | `0.0` | Response detail level. Positive = verbose, negative = concise. Range: [-1.0, 1.0]. |
| `formality` | `float` | `0.0` | Language formality. Positive = formal, negative = casual. Range: [-1.0, 1.0]. |
| `autonomy` | `float` | `0.5` | Decision independence. High values = agent acts without asking. Range: [0.0, 1.0]. |
| `initiative` | `float` | `0.3` | Proactive suggestion level. High values = agent volunteers suggestions. Range: [0.0, 1.0]. |
| `speed_vs_quality` | `float` | `0.0` | Trade-off preference. Positive = favor quality, negative = favor speed. Range: [-1.0, 1.0]. |
| `tier1_direct` | `bool` | `True` | Enable Tier 1 (direct) feedback processing. |
| `tier2_user_insights` | `bool` | `True` | Enable Tier 2 (user insight) feedback. |
| `tier3_enterprise` | `bool` | `False` | Enable Tier 3 (enterprise-wide) feedback aggregation. |
| `tier4_global` | `bool` | `False` | Enable Tier 4 (global population) feedback. |
| `behavioral_lr` | `float` | `0.12` | Learning rate for behavioral weight updates. |
| `tool_lr` | `float` | `0.08` | Learning rate for tool preference weight updates. |
| `model_lr` | `float` | `0.04` | Learning rate for model preference weight updates. |

#### Example

```python
from cortex import WeightConfig

# Concise, formal, autonomous agent
config = WeightConfig(
    verbosity=-0.3,
    formality=0.4,
    autonomy=0.8,
    initiative=0.6,
    speed_vs_quality=0.2,
    behavioral_lr=0.15,  # Faster learning
)

agent = engine.create_agent(
    name="assistant",
    system_prompt="You are a professional assistant.",
    weight_config=config,
)
```

---

### `ContextManagementConfig`

**Type**: `pydantic.BaseModel`

Configuration for the Cortical Context Engine (CCE). Controls how conversation history is managed across long sessions (10,000+ steps).

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `enabled` | `bool` | `True` | Enable context management. |
| `token_budget_ratio` | `float` | `0.80` | Fraction of model context window to use [0.0, 1.0]. |
| `hot_ratio` | `float` | `0.40` | Fraction of budget for hot (recent) context. |
| `warm_ratio` | `float` | `0.35` | Fraction of budget for warm (summarized) context. |
| `cold_ratio` | `float` | `0.25` | Fraction of budget for cold (archived) context. |
| `summarize_every_n_steps` | `int` | `20` | Trigger L2 summarization every N conversation steps. |
| `checkpoint_every_n_steps` | `int` | `50` | Create context checkpoints every N steps. |
| `profile` | `str` | `"general"` | Context profile: `"general"`, `"coding"`, or `"research"`. |

#### Example

```python
from cortex import ContextManagementConfig

# Optimized for long coding sessions
config = ContextManagementConfig(
    token_budget_ratio=0.85,
    hot_ratio=0.45,
    warm_ratio=0.30,
    cold_ratio=0.25,
    summarize_every_n_steps=15,
    profile="coding",
)
```

---

### `EnterpriseConfig`

**Type**: `pydantic.BaseModel`

Enterprise-level configuration for safety, compliance, and operational controls.

#### Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `safety_level` | `str` | `"moderate"` | Safety enforcement level: `"strict"`, `"moderate"`, or `"permissive"`. |
| `data_retention` | `str` | `"none"` | Data retention policy: `"none"`, `"session"`, or `"persistent"`. |
| `allowed_models` | `List[str]` | `[]` | Whitelist of allowed model names. Empty = all models allowed. |
| `blocked_topics` | `List[str]` | `[]` | Topics the agent must refuse to discuss. |
| `audit_log` | `bool` | `False` | Enable enterprise audit logging. |
| `feedback_collection` | `str` | `"off"` | Feedback scope: `"off"`, `"enterprise"`, or `"global"`. |
| `max_tokens_per_session` | `int` | `100000` | Maximum tokens allowed per session. |
| `compliance` | `List[str]` | `[]` | Compliance standards to enforce (e.g., `["SOC2", "GDPR", "HIPAA"]`). |

#### Example

```python
from cortex import EnterpriseConfig

config = EnterpriseConfig(
    safety_level="strict",
    audit_log=True,
    blocked_topics=["competitor_analysis", "legal_advice"],
    max_tokens_per_session=50000,
    compliance=["SOC2", "GDPR"],
    allowed_models=["gpt-4o", "gpt-4o-mini"],
)
```

---

## Import Shortcuts

All public types are available from the top-level package:

```python
import cortex

# These are equivalent:
from cortex import Engine, Agent, Response, ResponseMetadata
from cortex import WeightConfig, EnterpriseConfig, ContextManagementConfig

# Or use qualified imports:
from corteX.sdk import Engine, Agent
from corteX.sdk_config import Response, ResponseMetadata, WeightConfig
```

---

## See Also

- [Engine API Reference](./engine.md) - Engine creation
- [Session API Reference](./session.md) - Session execution returning Response
- [Weight Tuning Guide](../guides/config/weight-tuning.md) - How to tune weights
- [Context Profiles Guide](../guides/config/context-profiles.md) - CCE configuration
- [Safety Controls](../enterprise/safety.md) - Enterprise safety details
