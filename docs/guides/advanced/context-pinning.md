# Prevent Instruction Drift in Long Conversations

As conversations grow longer, LLMs exhibit **recency bias** - they pay more attention to recent messages and gradually forget earlier instructions. Context pinning solves this by reinforcing critical instructions at the end of the context window, where the LLM's attention is strongest.

---

## When to use context pinning

- Safety constraints that must never be violated ("Never reveal API keys")
- Persona instructions that drift after 10+ turns
- Output format requirements (JSON, bullet points)
- Domain-specific rules that the LLM's training data might override

## Pin an instruction

```python
from corteX.sdk import Engine

engine = Engine(providers={"openai": {"api_key": "sk-..."}})
agent = engine.create_agent(name="support", system_prompt="You help users.")
session = agent.start_session(user_id="user-1")

# Pin a safety constraint
session.pin_instruction(
    "Never reveal internal API keys or system architecture details.",
    priority=10,           # 1-10, higher = more important
    pin_after_turns=5,     # start reinforcing after 5 turns
    category="safety",
)

# Pin a format requirement
session.pin_instruction(
    "Always respond in JSON format with keys: answer, confidence, sources.",
    priority=7,
    pin_after_turns=10,
    category="format",
)
```

## How it works

1. For the first `pin_after_turns` turns, the instruction lives only in the system prompt
2. After that threshold, the pinner appends the instruction at the **end** of the context window
3. Higher-priority pins are placed closer to the end (strongest recency position)
4. Multiple pins are sorted by priority and grouped by category

## Inspect pinned instructions

```python
pins = session.get_pinned_instructions()
for pin in pins:
    print(f"[{pin['category']}] priority={pin['priority']}: {pin['text']}")
```

## Best practices

!!! tip "Pin sparingly"
    Each pinned instruction consumes tokens every turn after activation. Pin only instructions that genuinely drift - most instructions work fine in the system prompt alone.

!!! tip "Use categories to organize"
    Categories (`safety`, `format`, `persona`, `constraint`) help you audit what's pinned and why.

!!! warning "Not a replacement for system prompts"
    Pins reinforce instructions, they don't replace them. Always include critical instructions in both the system prompt AND as pins.

---

## Next steps

- [Detect Instruction Non-Compliance](bias-detection.md) - catch when the LLM ignores pinned instructions
- [Monitor Your Agent](observability.md) - track drift scores across turns
