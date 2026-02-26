# corteX SDK: LLM Temperature Configuration Guide

> Research date: 2026-02-09
> Context: corteX is an AI Agent SDK. Agents perform diverse tasks (planning, tool calling, code generation, summarization, conversation). The SDK should auto-configure temperature based on task classification to optimize output quality.

---

## 1. What Temperature Means Technically

### 1.1 Softmax Scaling

Temperature (T) is a hyperparameter that controls the randomness of LLM output by scaling the logits before the softmax function is applied.

**The math:**

Given a vector of raw logits `z` from the model's final layer, the probability of selecting token `i` is:

```
P(token_i) = exp(z_i / T) / sum(exp(z_j / T) for all j)
```

Where `T` is the temperature parameter.

**What this means:**

- **T = 1.0** -- Standard softmax. The probability distribution reflects the model's learned confidence as-is.
- **T < 1.0** (e.g., 0.2) -- Dividing logits by a small number amplifies differences between them. High-probability tokens become even more dominant. The distribution becomes **sharper/peakier**. Output is more deterministic and repetitive.
- **T > 1.0** (e.g., 1.5) -- Dividing logits by a larger number compresses differences between them. Low-probability tokens get a larger relative share. The distribution becomes **flatter/more uniform**. Output is more creative, diverse, and surprising.
- **T = 0.0** -- Greedy decoding. Always select the single highest-probability token. Fully deterministic (in theory; in practice, floating-point math and batching can introduce minor non-determinism).

### 1.2 Practical Effect

| Temperature | Distribution Shape | Behavior | Risk |
|---|---|---|---|
| 0.0 | Point mass (greedy) | Always picks top token | Repetitive, can loop |
| 0.1-0.3 | Very peaked | Highly predictable | May miss valid alternatives |
| 0.4-0.6 | Moderately peaked | Balanced | Safe default for most tasks |
| 0.7-0.9 | Broader | More varied, natural-sounding | Occasional off-topic tokens |
| 1.0 | Model default | Full model distribution | Model-dependent behavior |
| 1.1-2.0 | Very flat | Highly creative/random | Incoherent, hallucinations |

### 1.3 Interaction with Top-P (Nucleus Sampling)

Temperature and top-p both affect token selection but work differently:
- **Temperature** reshapes the entire probability distribution
- **Top-p** truncates the distribution to the smallest set of tokens whose cumulative probability exceeds `p`

Best practice across all major providers (OpenAI, Anthropic, Google): **Adjust temperature OR top-p, not both simultaneously.** Adjusting both creates unpredictable interactions.

---

## 2. Recommended Temperature Settings by Task Type

### 2.1 Master Reference Table

| Task Type | Recommended T | Top-P | Rationale |
|---|---|---|---|
| **Tool calling / Function calling** | 0.0-0.2 | 0.95 | Must produce valid JSON, correct function names, exact parameter formats. Any creativity risks parse failures. |
| **Validation / Verification** | 0.0-0.1 | 0.95 | Binary decisions (pass/fail) must be consistent and reproducible. |
| **Code generation** (boilerplate) | 0.0-0.3 | 0.95 | Syntactically correct, idiomatic code. Errors from randomness are expensive. |
| **Code generation** (creative/exploration) | 0.4-0.6 | 0.95 | When exploring different implementation approaches, moderate creativity helps. |
| **Data extraction / Structured output** | 0.0-0.2 | 0.95 | JSON, CSV, SQL generation must be format-perfect. |
| **Classification / Categorization** | 0.0-0.3 | 0.95 | Stable decision boundaries. Ambiguous cases may benefit from 0.3-0.5. |
| **Orchestration / Planning** | 0.3-0.5 | 0.95 | Plans should be logical and consistent but allow consideration of alternative strategies. |
| **Summarization / Analysis** | 0.3-0.6 | 0.95 | Must accurately represent source material while remaining concise and coherent. |
| **Conversational AI / Chatbots** | 0.6-0.8 | 0.95 | Natural, varied responses without sacrificing coherence. Customer service: 0.6-0.7. |
| **Creative writing / Brainstorming** | 0.7-1.0 | 0.95 | Encourages exploration of less obvious word choices and novel ideas. |

### 2.2 Detailed Notes Per Task

#### Tool Calling / Function Calling (T: 0.0-0.2)

This is the most critical task type for corteX agents. Tool calls must:
- Select the correct function name from available tools
- Generate valid JSON arguments matching the function schema
- Be deterministic -- the same context should produce the same tool call

A temperature of 0.0-0.2 ensures the model picks the highest-confidence function and produces well-formed arguments. Higher temperatures risk generating invalid function names or malformed parameter JSON that will cause runtime errors.

**Exception: Gemini 3 -- keep at 1.0** (see Section 3).

#### Orchestration / Planning (T: 0.3-0.5)

Agent orchestration involves deciding:
- Which sub-agents to invoke
- In what order
- What information to pass between them

A moderate temperature allows the planner to consider multiple valid strategies without becoming chaotic. Too low (0.0) can cause the planner to get stuck in repetitive patterns; too high (0.8+) can produce incoherent plans.

#### Code Generation (T: 0.0-0.6, task-dependent)

For boilerplate and known patterns (0.0-0.3):
- CRUD operations, API clients, configuration files
- Syntax correctness is paramount
- Higher temperatures increase syntax errors, logical inconsistencies, or non-functional code

For exploratory coding (0.4-0.6):
- Algorithm design, refactoring suggestions
- Moderate creativity finds better solutions
- Still constrained enough to produce working code

#### Summarization (T: 0.3-0.6)

Must balance accuracy (favoring low T) with readable, non-repetitive prose (favoring moderate T). A summary that is accurate but robotic at T=0.0 is less useful than one that is accurate and well-written at T=0.4.

#### Validation / Verification (T: 0.0-0.1)

When the agent is checking whether output meets criteria (schema validation, safety checks, compliance verification), the response must be consistent and reproducible. This is essentially a classification task with high stakes.

---

## 3. Model-Specific Notes

### 3.1 Google Gemini 3 (2026)

**Critical: Keep temperature at 1.0 for Gemini 3 models.**

Google's official documentation strongly recommends keeping temperature at the default value of 1.0 for all Gemini 3 variants (Flash, Pro, Ultra). This is a significant departure from previous models and other providers.

**Why:**
- Gemini 3's reasoning capabilities are optimized internally for T=1.0
- Setting temperature below 1.0 can cause **looping behavior** (the model repeats itself)
- Performance degrades on complex mathematical and reasoning tasks at lower temperatures
- The model's internal sampling and reasoning mechanisms already handle determinism appropriately at T=1.0

**What to do in corteX:**
- When `model.startswith("gemini-3")`, always set `temperature=1.0` regardless of task type
- Do NOT apply the task-based temperature table above to Gemini 3
- If deterministic output is needed, use `top_k=1` instead of lowering temperature (test thoroughly)
- Remove any explicit temperature settings that were carried over from Gemini 2.x configurations

**Additional Gemini 3 requirement for tool calling:**
- Thought signatures must be passed back between API calls -- these are encrypted representations of the model's reasoning state and are mandatory for proper function calling behavior

**Source:** [Gemini 3 Developer Guide](https://ai.google.dev/gemini-api/docs/gemini-3), [Gemini 3 Prompting Guide (Vertex AI)](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/start/gemini-3-prompting-guide)

### 3.2 OpenAI GPT-5 / GPT-5 mini

**Default temperature:** 1.0 (API default, not published in ChatGPT UI)

**Recommendations:**
- **Structured output / tool calling:** T=0.0-0.2. GPT-4o supports Structured Outputs (`response_format: { type: "json_schema" }`) which constrain output format independently of temperature, but lower temperature still improves reliability.
- **Code generation:** T=0.0-0.3 for correctness.
- **Creative tasks:** T=0.7-1.0.
- **General Q&A:** T=0.4-0.7.

**Notes:**
- Even at T=0.0, GPT-4o is not fully deterministic due to floating-point non-determinism in GPU computation.
- For structured output, combine low temperature with JSON schema enforcement for best results.
- Do not set both `temperature` and `top_p` -- use one or the other.

**Source:** [OpenAI API Reference](https://platform.openai.com/docs/api-reference/chat), [OpenAI Community: Temperature in GPT-5 models](https://community.openai.com/t/temperature-in-gpt-5-models/1337133)

### 3.3 Anthropic Claude (Opus 4.6, Sonnet 4.6, Haiku 4.5)

**Default temperature:** 1.0

**Recommendations from Anthropic:**
- **Analytical / multiple choice:** T closer to 0.0
- **Creative and generative:** T closer to 1.0
- **General production use:** T=0.4-0.5 is a safe middle ground
- Temperature is usually the only sampling parameter you need to adjust

**Notes:**
- Newer Claude models (Opus 4.6, etc.) do NOT allow both `temperature` and `top_p` to be specified simultaneously -- the API will reject the request.
- Even at T=0.0, results are not fully deterministic.
- Claude's extended thinking features operate independently of temperature settings.

**Source:** [Anthropic API Docs: Glossary](https://platform.claude.com/docs/en/about-claude/glossary), [PromptHub: Anthropic Best Practices](https://www.prompthub.us/blog/using-anthropic-best-practices-parameters-and-large-context-windows)

### 3.4 Model Comparison Summary

| Model | Default T | Tool Calling T | Creative T | Special Notes |
|---|---|---|---|---|
| Gemini 3 (all) | 1.0 | **1.0** (do not change) | **1.0** (do not change) | Changing T causes looping/degradation |
| GPT-4o | 1.0 | 0.0-0.2 | 0.7-1.0 | Use JSON schema enforcement alongside |
| GPT-4o-mini | 1.0 | 0.0-0.2 | 0.7-1.0 | Same as GPT-4o |
| Claude Opus 4.6 | 1.0 | 0.0-0.2 | 0.7-1.0 | Cannot set both T and top_p |
| Claude Sonnet 4.6 | 1.0 | 0.0-0.2 | 0.7-1.0 | Cannot set both T and top_p |

---

## 4. How corteX Should Auto-Set Temperature

### 4.1 Architecture: Task-Aware Temperature Router

corteX should implement a temperature router that automatically selects the optimal temperature based on:
1. The **task classification** (what the agent is doing)
2. The **model being used** (model-specific overrides)
3. An optional **user override** (for advanced users)

### 4.2 Task Classification Taxonomy

```python
class AgentTaskType(str, Enum):
    TOOL_CALLING = "tool_calling"           # Function/tool invocation
    ORCHESTRATION = "orchestration"         # Multi-agent planning and routing
    CODE_GENERATION = "code_generation"     # Writing code
    CODE_REVIEW = "code_review"             # Reviewing/analyzing code
    SUMMARIZATION = "summarization"         # Condensing information
    DATA_EXTRACTION = "data_extraction"     # Pulling structured data from text
    CLASSIFICATION = "classification"       # Categorizing inputs
    VALIDATION = "validation"               # Checking correctness
    CONVERSATION = "conversation"           # Natural dialogue
    CREATIVE = "creative"                   # Creative writing, brainstorming
```

### 4.3 Temperature Configuration Map

```python
# Default temperature map (non-Gemini-3 models)
TEMPERATURE_MAP: dict[AgentTaskType, float] = {
    AgentTaskType.TOOL_CALLING:     0.1,
    AgentTaskType.ORCHESTRATION:    0.4,
    AgentTaskType.CODE_GENERATION:  0.2,
    AgentTaskType.CODE_REVIEW:      0.1,
    AgentTaskType.SUMMARIZATION:    0.4,
    AgentTaskType.DATA_EXTRACTION:  0.1,
    AgentTaskType.CLASSIFICATION:   0.1,
    AgentTaskType.VALIDATION:       0.0,
    AgentTaskType.CONVERSATION:     0.7,
    AgentTaskType.CREATIVE:         0.9,
}

# Gemini 3 override -- always 1.0
GEMINI_3_TEMPERATURE = 1.0
```

### 4.4 Implementation: Temperature Router

```python
def resolve_temperature(
    task_type: AgentTaskType,
    model: str,
    user_override: float | None = None,
) -> float:
    """
    Resolve the optimal temperature for a given task and model.

    Priority:
    1. Model-specific hard overrides (e.g., Gemini 3 = 1.0 always)
    2. User-specified override (if provided and model allows it)
    3. Task-based default from TEMPERATURE_MAP
    """
    # Gemini 3 hard override -- Google explicitly warns against changing T
    if _is_gemini_3(model):
        if user_override is not None and user_override != GEMINI_3_TEMPERATURE:
            logger.warning(
                f"Gemini 3 requires temperature=1.0. Ignoring user override "
                f"of {user_override}. See: https://ai.google.dev/gemini-api/docs/gemini-3"
            )
        return GEMINI_3_TEMPERATURE

    # User override takes precedence for non-restricted models
    if user_override is not None:
        return max(0.0, min(2.0, user_override))  # Clamp to valid range

    # Task-based default
    return TEMPERATURE_MAP.get(task_type, 0.5)


def _is_gemini_3(model: str) -> bool:
    """Check if the model is a Gemini 3 variant."""
    gemini_3_prefixes = [
        "gemini-3", "gemini-3.0", "gemini-3-pro", "gemini-3-flash",
        "gemini-3-ultra", "models/gemini-3",
    ]
    model_lower = model.lower()
    return any(model_lower.startswith(prefix) for prefix in gemini_3_prefixes)
```

### 4.5 Integration Points in the Agent Lifecycle

```
User Request
    |
    v
[Task Classifier] --> AgentTaskType
    |
    v
[Temperature Router] --> resolve_temperature(task_type, model, user_override)
    |
    v
[LLM Call] with resolved temperature
    |
    v
[Response Processing]
```

**Where temperature is applied in a typical agent turn:**

1. **Planning step** (orchestration): T=0.4 -- Decide which tools/sub-agents to invoke
2. **Tool selection** (tool_calling): T=0.1 -- Pick the right function and generate arguments
3. **Tool result processing** (summarization): T=0.4 -- Synthesize tool outputs
4. **Response generation** (conversation): T=0.7 -- Generate the final user-facing response

A single agent turn may use **multiple different temperatures** across these steps.

### 4.6 Configuration API for SDK Users

```python
from cortex import Agent, TemperatureConfig

# Option 1: Fully automatic (recommended)
agent = Agent(model="gpt-4o")
# Temperature is auto-set based on task classification

# Option 2: Override for specific task types
agent = Agent(
    model="gpt-4o",
    temperature_config=TemperatureConfig(
        tool_calling=0.0,       # Stricter than default
        conversation=0.8,       # More creative than default
        # All other tasks use defaults
    )
)

# Option 3: Fixed temperature for all tasks (not recommended but supported)
agent = Agent(
    model="gpt-4o",
    temperature=0.3,  # Overrides all task-based defaults
)

# Option 4: Gemini 3 (temperature auto-locked to 1.0)
agent = Agent(model="gemini-3-pro")
# Any temperature setting is ignored with a warning
```

---

## 5. Testing and Validation Strategy

### 5.1 Temperature Evaluation Framework

Before deploying temperature defaults, validate them empirically:

1. **Consistency test:** Run the same prompt 20 times at each temperature. Measure output variance.
2. **Correctness test:** For tool calling and code generation, measure the percentage of outputs that parse/compile successfully.
3. **Quality test:** For creative and conversational tasks, use human evaluation or LLM-as-judge scoring.
4. **Regression test:** When updating temperature defaults, run the full eval suite to ensure no degradation.

### 5.2 Metrics to Track

| Task Type | Primary Metric | Secondary Metric |
|---|---|---|
| Tool calling | Parse success rate (%) | Correct function selection (%) |
| Code generation | Compilation success (%) | Test pass rate (%) |
| Orchestration | Plan validity (%) | Task completion rate (%) |
| Summarization | ROUGE score | Factual accuracy (%) |
| Conversation | User satisfaction (1-5) | Response diversity (unique n-grams) |
| Creative | Novelty score | Coherence score |

### 5.3 A/B Testing in Production

corteX should support A/B testing temperature settings:

```python
agent = Agent(
    model="gpt-4o",
    temperature_config=TemperatureConfig(
        tool_calling=ABTest(control=0.1, variant=0.0, split=0.5),
    )
)
```

This allows SDK users to empirically find their optimal settings.

---

## 6. Key Takeaways

1. **Temperature is not one-size-fits-all.** Different agent tasks require different settings. corteX should auto-configure this.

2. **Gemini 3 is the major exception.** Always use T=1.0. This is non-negotiable per Google's documentation -- lower values cause looping and degradation.

3. **Tool calling demands low temperature** (0.0-0.2) on all non-Gemini-3 models. This is the most common failure mode in agents -- a creative tool call that generates invalid JSON.

4. **A single agent turn uses multiple temperatures.** The planning step, tool calling step, and response generation step each benefit from different settings.

5. **Always allow user overrides** (except where the model vendor explicitly forbids it, like Gemini 3). Power users know their use cases better than defaults.

6. **Test empirically.** The values in this guide are starting points based on industry best practices and vendor documentation. corteX users should validate for their specific domain.

---

## 7. Sources

- [Google: Gemini 3 Developer Guide](https://ai.google.dev/gemini-api/docs/gemini-3)
- [Google: Gemini 3 Prompting Guide (Vertex AI)](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/start/gemini-3-prompting-guide)
- [Google: Experiment with Parameter Values](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/learn/prompts/adjust-parameter-values)
- [Google: Function Calling with Gemini API](https://ai.google.dev/gemini-api/docs/function-calling)
- [Google Developers Blog: Gemini API Updates for Gemini 3](https://developers.googleblog.com/new-gemini-api-updates-for-gemini-3/)
- [LiteLLM: Gemini 3 Support](https://docs.litellm.ai/blog/gemini_3)
- [OpenAI: API Reference - Chat Completions](https://platform.openai.com/docs/api-reference/chat)
- [OpenAI: Structured Outputs Guide](https://platform.openai.com/docs/guides/structured-outputs)
- [OpenAI Community: Temperature in GPT-5 Models](https://community.openai.com/t/temperature-in-gpt-5-models/1337133)
- [Anthropic: Claude API Glossary](https://platform.claude.com/docs/en/about-claude/glossary)
- [PromptHub: Anthropic Best Practices](https://www.prompthub.us/blog/using-anthropic-best-practices-parameters-and-large-context-windows)
- [IBM: What is LLM Temperature?](https://www.ibm.com/think/topics/llm-temperature)
- [Tetrate: LLM Temperature Settings Guide](https://tetrate.io/learn/ai/llm-temperature-guide)
- [Vellum: LLM Temperature Parameters](https://www.vellum.ai/llm-parameters/temperature)
- [Promptfoo: Evaluate LLM Temperature](https://www.promptfoo.dev/docs/guides/evaluate-llm-temperature/)
- [Prompt Engineering Guide: LLM Settings](https://www.promptingguide.ai/introduction/settings)
- [Medium: Stop Using Temperature 1.0 for Code Generation](https://medium.com/@glanzz/stop-using-temperature-1-0-385cb51ac863)
- [Sendbird: Boost AI Agent Performance with LLM Settings](https://sendbird.com/blog/what-is-an-ai-agent/ai-agent-enhancements)
- [big-AGI Issue #953: Temperature Support for Gemini 3](https://github.com/enricoros/big-AGI/issues/953)
