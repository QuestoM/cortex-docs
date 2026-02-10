# Build a Customer Support Agent

In this tutorial you will build a complete customer support agent for a fictional e-commerce company called "Acme Store." By the end, you will have an agent that looks up orders, searches FAQs, escalates to a human when needed, adapts its tone through weight tuning, tracks progress through multi-step troubleshooting, and respects enterprise safety boundaries.

---

## What you will build

- A support agent with three tools: order lookup, FAQ search, and human escalation
- Behavioral weight tuning for a formal, helpful conversational tone
- Goal tracking that monitors multi-step troubleshooting progress
- Enterprise safety controls that block sensitive topics
- A diagnostic loop that inspects brain metrics after each interaction

## Prerequisites

- Python 3.11+
- corteX installed: `pip install cortex-engine[openai]`
- An OpenAI API key (set as `OPENAI_API_KEY` or passed directly)

---

## Step 1: Set up the project

Create a new directory and file for your support agent.

```bash
mkdir acme-support && cd acme-support
touch support_agent.py
```

Add the imports and a basic `main` function skeleton:

```python title="support_agent.py"
import asyncio
import os
import cortex


async def main():
    pass  # We will fill this in step by step


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Step 2: Define the order lookup tool

The order lookup tool simulates querying a database of customer orders. In production, this would call your real order management API.

```python title="support_agent.py"
@cortex.tool(name="lookup_order", description="Look up an order by its ID")
async def lookup_order(order_id: str) -> str:
    """
    Retrieve the current status of a customer order.

    Args:
        order_id: The order identifier, e.g. ORD-1001
    """
    orders = {
        "ORD-1001": "Status: Shipped | ETA: Thursday | Carrier: FedEx | Tracking: FX12345",
        "ORD-1002": "Status: Processing | ETA: Next Monday | Payment: Confirmed",
        "ORD-1003": "Status: Delivered | Delivered: Yesterday | Signed by: Front Desk",
        "ORD-1004": "Status: Cancelled | Reason: Customer request | Refund: Pending",
    }
    return orders.get(order_id, f"Order {order_id} not found in our system.")
```

---

## Step 3: Define the FAQ search tool

The FAQ tool searches a small knowledge base. In production, this could query a vector database or search index.

```python title="support_agent.py"
@cortex.tool(name="search_faq", description="Search the FAQ knowledge base")
async def search_faq(query: str) -> str:
    """
    Search the Acme Store FAQ for answers to common questions.

    Args:
        query: The customer's question or keywords
    """
    faqs = {
        "return": "Return policy: Items can be returned within 30 days of delivery. "
                  "Start a return at acme.com/returns or contact support.",
        "shipping": "Standard shipping: 5-7 business days. Express: 2-3 business days. "
                    "Free shipping on orders over $50.",
        "payment": "We accept Visa, Mastercard, AMEX, and PayPal. "
                   "Payment is charged when the order ships.",
        "cancel": "Orders can be cancelled within 1 hour of placement. "
                  "After that, you must wait for delivery and initiate a return.",
    }
    query_lower = query.lower()
    for keyword, answer in faqs.items():
        if keyword in query_lower:
            return answer
    return "No FAQ found. Consider escalating to a human agent."
```

---

## Step 4: Define the escalation tool

When the agent cannot resolve an issue, it should escalate to a human support representative.

```python title="support_agent.py"
@cortex.tool(name="escalate_to_human", description="Escalate the conversation to a human agent")
async def escalate_to_human(reason: str, priority: str = "normal") -> str:
    """
    Transfer the conversation to a human support representative.

    Args:
        reason: Why the issue needs human attention
        priority: Urgency level -- normal, high, or urgent
    """
    return (
        f"Escalation created. Priority: {priority}. Reason: {reason}. "
        f"A human agent will respond within "
        f"{'15 minutes' if priority == 'urgent' else '2 hours'}."
    )
```

---

## Step 5: Configure behavioral weights

corteX behavioral weights control the agent's conversational style. For a support agent, you want a formal, moderately verbose, and helpful tone. Weights range from -1.0 to 1.0.

```python title="support_agent.py"
weights = cortex.WeightConfig(
    formality=0.6,       # Lean formal, but not stiff
    verbosity=-0.3,      # Slightly concise -- respect the customer's time
    initiative=0.4,      # Proactively suggest next steps
    autonomy=0.3,        # Ask before taking major actions
    explanation_depth=0.5 # Explain reasoning, especially for policies
)
```

!!! tip "Tuning weights"
    Start with small values (0.2-0.5) and adjust based on testing. Extreme values like 1.0 produce exaggerated behavior. The weight engine applies momentum and homeostatic clamping, so the effective behavior is always smoother than the raw numbers suggest.

---

## Step 6: Configure enterprise safety

Block sensitive topics and enable audit logging. In this example, the agent must never discuss competitor products or internal pricing formulas.

```python title="support_agent.py"
enterprise = cortex.EnterpriseConfig(
    safety_level="strict",
    blocked_topics=["competitor_info", "internal_pricing", "employee_data"],
    audit_log=True,
)
```

!!! warning "Safety levels"
    The `strict` level enables PII detection, content filtering, and prompt injection protection. In production, you should also set `max_autonomy` to prevent the agent from taking irreversible actions without human approval.

---

## Step 7: Create the engine, agent, and session

Now wire everything together inside the `main` function:

```python title="support_agent.py"
async def main():
    # 1. Create the engine
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": os.environ.get("OPENAI_API_KEY", "sk-...")},
        },
    )

    # 2. Create the agent with tools, weights, and safety
    agent = engine.create_agent(
        name="acme_support",
        system_prompt=(
            "You are a customer support agent for Acme Store. "
            "Be empathetic, professional, and concise. "
            "Always greet the customer, identify their issue, and offer clear next steps. "
            "Use the lookup_order tool for order questions, search_faq for policy questions, "
            "and escalate_to_human when you cannot resolve the issue."
        ),
        tools=[lookup_order, search_faq, escalate_to_human],
        goal_tracking=True,
        weight_config=weights,
        enterprise_config=enterprise,
    )

    # 3. Start a session
    session = agent.start_session(user_id="customer_42")
```

---

## Step 8: Run a multi-turn troubleshooting conversation

Add a helper function that sends a message, prints the response, and displays brain metrics:

```python title="support_agent.py"
    async def chat(message: str):
        """Send a message and display the response with diagnostics."""
        print(f"\n{'='*60}")
        print(f"Customer: {message}")
        print(f"{'='*60}")

        response = await session.run(message)

        print(f"\nAgent: {response.content}")
        print(f"\n--- Diagnostics ---")
        print(f"Model used:    {response.metadata.model_used}")
        print(f"Tokens used:   {response.metadata.tokens_used}")
        print(f"Latency:       {response.metadata.latency_ms:.0f}ms")
        print(f"Tools called:  {response.metadata.tools_called}")
        print(f"Goal progress: {response.metadata.goal_progress}")
```

Now simulate a realistic multi-step troubleshooting conversation:

```python title="support_agent.py"
    # Turn 1: Customer describes the problem
    await chat("Hi, I ordered something last week and it hasn't arrived yet.")

    # Turn 2: Agent asks for the order number
    await chat("Sure, my order number is ORD-1001.")

    # Turn 3: Follow-up question about the shipping policy
    await chat("That's helpful, thanks. What's your shipping policy for express orders?")

    # Turn 4: Request that requires escalation
    await chat("Actually, I want to file a formal complaint about the delay.")
```

---

## Step 9: Inspect brain metrics after the conversation

After the conversation, inspect the session's internal state to understand what the brain learned:

```python title="support_agent.py"
    # Inspect the brain state
    print("\n" + "="*60)
    print("SESSION BRAIN METRICS")
    print("="*60)

    # Synaptic weights -- how the agent's style adapted
    print("\nBehavioral Weights:")
    w = session.get_weights()
    for key, value in w.items():
        print(f"  {key}: {value}")

    # Goal tracking -- did the agent stay on track?
    print("\nGoal Progress:")
    progress = session.get_goal_progress()
    print(f"  {progress}")

    # Dual-process stats -- System 1 vs System 2 usage
    print("\nDual-Process Routing:")
    dp = session.get_dual_process_stats()
    print(f"  {dp}")

    # Tool reputation -- which tools earned trust?
    print("\nTool Reputation:")
    rep = session.get_reputation_stats()
    print(f"  {rep}")
```

---

## Step 10: Close the session and review final stats

Always close sessions to trigger weight consolidation:

```python title="support_agent.py"
    # Close the session
    stats = await session.close()
    print(f"\nSession closed. Total turns: {stats['turns']}, "
          f"Total tokens: {stats['total_tokens']}")
```

---

## Step 11: Run the complete agent

Save the file and run it:

```bash
export OPENAI_API_KEY="sk-..."
python support_agent.py
```

Expected output (your exact responses will vary):

```text
============================================================
Customer: Hi, I ordered something last week and it hasn't arrived yet.
============================================================

Agent: Hello! I am sorry to hear your order has not arrived yet. I would be happy
to help you track it down. Could you please provide me with your order number?
It typically starts with "ORD-" followed by four digits.

--- Diagnostics ---
Model used:    gpt-4o
Tokens used:   287
Latency:       1102ms
Tools called:  []
Goal progress: 0.15

============================================================
Customer: Sure, my order number is ORD-1001.
============================================================

Agent: Thank you. I have located your order. Here are the details:

- **Status:** Shipped
- **Estimated arrival:** Thursday
- **Carrier:** FedEx
- **Tracking number:** FX12345

You can track your package directly at fedex.com using that tracking number.
Is there anything else I can help you with?

--- Diagnostics ---
Model used:    gpt-4o
Tokens used:   412
Latency:       1845ms
Tools called:  ['lookup_order']
Goal progress: 0.55
```

!!! info "Goal progress"
    Notice how `goal_progress` increases from 0.15 to 0.55 as the agent identifies and begins resolving the customer's issue. The goal tracker detects that the core question ("where is my order?") is being addressed.

---

## Complete code

Here is the full `support_agent.py` for reference:

```python title="support_agent.py"
import asyncio
import os
import cortex


@cortex.tool(name="lookup_order", description="Look up an order by its ID")
async def lookup_order(order_id: str) -> str:
    """
    Retrieve the current status of a customer order.

    Args:
        order_id: The order identifier, e.g. ORD-1001
    """
    orders = {
        "ORD-1001": "Status: Shipped | ETA: Thursday | Carrier: FedEx | Tracking: FX12345",
        "ORD-1002": "Status: Processing | ETA: Next Monday | Payment: Confirmed",
        "ORD-1003": "Status: Delivered | Delivered: Yesterday | Signed by: Front Desk",
        "ORD-1004": "Status: Cancelled | Reason: Customer request | Refund: Pending",
    }
    return orders.get(order_id, f"Order {order_id} not found in our system.")


@cortex.tool(name="search_faq", description="Search the FAQ knowledge base")
async def search_faq(query: str) -> str:
    """
    Search the Acme Store FAQ for answers to common questions.

    Args:
        query: The customer's question or keywords
    """
    faqs = {
        "return": "Return policy: Items can be returned within 30 days of delivery. "
                  "Start a return at acme.com/returns or contact support.",
        "shipping": "Standard shipping: 5-7 business days. Express: 2-3 business days. "
                    "Free shipping on orders over $50.",
        "payment": "We accept Visa, Mastercard, AMEX, and PayPal. "
                   "Payment is charged when the order ships.",
        "cancel": "Orders can be cancelled within 1 hour of placement. "
                  "After that, you must wait for delivery and initiate a return.",
    }
    query_lower = query.lower()
    for keyword, answer in faqs.items():
        if keyword in query_lower:
            return answer
    return "No FAQ found. Consider escalating to a human agent."


@cortex.tool(name="escalate_to_human", description="Escalate the conversation to a human agent")
async def escalate_to_human(reason: str, priority: str = "normal") -> str:
    """
    Transfer the conversation to a human support representative.

    Args:
        reason: Why the issue needs human attention
        priority: Urgency level -- normal, high, or urgent
    """
    return (
        f"Escalation created. Priority: {priority}. Reason: {reason}. "
        f"A human agent will respond within "
        f"{'15 minutes' if priority == 'urgent' else '2 hours'}."
    )


weights = cortex.WeightConfig(
    formality=0.6,
    verbosity=-0.3,
    initiative=0.4,
    autonomy=0.3,
    explanation_depth=0.5,
)

enterprise = cortex.EnterpriseConfig(
    safety_level="strict",
    blocked_topics=["competitor_info", "internal_pricing", "employee_data"],
    audit_log=True,
)


async def main():
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": os.environ.get("OPENAI_API_KEY", "sk-...")},
        },
    )

    agent = engine.create_agent(
        name="acme_support",
        system_prompt=(
            "You are a customer support agent for Acme Store. "
            "Be empathetic, professional, and concise. "
            "Always greet the customer, identify their issue, and offer clear next steps. "
            "Use the lookup_order tool for order questions, search_faq for policy questions, "
            "and escalate_to_human when you cannot resolve the issue."
        ),
        tools=[lookup_order, search_faq, escalate_to_human],
        goal_tracking=True,
        weight_config=weights,
        enterprise_config=enterprise,
    )

    session = agent.start_session(user_id="customer_42")

    async def chat(message: str):
        print(f"\n{'='*60}")
        print(f"Customer: {message}")
        print(f"{'='*60}")
        response = await session.run(message)
        print(f"\nAgent: {response.content}")
        print(f"\n--- Diagnostics ---")
        print(f"Model used:    {response.metadata.model_used}")
        print(f"Tokens used:   {response.metadata.tokens_used}")
        print(f"Latency:       {response.metadata.latency_ms:.0f}ms")
        print(f"Tools called:  {response.metadata.tools_called}")
        print(f"Goal progress: {response.metadata.goal_progress}")

    await chat("Hi, I ordered something last week and it hasn't arrived yet.")
    await chat("Sure, my order number is ORD-1001.")
    await chat("That's helpful, thanks. What's your shipping policy for express orders?")
    await chat("Actually, I want to file a formal complaint about the delay.")

    print("\n" + "="*60)
    print("SESSION BRAIN METRICS")
    print("="*60)
    print("\nBehavioral Weights:")
    for key, value in session.get_weights().items():
        print(f"  {key}: {value}")
    print("\nGoal Progress:")
    print(f"  {session.get_goal_progress()}")
    print("\nDual-Process Routing:")
    print(f"  {session.get_dual_process_stats()}")
    print("\nTool Reputation:")
    print(f"  {session.get_reputation_stats()}")

    stats = await session.close()
    print(f"\nSession closed. Total turns: {stats['turns']}, "
          f"Total tokens: {stats['total_tokens']}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## What you learned

- How to define tools with `@cortex.tool` and register them with an agent
- How to tune behavioral weights with `cortex.WeightConfig` for a specific conversational tone
- How to enable goal tracking and observe progress across a multi-turn conversation
- How to configure enterprise safety controls with `cortex.EnterpriseConfig`
- How to inspect brain metrics: weights, goal progress, dual-process stats, and tool reputation

## Next steps

- [Build a Code Review Agent](code-review-agent.md) -- learn about context profiles and dual-process routing
- [Set Up Multi-Provider Failover](multi-provider.md) -- add resilience with multiple LLM providers
- [Deploy to Production](production.md) -- take this agent to production with full enterprise controls
