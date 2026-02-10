# Build a Research Agent

In this tutorial you will build a research agent that conducts multi-step investigations across web sources and documents. You will configure the research context profile, use the memory fabric to maintain coherence across a long session, and track progress through a structured research plan with goal tracking.

---

## What you will build

- A research agent with web search and document reading tools
- The `research` context profile for citation-preserving compression
- Memory fabric integration for long sessions that span many turns
- Goal tracking across a structured multi-step research plan

## Prerequisites

- Python 3.11+
- corteX installed: `pip install cortex-engine[openai]`
- An OpenAI API key

---

## Step 1: Set up the project

```bash
mkdir research-agent && cd research-agent
touch researcher.py
```

```python title="researcher.py"
import asyncio
import json
import os
import cortex
```

---

## Step 2: Define the web search tool

This tool simulates searching the web and returning results. In production, you would integrate with a real search API such as SerpAPI, Tavily, or Brave Search.

```python title="researcher.py"
@cortex.tool(name="web_search", description="Search the web for information")
async def web_search(query: str, num_results: int = 3) -> str:
    """
    Search the web and return a list of results with titles, URLs, and snippets.

    Args:
        query: The search query
        num_results: Number of results to return (default 3)
    """
    # Simulated search results for the tutorial
    search_db = {
        "transformer architecture": [
            {"title": "Attention Is All You Need (Vaswani et al., 2017)",
             "url": "https://arxiv.org/abs/1706.03762",
             "snippet": "We propose a new simple network architecture, the Transformer, "
                        "based solely on attention mechanisms, dispensing with recurrence "
                        "and convolutions entirely."},
            {"title": "An Introduction to Transformers (Turner, 2024)",
             "url": "https://arxiv.org/abs/2304.10557",
             "snippet": "A concise introduction covering self-attention, positional "
                        "encoding, and the encoder-decoder structure."},
            {"title": "The Illustrated Transformer (Alammar, 2018)",
             "url": "https://jalammar.github.io/illustrated-transformer/",
             "snippet": "A visual guide to the Transformer model architecture with "
                        "step-by-step illustrations of attention computation."},
        ],
        "self-attention mechanism": [
            {"title": "Self-Attention and Positional Encoding",
             "url": "https://d2l.ai/chapter_attention/self-attention.html",
             "snippet": "Self-attention computes attention weights between all pairs "
                        "of positions in a sequence, enabling global context access."},
            {"title": "Multi-Head Attention Explained",
             "url": "https://lilianweng.github.io/attention/",
             "snippet": "Multi-head attention runs multiple attention functions in "
                        "parallel, each learning different relationship patterns."},
        ],
        "transformer applications NLP": [
            {"title": "BERT: Pre-training of Deep Bidirectional Transformers",
             "url": "https://arxiv.org/abs/1810.04805",
             "snippet": "BERT uses bidirectional transformer pre-training to achieve "
                        "state-of-the-art results on eleven NLP benchmarks."},
            {"title": "GPT-4 Technical Report (OpenAI, 2023)",
             "url": "https://arxiv.org/abs/2303.08774",
             "snippet": "GPT-4 is a large-scale multimodal model that demonstrates "
                        "human-level performance on professional benchmarks."},
        ],
    }

    # Simple keyword matching for the simulation
    for key, results in search_db.items():
        if any(word in query.lower() for word in key.split()):
            formatted = [
                f"[{r['title']}]({r['url']})\n  {r['snippet']}"
                for r in results[:num_results]
            ]
            return f"Found {len(formatted)} results:\n\n" + "\n\n".join(formatted)

    return "No results found. Try different search terms."
```

---

## Step 3: Define the document reader tool

This tool simulates fetching and reading a document from a URL. In production, this would use an HTTP client or document parser.

```python title="researcher.py"
@cortex.tool(name="read_document", description="Read and extract content from a URL")
async def read_document(url: str) -> str:
    """
    Fetch a document from a URL and return its main content.

    Args:
        url: The URL of the document to read
    """
    # Simulated document content
    documents = {
        "https://arxiv.org/abs/1706.03762": (
            "Title: Attention Is All You Need\n"
            "Authors: Vaswani, Shazeer, Parmar, et al.\n"
            "Year: 2017\n\n"
            "Abstract: The dominant sequence transduction models are based on complex "
            "recurrent or convolutional neural networks. We propose the Transformer, "
            "a model architecture eschewing recurrence and relying entirely on an "
            "attention mechanism to draw global dependencies between input and output. "
            "The Transformer achieves 28.4 BLEU on the WMT 2014 English-to-German "
            "translation task, surpassing the best previously reported results by "
            "over 2 BLEU. On the WMT 2014 English-to-French translation task, the "
            "model establishes a new single-model SOTA BLEU score of 41.8."
        ),
        "https://jalammar.github.io/illustrated-transformer/": (
            "Title: The Illustrated Transformer\n"
            "Author: Jay Alammar\n\n"
            "Key Concepts:\n"
            "1. Self-Attention: Each word attends to all other words in the sequence.\n"
            "2. Multi-Head Attention: Multiple attention heads capture different patterns.\n"
            "3. Positional Encoding: Sinusoidal functions encode word positions.\n"
            "4. Feed-Forward Networks: Applied identically to each position.\n"
            "5. Residual Connections: Stabilize deep network training.\n"
            "6. Layer Normalization: Applied after each sub-layer."
        ),
    }

    content = documents.get(url)
    if content:
        return content
    return f"Could not fetch content from {url}. The page may require authentication."
```

---

## Step 4: Configure the research context profile

The `research` context profile preserves citations and source references during compression while aggressively compressing navigation steps and redundant search results.

```python title="researcher.py"
context_config = cortex.ContextManagementConfig(
    profile="research",         # (1)!
    token_budget_ratio=0.85,    # Use 85% of the context window
)
```

1. The `research` profile adjusts compression priorities: citations and source URLs are preserved at L0 (verbatim) even when surrounding text is compressed to L2 (summary). This prevents the agent from losing track of where information came from.

---

## Step 5: Configure weights for a thorough research style

A research agent should be detail-oriented, methodical, and prioritize quality over speed.

```python title="researcher.py"
weights = cortex.WeightConfig(
    autonomy=0.5,            # Balance between independent work and user check-ins
    verbosity=0.4,           # Provide detailed findings
    initiative=0.5,          # Suggest follow-up research directions
    detail_level=0.6,        # Thorough coverage
    explanation_depth=0.7,   # Deep explanations with reasoning
    speed_vs_quality=-0.4,   # Prioritize quality over speed
)
```

---

## Step 6: Build the engine and agent

```python title="researcher.py"
async def main():
    engine = cortex.Engine(
        providers={
            "openai": {"api_key": os.environ.get("OPENAI_API_KEY", "sk-...")},
        },
    )

    agent = engine.create_agent(
        name="researcher",
        system_prompt=(
            "You are a thorough research agent. When given a topic, you: "
            "1. Break it into sub-questions. "
            "2. Search for authoritative sources on each sub-question. "
            "3. Read and synthesize the most relevant documents. "
            "4. Produce a structured summary with citations. "
            "Always cite your sources with [Author, Year] format. "
            "Track your progress and tell the user which steps remain."
        ),
        tools=[web_search, read_document],
        goal_tracking=True,
        weight_config=weights,
        context_config=context_config,
    )

    session = agent.start_session(user_id="researcher_1")
```

---

## Step 7: Start a multi-step research task

Assign the agent a research topic and let it work through the plan across multiple turns:

```python title="researcher.py"
    # Turn 1: Assign the research topic
    print("--- Step 1: Assign Research Topic ---")
    r1 = await session.run(
        "Research the Transformer architecture in deep learning. "
        "Cover: (a) the original architecture, (b) the self-attention mechanism, "
        "and (c) key applications in NLP. Produce a structured report."
    )
    print(r1.content)
    print(f"\nGoal progress: {r1.metadata.goal_progress}")
    print(f"Tools called:  {r1.metadata.tools_called}")
```

Expected output:

```text
--- Step 1: Assign Research Topic ---
I have broken the research into three sub-questions:

1. What is the original Transformer architecture?
2. How does the self-attention mechanism work?
3. What are the key applications of Transformers in NLP?

Let me start by searching for the original Transformer paper...

[Search results and initial findings]

Goal progress: 0.2
Tools called:  ['web_search']
```

---

## Step 8: Continue the research across turns

```python title="researcher.py"
    # Turn 2: Dive deeper into a specific source
    print("\n--- Step 2: Deep Dive ---")
    r2 = await session.run(
        "Read the original 'Attention Is All You Need' paper and "
        "the Illustrated Transformer guide. Summarize the key architectural components."
    )
    print(r2.content)
    print(f"\nGoal progress: {r2.metadata.goal_progress}")

    # Turn 3: Move to the next sub-question
    print("\n--- Step 3: Applications ---")
    r3 = await session.run(
        "Now research key NLP applications of Transformers. "
        "Focus on BERT and GPT as the two major paradigms."
    )
    print(r3.content)
    print(f"\nGoal progress: {r3.metadata.goal_progress}")

    # Turn 4: Request the final synthesis
    print("\n--- Step 4: Final Report ---")
    r4 = await session.run(
        "Synthesize everything into a final structured report with sections "
        "for Architecture, Self-Attention, Applications, and Conclusion. "
        "Include all citations."
    )
    print(r4.content)
    print(f"\nGoal progress: {r4.metadata.goal_progress}")
```

!!! info "Memory fabric at work"
    By Turn 4, the session has accumulated a significant amount of context from multiple searches and document reads. The memory fabric automatically manages this: working memory holds the current task state, episodic memory records each research step and its outcome, and the context engine compresses older search results while preserving citations. This is why the agent can produce a coherent final report that references findings from Turn 1 even though those results have been compressed.

---

## Step 9: Inspect research session metrics

```python title="researcher.py"
    # Inspect session state
    print("\n" + "="*60)
    print("RESEARCH SESSION METRICS")
    print("="*60)

    print(f"\nGoal progress: {session.get_goal_progress()}")
    print(f"Dual-process:  {session.get_dual_process_stats()}")
    print(f"Tool reputation: {session.get_reputation_stats()}")

    print("\nBehavioral Weights (post-session):")
    for key, value in session.get_weights().items():
        print(f"  {key}: {value}")
```

!!! tip "Monitoring goal progress"
    A well-structured research task should show steady goal progress: approximately 0.2 after breaking down the topic, 0.5 after the deep dive, 0.75 after covering all sub-questions, and approaching 1.0 after the final synthesis. If progress stalls or drifts, the goal tracker will trigger System 2 replanning.

---

## Step 10: Close the session

```python title="researcher.py"
    stats = await session.close()
    print(f"\nSession closed. Turns: {stats['turns']}, Tokens: {stats['total_tokens']}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## Complete code

```python title="researcher.py"
import asyncio
import json
import os
import cortex


@cortex.tool(name="web_search", description="Search the web for information")
async def web_search(query: str, num_results: int = 3) -> str:
    """Search the web and return results with titles, URLs, and snippets."""
    search_db = {
        "transformer architecture": [
            {"title": "Attention Is All You Need (Vaswani et al., 2017)",
             "url": "https://arxiv.org/abs/1706.03762",
             "snippet": "The Transformer: a model based solely on attention mechanisms."},
            {"title": "The Illustrated Transformer (Alammar, 2018)",
             "url": "https://jalammar.github.io/illustrated-transformer/",
             "snippet": "A visual guide to the Transformer model architecture."},
        ],
        "self-attention mechanism": [
            {"title": "Self-Attention and Positional Encoding",
             "url": "https://d2l.ai/chapter_attention/self-attention.html",
             "snippet": "Self-attention computes weights between all pairs of positions."},
        ],
        "transformer applications NLP": [
            {"title": "BERT: Pre-training of Bidirectional Transformers",
             "url": "https://arxiv.org/abs/1810.04805",
             "snippet": "BERT achieves SOTA on eleven NLP benchmarks."},
            {"title": "GPT-4 Technical Report (OpenAI, 2023)",
             "url": "https://arxiv.org/abs/2303.08774",
             "snippet": "GPT-4 demonstrates human-level performance on benchmarks."},
        ],
    }
    for key, results in search_db.items():
        if any(word in query.lower() for word in key.split()):
            formatted = [f"[{r['title']}]({r['url']})\n  {r['snippet']}"
                         for r in results[:num_results]]
            return f"Found {len(formatted)} results:\n\n" + "\n\n".join(formatted)
    return "No results found."


@cortex.tool(name="read_document", description="Read and extract content from a URL")
async def read_document(url: str) -> str:
    """Fetch a document from a URL and return its main content."""
    documents = {
        "https://arxiv.org/abs/1706.03762": (
            "Title: Attention Is All You Need\n"
            "Authors: Vaswani, Shazeer, Parmar, et al. (2017)\n\n"
            "The Transformer uses self-attention to draw global dependencies. "
            "Achieves 28.4 BLEU on WMT 2014 EN-DE, 41.8 BLEU on EN-FR."
        ),
        "https://jalammar.github.io/illustrated-transformer/": (
            "Title: The Illustrated Transformer\n"
            "Key: Self-attention, multi-head attention, positional encoding, "
            "feed-forward networks, residual connections, layer normalization."
        ),
    }
    return documents.get(url, f"Could not fetch {url}.")


context_config = cortex.ContextManagementConfig(profile="research", token_budget_ratio=0.85)
weights = cortex.WeightConfig(
    autonomy=0.5, verbosity=0.4, initiative=0.5,
    detail_level=0.6, explanation_depth=0.7, speed_vs_quality=-0.4,
)


async def main():
    engine = cortex.Engine(
        providers={"openai": {"api_key": os.environ.get("OPENAI_API_KEY", "sk-...")}},
    )
    agent = engine.create_agent(
        name="researcher",
        system_prompt=(
            "You are a thorough research agent. Break topics into sub-questions, "
            "search for authoritative sources, read key documents, and produce "
            "structured summaries with [Author, Year] citations."
        ),
        tools=[web_search, read_document],
        goal_tracking=True,
        weight_config=weights,
        context_config=context_config,
    )
    session = agent.start_session(user_id="researcher_1")

    r1 = await session.run(
        "Research the Transformer architecture in deep learning. "
        "Cover: (a) original architecture, (b) self-attention, (c) NLP applications."
    )
    print(r1.content)
    print(f"Goal progress: {r1.metadata.goal_progress}")

    r2 = await session.run(
        "Read the original paper and the Illustrated Transformer guide. "
        "Summarize key architectural components."
    )
    print(r2.content)
    print(f"Goal progress: {r2.metadata.goal_progress}")

    r3 = await session.run(
        "Research key NLP applications: BERT and GPT paradigms."
    )
    print(r3.content)
    print(f"Goal progress: {r3.metadata.goal_progress}")

    r4 = await session.run(
        "Synthesize into a final report: Architecture, Self-Attention, "
        "Applications, Conclusion. Include all citations."
    )
    print(r4.content)
    print(f"Goal progress: {r4.metadata.goal_progress}")

    print(f"\nGoal progress: {session.get_goal_progress()}")
    print(f"Dual-process:  {session.get_dual_process_stats()}")
    print(f"Tool reputation: {session.get_reputation_stats()}")

    stats = await session.close()
    print(f"\nSession closed. Turns: {stats['turns']}, Tokens: {stats['total_tokens']}")


if __name__ == "__main__":
    asyncio.run(main())
```

---

## What you learned

- How to configure the `research` context profile for citation-preserving compression
- How the memory fabric maintains coherence across long multi-turn sessions
- How goal tracking monitors progress through a structured research plan
- How to build tools that simulate external data sources for development and testing

## Next steps

- [Set Up Multi-Provider Failover](multi-provider.md) -- add resilience with multiple LLM providers
- [Deploy to Production](production.md) -- enterprise configuration, logging, and monitoring
- [Context & Memory concepts](../concepts/context/index.md) -- deep dive into how the memory fabric works
