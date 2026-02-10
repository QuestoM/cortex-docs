# Context Management for Long-Running AI Agents: Comprehensive Research & Architecture Proposal

**Date:** 2026-02-09
**Author:** corteX Research Team
**Scope:** Competitive analysis of context management across major AI frameworks, gap analysis, and recommended architecture for corteX's 10,000+ step workflows.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Competitor Analysis](#2-competitor-analysis)
   - 2.1 Anthropic / Claude Code
   - 2.2 OpenAI Agents SDK
   - 2.3 LangChain / LangGraph
   - 2.4 Google Gemini / ADK
   - 2.5 Letta (MemGPT) & Mem0
   - 2.6 Microsoft Agent Framework (AutoGen + Semantic Kernel)
3. [Academic Research & Key Findings](#3-academic-research--key-findings)
4. [Strengths and Weaknesses Matrix](#4-strengths-and-weaknesses-matrix)
5. [Gap Analysis: What No Competitor Does Well](#5-gap-analysis-what-no-competitor-does-well)
6. [Recommended Architecture for corteX](#6-recommended-architecture-for-cortex)
7. [Built-In vs. SDK-Configurable Boundary](#7-built-in-vs-sdk-configurable-boundary)
8. [Mathematical Models](#8-mathematical-models)
9. [Implementation Roadmap](#9-implementation-roadmap)
10. [References](#10-references)

---

## 1. Executive Summary

corteX is designed for enterprise tasks that run for hours with thousands of agent steps: codebase-wide refactors across 500+ files, multi-day research investigations, and 24/7 operational monitoring. Every existing AI agent framework was designed primarily for short sessions (10-50 turns). Their context management solutions are afterthoughts, not architectural foundations.

This document analyzes how six major frameworks handle context management, identifies critical gaps none of them address, and proposes a purpose-built context management architecture that will be corteX's primary competitive advantage.

**Key finding:** No existing framework supports graceful degradation across 10,000+ steps. The best current solution (Anthropic's server-side compaction) handles indefinite sessions but loses semantic fidelity at scale. corteX's architecture must go far beyond simple summarization to implement hierarchical, importance-weighted, temporally-aware context management that mirrors human memory consolidation.

---

## 2. Competitor Analysis

### 2.1 Anthropic / Claude Code

Anthropic has the most sophisticated production context management system as of early 2026, combining three complementary strategies.

#### 2.1.1 Server-Side Compaction (Beta: `compact-2026-01-12`)

**How it works:**
- When input tokens exceed a configurable trigger threshold (default: 150,000 tokens; minimum: 50,000), Claude automatically generates a summary of the entire conversation.
- The summary is returned as a `compaction` block within the assistant response.
- On subsequent requests, the API ignores all message blocks preceding the most recent `compaction` block, effectively replacing the full history with the summary.
- Multiple compactions can occur within a single long-running request (e.g., during server tool loops).

**Key parameters:**
| Parameter | Default | Description |
|-----------|---------|-------------|
| `trigger.value` | 150,000 tokens | Token threshold to trigger compaction |
| `pause_after_compaction` | false | Pause to allow injecting preserved messages |
| `instructions` | (default prompt) | Custom summarization prompt (full replacement) |

**Default summarization prompt:**
> "You have written a partial transcript for the initial task above. Please write a summary of the transcript. The purpose of this summary is to provide continuity so you can continue to make progress towards solving the task in a future context, where the raw history above may not be accessible and will be replaced with this summary. Write down anything that would be helpful, including the state, next steps, learnings etc."

**Token budget enforcement pattern:**
```python
TRIGGER_THRESHOLD = 100_000
TOTAL_TOKEN_BUDGET = 3_000_000
n_compactions = 0
# After each compaction:
if n_compactions * TRIGGER_THRESHOLD >= TOTAL_TOKEN_BUDGET:
    # Force wrap-up
```

**Cost implications:** Compaction requires an additional sampling step using the same model. The `usage.iterations` array separates compaction usage from message usage for accurate billing. No option to use a cheaper model for summarization.

#### 2.1.2 Context Editing (Beta: `context-management-2025-06-27`)

**Tool result clearing (`clear_tool_uses_20250919`):**
- Automatically clears the oldest tool results in chronological order when approaching token limits.
- Replaced results get placeholder text so Claude knows information was removed.
- Optional: clear both tool inputs and outputs via `clear_tool_inputs: true`.
- In a 100-turn web search evaluation, context editing reduced token consumption by 84%.

**Thinking block clearing (`clear_thinking_20251015`):**
- Previous thinking blocks are already stripped from context window calculation by default.
- This strategy explicitly removes them from the message history as well.

**Combined performance:** Context editing alone delivers 29% improvement on complex multi-step tasks. Combined with the memory tool, performance improves 39% over baseline.

#### 2.1.3 Memory Tool (File-Based External Memory)

- Claude can create, read, update, and delete files in a dedicated memory directory.
- Persists across conversations and sessions.
- Stored on the developer's infrastructure, not Anthropic's servers.
- Enables agents to build knowledge bases over time without keeping everything in context.

#### 2.1.4 Claude Code Auto-Compact Behavior

- Triggers automatically at approximately 95% context capacity (~25% remaining, recently adjusted to ~33K token buffer / 16.5%).
- Since v2.0.64, compaction is instant (no user-visible delay).
- Known issue: Auto-compact can disrupt workflow when it triggers at an inopportune moment. Manual `/compact` at strategic points is recommended by power users.
- Claude Code maintains a CLAUDE.md file as persistent memory across sessions.

#### 2.1.5 Limitations

- **Same model for summarization:** No option to use a cheaper/faster model for the compaction step. This is expensive at scale.
- **Flat summarization:** The compaction prompt produces a single summary. There is no progressive summarization, no importance hierarchy, and no selective detail preservation.
- **No semantic graph:** Compaction treats the conversation as a linear text stream. It does not extract entities, relationships, or structured state.
- **Context rot still applies:** Even with compaction, the summarized context occupies attention budget and degrades performance as it grows.

---

### 2.2 OpenAI Agents SDK

OpenAI's approach splits context management into two explicit strategies via their Session abstraction.

#### 2.2.1 Context Trimming (`TrimmingSession`)

**Mechanism:**
- Maintains a `deque` of chronological items.
- A "turn" = one user message + all subsequent items (assistant replies, tool calls/results) until the next user message.
- `_trim_to_last_turns()` scans backward, identifies the last N user messages, and discards everything before the earliest.
- Thread-safe via `asyncio.Lock`.

**Characteristics:**
- Zero latency overhead (no LLM calls).
- Deterministic, debuggable behavior.
- Abrupt context loss at the boundary -- no gradual degradation.
- Cannot retain long-range constraints, decisions, or rationales.

**Best for:** Short, independent, sequential tasks where only recent context matters.

#### 2.2.2 Context Summarization (`SummarizingSession`)

**Mechanism:**
- When real user turns exceed `context_limit`, everything before the last `keep_last_n_turns` turns is summarized into a synthetic `user -> assistant` message pair.
- `LLMSummarizer` uses a structured prompt with sections: Product & Environment, Reported Issue, Steps Tried & Results, etc.
- Tracks "real" vs. "synthetic" messages via metadata flags.
- Post-summarization: injects synthetic user ("Summarize conversation...") + synthetic assistant (generated summary) + preserved recent turns.
- Revalidates conditions after async summarization to prevent race conditions.

**Summary prompt principles:**
- Contradiction checking and temporal ordering.
- Marks unverified facts rather than inferring.
- Trims verbose tool outputs to `tool_trim_limit` (default: 600 chars).
- Incorporates tool performance insights and next-step guidance.
- Maximum summary length: `max_tokens` (default: 400).

**Comparison:**
| Dimension | Trimming | Summarization |
|-----------|----------|---------------|
| Latency/Cost | Minimal | Higher at refresh points |
| Long-range Recall | Weak (hard cutoff) | Strong (compressed carry-forward) |
| Risk Type | Context loss | Context distortion/poisoning |
| Observability | Simple logs | Must audit summary prompts/outputs |
| Eval Stability | High (deterministic) | Requires robust eval |

#### 2.2.3 Handoff Pattern

- When an agent hands off to another, the full conversation history transfers by default.
- `RunConfig.nest_handoff_history` collapses the prior transcript into a single assistant summary in a `<CONVERSATION HISTORY>` block.
- `input_filter` allows filtering what context reaches the next agent.
- Each handoff appends new turns to the same run context.

**Limitation:** No built-in mechanism for progressive compression across a chain of many handoffs. The `<CONVERSATION HISTORY>` block grows linearly.

#### 2.2.4 Conversations API (Server-Side State)

- Durable threads with replayable state.
- Server-managed conversation persistence.
- Details on internal context management strategy are limited in public documentation.

#### 2.2.5 Limitations

- **No built-in token counting integration:** Token budget is set implicitly by `max_turns`, not by actual token counts.
- **Binary choice:** Developers must choose trimming OR summarization upfront; no hybrid mode.
- **Context poisoning risk:** Summarization can introduce subtle distortions that compound over many cycles.
- **No hierarchical memory:** All context is treated as a flat conversation history.

---

### 2.3 LangChain / LangGraph

LangChain provides the widest variety of memory abstractions but suffers from fragmentation and architectural debt.

#### 2.3.1 Memory Types

**ConversationBufferMemory:**
- Stores the complete conversation verbatim.
- Simple but will exceed context window rapidly.
- No compression, no eviction.

**ConversationBufferWindowMemory:**
- Keeps the last `k` interactions.
- Same as OpenAI's trimming: abrupt cutoff.
- No importance weighting.

**ConversationSummaryMemory:**
- Summarizes the full conversation using an LLM after each interaction.
- Quality depends entirely on the summarization LLM.
- Cumulative error: each summary summarizes the previous summary plus the new turn.
- No selective preservation of important details.

**ConversationSummaryBufferMemory:**
- Hybrid: keeps recent messages verbatim + summary of older messages.
- Uses token count (not message count) to determine the flush threshold via `max_token_limit`.
- Currently the most sophisticated LangChain memory type.
- `trim_messages` from `langchain_core.messages` provides precise token-count-based trimming.

**ConversationTokenBufferMemory:**
- Uses `max_token_limit` to keep only messages fitting within a token budget.
- FIFO eviction of oldest messages.

#### 2.3.2 LangGraph State & Persistence

**Checkpointing:**
- Every "super-step" of a LangGraph workflow is checkpointed.
- Checkpoints stored in threads (identified by `thread_id`).
- Storage backends: SQLite (dev), PostgreSQL (production), Cosmos DB (Azure).
- Enables: human-in-the-loop, time travel (rewind to any checkpoint), fault tolerance.

**State management:**
- Graph state is a typed schema (TypedDict or Pydantic model).
- State is the canonical source of truth, not the conversation history.
- Reducers control how state updates merge (replace, append, custom).

**Message handling for long conversations (LangGraph):**
- Message filtering: select specific message types or roles.
- Message trimming: `trim_messages` limits by token count.
- LLM-based summarization: periodically summarize and replace old messages in state.
- These are manual patterns, not automatic -- developers must implement them.

#### 2.3.3 Limitations

- **Fragmented API:** Six different memory classes with different interfaces, all marked as legacy.
- **No automatic management:** Developers must manually implement summarization triggers, token counting, and eviction logic.
- **Cumulative summarization error:** `ConversationSummaryMemory` compounds errors with each summarization cycle.
- **No importance weighting:** All messages are treated equally; a critical decision and a routine acknowledgment receive the same treatment.
- **Weak token counting:** Uses approximate methods; no native integration with model-specific tokenizers.
- **LangGraph checkpointing is for state, not context:** Checkpoints persist graph state but don't solve the LLM context window problem.

---

### 2.4 Google Gemini / ADK

Google takes advantage of Gemini's massive context windows (up to 2M tokens with Gemini 2.5 Pro) and supplements with two novel approaches.

#### 2.4.1 Interactions API (Stateful Server-Side Conversations)

**Mechanism:**
- Server-side conversation state management using `previous_interaction_id`.
- Instead of sending the full conversation history, the client sends only the new turn plus the interaction ID, and the server reconstructs context.
- The server stores and manages the full history.
- Optional: developers can operate in stateless mode by sending full history each time.

**Retention:**
- Paid tier: 55 days.
- Free tier: 1 day.

**ADK integration:** `use_interactions_api=True` in the Gemini model configuration.

**Advantage:** Eliminates client-side context management overhead. The server handles history efficiently.

**Limitation:** Opaque -- developers cannot control what the server does internally with the context. No customizable compression or importance weighting.

#### 2.4.2 ADK Session, State, and Memory

**Session:** A single interaction thread, managed by `SessionService`. Contains event history (chronological messages/actions) and state.

**State:** Key-value store for the current conversation thread. Used for conversational memory (e.g., shopping cart items, user preferences). Scoped to a single session.

**Memory:** Cross-session knowledge store managed by `MemoryService`. Ingests information from completed sessions. Provides search capabilities for recalling information across sessions.

**Implementation note:** In-memory implementations are volatile. Production requires database-backed services.

#### 2.4.3 Thought Signatures (Gemini 3)

**What they are:** Encrypted representations of the model's internal reasoning state.

**Purpose:** Preserve reasoning continuity across multi-turn interactions, especially during function calling. The model can resume its chain of thought from the exact state where it paused, rather than re-computing from raw message history.

**Requirements:**
- Must be passed back during function calling in Gemini 3 models (validation error if omitted).
- For text/chat: not strictly enforced, but omitting degrades reasoning quality.
- Official SDKs handle this automatically when using the chat feature.

**Significance for corteX:** This is the only production system that preserves internal reasoning state across turns, not just conversation text. This is a fundamentally different approach to continuity.

#### 2.4.4 Limitations

- **Reliance on massive context windows:** Google's strategy partially depends on "just make the window bigger." As Chroma's context rot research shows, this does not scale linearly in quality.
- **Opaque server-side management:** Developers cannot customize how the Interactions API manages long histories internally.
- **Thought signatures are model-specific:** Only works with Gemini models; not a general solution.
- **ADK memory is basic:** Cross-session memory is a search index, not a structured knowledge graph.

---

### 2.5 Letta (MemGPT) & Mem0

These two projects represent the most research-driven approaches to memory management for LLM agents.

#### 2.5.1 MemGPT / Letta: OS-Inspired Virtual Context Management

**Core concept:** Treat the LLM's context window like virtual memory in an operating system. Data moves between fast (in-context) and slow (out-of-context) storage.

**Two-tier architecture:**
1. **In-context memory (main context):** The active context window. Segmented into:
   - **Persona block:** Agent's personality and instructions (self-editable).
   - **User information block:** Facts about the current user (dynamically updatable).
   - **Conversation buffer:** Recent messages.
2. **Out-of-context memory (external context):**
   - **Archival memory:** Vector database (Chroma, pgvector). Stores long-term memories that don't fit in context.
   - **Recall memory:** Full conversation history log. Searchable by date and text.

**Data movement mechanism:**
- The LLM itself decides what to move in and out of context using designated memory-editing tools:
  - `core_memory_append` / `core_memory_replace`: Edit in-context memory blocks.
  - `archival_memory_insert` / `archival_memory_search`: Move data to/from archival storage.
  - `conversation_search` / `conversation_search_date`: Query recall memory.
- **Heartbeat mechanism:** The agent can request additional processing steps by setting `request_heartbeat=true`, enabling multi-step memory operations within a single turn.

**Key innovation:** The agent is self-aware of its memory limitations and actively manages what's in context. This is fundamentally different from external systems that manage context on behalf of the agent.

**Limitations:**
- **LLM-dependent quality:** Memory management quality depends on the LLM's ability to effectively use memory tools. Weaker models make poor memory management decisions.
- **Overhead:** Every memory operation requires a tool call, consuming tokens and adding latency.
- **No automatic compression:** The system does not automatically summarize or compress; it relies on the LLM to decide when and how to condense information.
- **Single-tier compression:** Archival memory is a flat vector store, not a hierarchical system.

#### 2.5.2 Mem0: Graph-Enhanced Memory with Consolidation

**Architecture (three-phase pipeline):**

1. **Extraction:** Processes the most recent M messages to extract potential new memories. Uses the LLM as a memory extractor.

2. **Update (Consolidation):** Evaluates extracted memories against the S most similar historical memories from the database. Decides: add, update, delete, or no-op. Uses a tool-call mechanism for structured decisions.

3. **Retrieval:** Semantic search over stored memories. Returns memories exceeding a configurable relevance threshold, ranked by similarity.

**Mem0^g (Graph Memory):**
- Stores memories as a directed, labeled graph.
- Entity extractor identifies nodes; relations generator infers labeled edges.
- Examples: `user --lives_in--> city`, `user --owns--> product`.
- Uses Neo4j as the graph database.
- Conflict detector flags contradictions; LLM-powered update resolver decides merges/invalidations.
- Semantic triplet retrieval: encode query as dense embedding, match against graph triplet encodings.

**Performance results:**
- 26% relative improvement over OpenAI's memory system on LLM-as-a-Judge metric.
- 91% lower p95 latency vs. baseline.
- 90%+ token cost savings.
- Graph memory (Mem0^g) adds ~2% improvement over base Mem0.

**Limitations:**
- **Extraction quality:** Depends on LLM's ability to identify salient information.
- **Graph construction overhead:** Entity extraction and relation generation add latency.
- **Not designed for multi-thousand-step workflows:** Optimized for conversational memory, not agentic task execution with tool-heavy workflows.

---

### 2.6 Microsoft Agent Framework (AutoGen + Semantic Kernel)

#### 2.6.1 Architecture Overview

Microsoft Agent Framework (MAF) merges AutoGen's multi-agent abstractions with Semantic Kernel's enterprise features. Key characteristics:
- Event-driven, distributed architecture (built on Microsoft Orleans).
- Pluggable memory providers (Redis, Pinecone, etc.).
- Session-based state management with YAML/JSON declarative definitions.
- OpenTelemetry for observability.
- Long-running durability for stateful tasks.
- Human-in-the-loop approval flows.

#### 2.6.2 Memory Approach

- Short-term memory: conversation context within an agent session.
- Long-term memory: external stores via pluggable connectors.
- Memory is treated as a plugin, not a core architectural component.
- No built-in compression, summarization, or progressive context management.

#### 2.6.3 Limitations

- **Memory as an afterthought:** The framework focuses on orchestration and multi-agent coordination, not context management.
- **No built-in token budget management:** Developers must implement their own token counting and context trimming.
- **Plugin fragmentation:** Different memory providers have different capabilities and APIs.
- **No automatic consolidation:** No mechanism to promote short-term memories to long-term storage automatically.

---

## 3. Academic Research & Key Findings

### 3.1 Context Rot (Chroma Research, 2025)

**Definition:** Systematic degradation of LLM performance as input context length increases, even on trivially simple tasks.

**Key findings from evaluation of 18 SOTA models:**
- Performance degradation is gradual, not cliff-like, across token counts from 25 to 10,000+ tokens.
- Lower semantic similarity between query and target information amplifies degradation with increasing context.
- Single distractors reduce accuracy; multiple distractors compound the effect.
- Models show better accuracy when target information appears early in context (primacy bias).
- **Structural sensitivity:** Models perform worse on logically coherent haystacks than shuffled ones, suggesting attention allocation is influenced by document structure.
- **Model-specific behaviors:**
  - Claude models: Conservative under ambiguity, tend to abstain.
  - GPT models: Highest hallucination rates, generate confident incorrect answers.
  - Gemini 2.5 Pro: Greatest variability in spurious content generation.

**Implication for corteX:** Simply filling a large context window is counterproductive. Context must be curated, not accumulated. Every token must earn its place.

### 3.2 The Complexity Trap (JetBrains Research, NeurIPS 2025)

**Title:** "Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management"

**Experimental setup:** 500 instances from SWE-bench Verified.

**Key findings:**
- **Observation masking** (replacing older tool outputs with placeholders while preserving reasoning/action history) achieves over 50% cost reduction vs. unmanaged context.
- **LLM summarization** also achieves over 50% cost reduction but causes **trajectory elongation**: agents run 13-15% longer. With Gemini 2.5 Flash, trajectories reached 52 turns; with Qwen3-Coder, ~15% longer runs.
- Summarization consumed over 7% of total per-instance cost for the summarization step itself.
- Summaries may obscure stopping signals, causing agents to iterate unnecessarily.
- **Qwen3-Coder with masking:** 2.6% higher solve rate while being 52% cheaper than raw context.
- **Hybrid approach** (masking as primary layer + selective summarization for specific segments) achieved the best overall efficiency.

**Implication for corteX:** Observation masking should be the first-line strategy. LLM summarization should be used selectively and sparingly, not as the default compression mechanism.

### 3.3 Chain-of-Agents (Google, NeurIPS 2024)

Multi-agent collaboration framework for long-context tasks. Agents collaborate through natural language to aggregate information and reason across context that no single agent could process. Relevant as a distributed context processing strategy.

### 3.4 Memory in the Age of AI Agents (Survey, 2025)

Comprehensive survey cataloging memory approaches across the agent ecosystem. Key taxonomy:
- **Working memory:** In-context, limited capacity.
- **Episodic memory:** Experience-based, supports learning from past attempts.
- **Semantic memory:** Factual knowledge, domain information.
- **Procedural memory:** Learned skills and procedures.

This taxonomy aligns closely with corteX's existing memory architecture.

---

## 4. Strengths and Weaknesses Matrix

| Capability | Anthropic | OpenAI SDK | LangChain/Graph | Google ADK | Letta/MemGPT | Mem0 | **corteX (needed)** |
|------------|-----------|------------|------------------|------------|--------------|------|---------------------|
| Auto-compaction | Server-side, configurable | Manual trim/summarize | Manual, fragmented | Server-side (opaque) | Agent-directed | N/A | Intelligent, multi-strategy |
| Progressive summarization | None (flat summary) | None | SummaryBuffer (basic) | None | Agent-directed | Consolidation | Multi-resolution temporal |
| Token budget management | Compaction counter pattern | Implicit via max_turns | max_token_limit | Massive window strategy | N/A | 90% savings | Precise, per-step budgeting |
| Importance weighting | None | None | None | None | Via agent judgment | Via similarity scoring | Mathematical model |
| Structured state extraction | None | None | Graph state (manual) | Key-value state | Core memory blocks | Graph memory | Automatic entity/relation extraction |
| Cross-session persistence | Memory tool (files) | Conversations API | Checkpointing (state) | SessionService + MemoryService | Archival + recall memory | Graph DB | MemoryFabric (existing) |
| Tool result management | Clear stale results (84% savings) | None built-in | None | None | Via archival memory | N/A | Tiered tool result caching |
| Reasoning continuity | None | None | None | Thought signatures | Heartbeat mechanism | None | Reasoning chain preservation |
| 10,000+ step support | Degrades (compaction loops) | Fails (context overflow) | Fails (manual effort) | Possible (huge window) | Possible but expensive | Not designed for this | **Core design target** |
| Observation masking | Context editing (tool clearing) | None | None | None | None | None | Hybrid masking + selective summarization |

---

## 5. Gap Analysis: What No Competitor Does Well

### Gap 1: Multi-Resolution Temporal Context

No framework implements progressive summarization where detail resolution varies by recency:
- Last 10 steps: full verbatim detail.
- Steps 11-50: key actions and results, tool outputs condensed.
- Steps 51-200: summarized into decision points and outcomes.
- Steps 200+: compressed to goals achieved and lessons learned.

Every existing system uses a binary approach: either full detail or single flat summary.

### Gap 2: Importance-Weighted Context Packing

No framework uses a mathematical model to score which context items should occupy the limited window. Current approaches use:
- Recency only (trimming/windowing).
- Everything-or-nothing (summarize all or keep all).
- Agent judgment (Letta) -- inconsistent and expensive.

What's needed: a scoring function that considers recency, relevance to current goal, causal importance (did this decision affect downstream steps?), and reference frequency.

### Gap 3: Structured State Extraction from Conversation

No framework automatically extracts structured state (entities, relationships, decisions, open questions) from conversation history into a queryable format. Mem0's graph memory comes closest but is designed for user preferences, not complex task state.

### Gap 4: Token Budget Optimization Across Multi-Step Workflows

No framework optimizes total token spend across thousands of steps. Anthropic's compaction counter pattern is the closest, but it's a simple threshold check, not an optimization algorithm that trades off information retention vs. cost.

### Gap 5: Fault-Tolerant Context Recovery

If context gets corrupted (bad summarization, hallucinated state), no framework provides recovery mechanisms. LangGraph's checkpointing enables time travel for state, but not for LLM context.

### Gap 6: Domain-Aware Compression

All frameworks use generic summarization. None understand domain-specific importance signals:
- In a code refactoring task: file paths, function signatures, and test results are critical; verbose error messages are not.
- In a research task: key findings and source citations are critical; search query history is not.
- In an operations task: current system state and anomaly patterns are critical; routine health checks are not.

### Gap 7: Concurrent Context Management

No framework handles context management for parallel sub-tasks within a single workflow. When multiple tool calls execute concurrently, their results must be intelligently merged into the context without exceeding budgets.

---

## 6. Recommended Architecture for corteX

### 6.1 Overview: The Cortical Context Engine (CCE)

The Cortical Context Engine is a purpose-built context management system for long-running workflows. It sits between the corteX orchestrator and the LLM provider, managing what information occupies the context window at each step.

```
                                    +-------------------+
                                    |  Semantic Memory  |
                                    |    (Neocortex)    |
                                    +--------+----------+
                                             |
+----------+    +-----+    +---------+    +--+--+    +---------+
| Orchestr.|<-->| CCE |<-->| Context |<-->| LLM |<-->| Response|
|   Loop   |    |     |    |  Window |    |     |    | Handler |
+----------+    +--+--+    +---------+    +-----+    +---------+
                   |
          +--------+--------+
          |        |        |
     +----+---+ +--+---+ +-+------+
     |Working | |Episo.| |Context |
     |Memory  | |Memory| |Archive |
     |(Hot)   | |(Warm)| |(Cold)  |
     +--------+ +------+ +--------+
```

### 6.2 Three-Temperature Memory Hierarchy

Inspired by CPU cache hierarchies and MemGPT's virtual memory concept, but with automatic management (not agent-directed).

#### Hot Memory (Working Context)
- **What:** Current step's immediate needs. System prompt, current goal, active tool results, recent 5-10 turns verbatim.
- **Size budget:** 40% of context window.
- **Management:** Always present, refreshed every step.
- **Eviction:** Automatic via importance scoring.

#### Warm Memory (Session Context)
- **What:** Compressed recent history. Summarized older turns, key decisions, active state variables, relevant episodic memories.
- **Size budget:** 35% of context window.
- **Management:** Progressive summarization. Detail degrades with age using configurable time constants.
- **Eviction:** Lowest-importance items evicted first; evicted items move to Cold.

#### Cold Memory (Archive Context)
- **What:** Full history in external storage. Searchable by semantic query, temporal range, and entity reference.
- **Size budget:** Unlimited (external storage).
- **Management:** Indexed, queryable, retrievable on demand.
- **Promotion:** Items can be promoted back to Warm or Hot when relevance increases.

### 6.3 The Context Window Packer

At each LLM call, the Context Window Packer assembles the optimal context window from the three temperature tiers.

**Algorithm (ContextPack):**

```
Input:
  - system_prompt: S (fixed, always included)
  - current_goal: G
  - hot_items: List[ContextItem]  # recent turns, active tool results
  - warm_items: List[ContextItem]  # compressed history, decisions
  - cold_candidates: List[ContextItem]  # retrieved from archive
  - token_budget: B
  - reserved_output: R  # tokens reserved for model response

Output:
  - packed_context: List[ContextItem]  # ordered items for context window

Algorithm:
  1. available = B - tokens(S) - tokens(G) - R
  2. Sort hot_items by recency (newest first)
  3. Pack hot_items until 40% of available is used or hot_items exhausted
  4. Sort warm_items by importance_score (highest first)
  5. Pack warm_items until 35% of available is used or warm_items exhausted
  6. Sort cold_candidates by relevance_to_current_goal
  7. Pack cold_candidates into remaining space
  8. If still under budget, expand hot_items (include older recent turns)
  9. Return packed_context ordered by: system_prompt, warm_context,
     cold_context, hot_context (recent at bottom, closest to generation)
```

**Rationale for ordering:** LLMs show primacy and recency bias. Place stable context (summaries, knowledge) early where primacy bias helps recall. Place recent turns last where recency bias ensures accurate continuation.

### 6.4 Progressive Summarization Engine

The summarization engine operates on a schedule, not on every turn.

**Trigger conditions (any of):**
1. Hot memory exceeds 40% budget.
2. N steps since last summarization (configurable, default: 20).
3. Goal change detected.
4. Total token spend since last summarization exceeds threshold.

**Summarization levels:**

| Level | Age (steps) | Detail | Method |
|-------|-------------|--------|--------|
| L0 (Verbatim) | 0-10 | Full messages, full tool results | No compression |
| L1 (Condensed) | 11-50 | Actions + key results, tool outputs truncated | Observation masking + selective trimming |
| L2 (Summary) | 51-200 | Decision points, outcomes, state changes | LLM summarization with structured prompt |
| L3 (Digest) | 200+ | Goals achieved, lessons learned, key facts | LLM summarization into structured schema |

**Important:** L1 uses observation masking (per JetBrains research), NOT LLM summarization. This is cheaper, faster, and avoids trajectory elongation.

### 6.5 Importance Scoring Function

Each context item receives a composite importance score:

```
importance(item) = w_r * recency(item)
                 + w_v * relevance(item, current_goal)
                 + w_c * causal_weight(item)
                 + w_f * reference_frequency(item)
                 + w_s * success_correlation(item)
                 + w_d * domain_weight(item)
```

Where:
- `recency(item)` = exponential decay from creation time.
- `relevance(item, goal)` = semantic similarity (embedding distance) to current goal.
- `causal_weight(item)` = 1.0 if item is a decision that affects downstream steps, 0.0 otherwise.
- `reference_frequency(item)` = how often this item has been referenced in subsequent steps.
- `success_correlation(item)` = from episodic memory: items from successful trajectories score higher.
- `domain_weight(item)` = configurable per-domain weights (e.g., file paths score high in coding tasks).

Default weights: `w_r=0.25, w_v=0.25, w_c=0.20, w_f=0.10, w_s=0.10, w_d=0.10`.

### 6.6 Structured State Extraction

At each summarization boundary, extract structured state into a `TaskState` object:

```python
@dataclass
class TaskState:
    """Extracted structured state from conversation history."""
    current_goal: str
    sub_goals: List[SubGoal]            # Active sub-goals with status
    decisions_made: List[Decision]       # Key decisions and their rationale
    entities: Dict[str, EntityState]     # Files, APIs, services, etc.
    constraints: List[str]              # Active constraints and requirements
    open_questions: List[str]           # Unresolved questions
    progress_percentage: float          # Estimated overall progress
    error_patterns: List[ErrorPattern]  # Recurring errors and their resolutions
    tool_usage_stats: Dict[str, ToolStats]  # Per-tool success rates
```

This structured state is always included in the context window (part of Warm Memory) and is updated at each summarization cycle. It provides the LLM with a structured overview of the task state, reducing reliance on the conversational history for state tracking.

### 6.7 Domain-Aware Compression Profiles

Configurable compression profiles that understand domain-specific importance:

```python
CODING_PROFILE = CompressionProfile(
    high_importance=["file_path", "function_signature", "test_result",
                     "error_message", "git_diff"],
    low_importance=["verbose_log", "package_install_output",
                    "full_file_listing"],
    preserve_verbatim=["code_snippet", "configuration_change"],
    tool_output_trim={
        "file_read": 500,      # chars to keep from file reads
        "shell_exec": 200,     # chars to keep from shell output
        "search_results": 300, # chars per search result
    },
)

RESEARCH_PROFILE = CompressionProfile(
    high_importance=["source_citation", "key_finding", "data_point",
                     "methodology", "conclusion"],
    low_importance=["search_query", "navigation_step",
                    "page_load_status"],
    preserve_verbatim=["direct_quote", "statistical_result"],
    tool_output_trim={
        "web_search": 400,
        "document_read": 600,
        "database_query": 300,
    },
)
```

### 6.8 Context Recovery and Checkpointing

**Context checkpoints:** At configurable intervals (default: every 50 steps), the full context state (Hot + Warm + structured TaskState) is persisted.

**Recovery protocol:**
1. If the LLM produces an inconsistent response (detected via output validation), roll back to the last checkpoint.
2. Re-pack context from the checkpoint, incorporating any Cold Memory items that may have been relevant but were evicted.
3. Retry the step with the restored context.

**Checkpoint storage:** Uses the existing corteX `MemoryFabric` backend system (InMemoryBackend for dev, FileBackend for persistent, extensible to Redis/SQLite).

---

## 7. Built-In vs. SDK-Configurable Boundary

### 7.1 Built-In (Always Active, Not Configurable)

These are core to corteX's value proposition and must work without developer configuration:

| Feature | Rationale |
|---------|-----------|
| Token counting per step | Fundamental to all context management |
| Context window overflow prevention | Safety mechanism; must never exceed limits |
| Hot/Warm/Cold temperature assignment | Core architectural pattern |
| Observation masking for L1 compression | Proven most cost-effective by research |
| Structured TaskState extraction | Required for multi-thousand-step coherence |
| Context checkpoint persistence | Required for fault tolerance |
| Usage telemetry and cost tracking | Enterprise requirement |

### 7.2 SDK-Configurable (Developer Controls)

| Feature | Configuration | Default |
|---------|--------------|---------|
| Token budget per step | `context_config.token_budget` | 80% of model's context window |
| Output token reservation | `context_config.output_reservation` | 4096 tokens |
| Summarization trigger interval | `context_config.summarize_every_n_steps` | 20 steps |
| Compression profile | `context_config.compression_profile` | `GENERAL_PROFILE` |
| Importance weights | `context_config.importance_weights` | Default weights |
| Hot/Warm/Cold budget ratios | `context_config.tier_ratios` | [0.40, 0.35, 0.25] |
| Summarization model | `context_config.summarization_model` | Same as primary (can be cheaper) |
| Custom summarization prompt | `context_config.summarization_prompt` | Built-in structured prompt |
| L2/L3 summarization style | `context_config.summarization_style` | "structured" |
| Checkpoint interval | `context_config.checkpoint_every_n_steps` | 50 steps |
| Max total token budget | `context_config.max_total_tokens` | None (unlimited) |
| Context recovery strategy | `context_config.recovery_strategy` | "rollback_to_checkpoint" |

### 7.3 Extension Points (For Advanced Users)

| Extension | Interface | Purpose |
|-----------|-----------|---------|
| Custom importance scorer | `ImportanceScorer` protocol | Domain-specific scoring |
| Custom compression profile | `CompressionProfile` dataclass | Domain-specific compression rules |
| Custom state extractor | `StateExtractor` protocol | Extract domain-specific structured state |
| Custom memory backend | `MemoryBackend` ABC (existing) | Alternative storage (Redis, Qdrant, etc.) |
| Context middleware | `ContextMiddleware` protocol | Intercept and modify context before LLM calls |
| Summarization validator | `SummarizationValidator` protocol | Validate summary quality before accepting |

---

## 8. Mathematical Models

### 8.1 Token Budget Formula

For a workflow of N total steps with model context window C:

```
Available per step:
  A(step) = C - S_system - R_output - O_fixed

Where:
  C = model context window (e.g., 200,000 for Claude, 1,000,000 for Gemini)
  S_system = system prompt tokens (typically 500-2000)
  R_output = reserved output tokens (typically 4096-8192)
  O_fixed = fixed overhead (tool definitions, etc.)
  A(step) = available tokens for dynamic context
```

### 8.2 Compression Ratio Model

As steps increase, the required compression ratio increases:

```
For N steps completed, each generating avg_tokens_per_step:

Total raw history tokens:
  H(N) = N * avg_tokens_per_step

Required compression ratio to fit in available budget A:
  CR(N) = H(N) / A

For a 10,000-step workflow with avg 2000 tokens/step and A = 150,000:
  H(10000) = 20,000,000 tokens
  CR(10000) = 133:1

This means at step 10,000, the system must compress 20M tokens of history
into 150K tokens of context -- a 133:1 compression ratio.
```

### 8.3 Progressive Summarization Compression Ratios

Each summarization level achieves different compression:

```
Level   | Compression Ratio | Method          | Quality Loss
--------|-------------------|-----------------|-------------
L0      | 1:1               | Verbatim        | 0%
L1      | 3:1 to 5:1        | Obs. masking    | ~5% (action history preserved)
L2      | 10:1 to 20:1      | LLM summary     | ~15-25% (key decisions preserved)
L3      | 50:1 to 100:1     | Structured digest| ~40-60% (goals/lessons only)
```

**Multi-level effective compression at step 10,000:**
```
Steps 1-10 (L0):     10 * 2000 = 20,000 tokens  -> 20,000 tokens (1:1)
Steps 11-50 (L1):    40 * 2000 = 80,000 tokens  -> 20,000 tokens (4:1)
Steps 51-200 (L2):   150 * 2000 = 300,000 tokens -> 20,000 tokens (15:1)
Steps 201-10000 (L3): 9800 * 2000 = 19,600,000   -> 30,000 tokens (653:1)
                                                   ----------
Total in context:                                   90,000 tokens
Effective overall ratio:                            222:1
```

This achieves the required 133:1 compression with room to spare.

### 8.4 Importance Scoring: Exponential Decay Model

```
recency(item) = exp(-lambda * (current_step - item.step))

Where lambda controls the decay rate:
  lambda = ln(2) / half_life_steps

Default half_life_steps = 30 (importance halves every 30 steps)

Example: Item from 100 steps ago:
  recency = exp(-0.0231 * 100) = exp(-2.31) = 0.099
  (approximately 10% of original recency score)
```

### 8.5 Total Cost Model

```
Cost per step (without context management):
  C_raw(step) = input_tokens(step) * price_per_input_token
              + output_tokens(step) * price_per_output_token

Cost per step (with CCE):
  C_cce(step) = A(step) * price_per_input_token      # Capped context size
              + output_tokens(step) * price_per_output_token
              + summarization_cost(step)              # Amortized over interval

Where summarization_cost is amortized:
  summarization_cost(step) = (summary_input + summary_output) * price
                           / summarization_interval

Expected savings at step N:
  Savings(N) = 1 - C_cce(N) / C_raw(N)

For N=100 (still within context window): Savings ~= 0% (no compression needed)
For N=500 (moderate compression): Savings ~= 40-60%
For N=5000 (heavy compression): Savings ~= 85-95%
For N=10000 (maximum compression): Savings ~= 95-99%
```

### 8.6 Context Quality Decay Model

Based on Chroma's context rot research, model performance degrades with context size:

```
quality(tokens_in_context) = Q_max * exp(-alpha * tokens_in_context / C)

Where:
  Q_max = peak quality (when context is minimal)
  alpha = model-specific decay constant
  C = model context window size

Empirical alpha values (approximate, from context rot research):
  Claude Opus 4: alpha ~= 0.3  (moderate decay)
  GPT-4.1:       alpha ~= 0.4  (faster decay)
  Gemini 2.5 Pro: alpha ~= 0.25 (slower decay, larger window helps)
```

**Optimal context size** is NOT "fill the window" but rather the point where:
```
d(quality * information) / d(tokens) = 0

This typically occurs at 50-70% of the context window, depending on model.
```

This is why CCE targets 80% of context window as the budget, not 95%.

---

## 9. Implementation Roadmap

### Phase 1: Foundation (Weeks 1-3)

**Goal:** Token-aware context management with basic compression.

**Deliverables:**
1. `TokenCounter` - Model-aware token counting (tiktoken for OpenAI, Anthropic API for Claude, Gemini tokenizer for Gemini).
2. `ContextWindowPacker` - Basic packer that assembles context within token budget.
3. `ObservationMasker` - L1 compression: replace old tool outputs with placeholders.
4. `ContextConfig` - Configuration dataclass with all SDK-configurable parameters.
5. Integration with existing `MemoryFabric` as the storage layer.

**Validation:**
- Unit tests: token counting accuracy within 2% of API ground truth.
- Integration test: 100-step workflow stays within context budget.
- Benchmark: measure token savings vs. raw context at 50, 100, 200 steps.

### Phase 2: Progressive Summarization (Weeks 4-6)

**Goal:** Multi-resolution temporal compression.

**Deliverables:**
1. `ProgressiveSummarizer` - Implements L0/L1/L2/L3 compression levels.
2. `ImportanceScorer` - Composite scoring function with configurable weights.
3. `TaskStateExtractor` - Extracts structured state from conversation history.
4. `SummarizationScheduler` - Triggers summarization based on configurable conditions.
5. Warm Memory management: promotion/demotion between Hot and Warm tiers.

**Validation:**
- Unit tests: summarization preserves key decisions (LLM-as-judge evaluation).
- Integration test: 1000-step workflow maintains task coherence.
- A/B test: compare task completion rates with vs. without progressive summarization.
- Cost benchmark: measure total token spend reduction vs. Anthropic compaction.

### Phase 3: Intelligent Context (Weeks 7-10)

**Goal:** Domain-aware compression and structured context.

**Deliverables:**
1. `CompressionProfile` system with built-in profiles for coding, research, operations.
2. Cold Memory integration: archive and retrieval from MemoryFabric backends.
3. `ContextCheckpointer` - Periodic full context snapshots.
4. `ContextRecovery` - Rollback to checkpoint on detected corruption.
5. Relevance-based Cold Memory promotion (semantic search for relevant archived items).

**Validation:**
- Integration test: 5000-step coding workflow with context recovery.
- Domain test: coding profile preserves file paths and signatures vs. generic profile.
- Recovery test: inject context corruption, verify successful rollback.

### Phase 4: Enterprise & Optimization (Weeks 11-14)

**Goal:** Production-ready with observability and cost optimization.

**Deliverables:**
1. `ContextTelemetry` - OpenTelemetry integration for context metrics (compression ratios, importance distributions, eviction rates).
2. Cost optimizer: auto-select summarization model (use cheaper model when available).
3. Concurrent context management for parallel sub-tasks.
4. `SummarizationValidator` - Quality assurance for generated summaries.
5. Context management dashboard integration with corteX UI Kit.
6. Load testing: 10,000-step workflow with realistic tool usage patterns.

**Validation:**
- Load test: 10,000-step workflow completes without context-related failures.
- Cost benchmark: demonstrate 90%+ token savings vs. raw context at scale.
- Quality benchmark: task completion rate at 10,000 steps is within 10% of step 100.
- Telemetry: all context operations are observable via standard tooling.

### Phase 5: Advanced Features (Weeks 15+)

**Goal:** Competitive differentiation.

**Deliverables:**
1. Reasoning chain preservation (inspired by Gemini thought signatures but model-agnostic).
2. Cross-workflow learning: share context insights across different workflow instances.
3. Adaptive compression: auto-tune compression parameters based on task performance.
4. Predictive context prefetching: anticipate what Cold Memory items will be needed next.
5. Multi-model context optimization: different packing strategies for different LLM providers.

---

## 10. References

### Anthropic / Claude Code
- [Compaction - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/compaction)
- [Context Editing - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/context-editing)
- [Context Windows - Claude API Docs](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Managing Context on the Claude Developer Platform](https://www.anthropic.com/news/context-management)
- [How Claude Code Works](https://code.claude.com/docs/en/how-claude-code-works)
- [What is Claude Code Auto-Compact - ClaudeLog](https://claudelog.com/faqs/what-is-claude-code-auto-compact/)
- [Claude Code Compaction - Steve Kinney](https://stevekinney.com/courses/ai-development/claude-code-compaction)
- [Claude Code Context Buffer: The 33K-45K Token Problem](https://claudefa.st/blog/guide/mechanics/context-buffer-management)
- [How Claude Code Got Better by Protecting More Context](https://hyperdev.matsuoka.com/p/how-claude-code-got-better-by-protecting)

### OpenAI Agents SDK
- [Context Management - OpenAI Agents SDK](https://openai.github.io/openai-agents-python/context/)
- [Session Memory - OpenAI Cookbook](https://developers.openai.com/cookbook/examples/agents_sdk/session_memory)
- [Context Personalization - OpenAI Cookbook](https://developers.openai.com/cookbook/examples/agents_sdk/context_personalization/)
- [Handoffs - OpenAI Agents SDK](https://openai.github.io/openai-agents-python/handoffs/)
- [A Practical Guide to Building Agents - OpenAI](https://openai.com/business/guides-and-resources/a-practical-guide-to-building-ai-agents/)

### LangChain / LangGraph
- [ConversationSummaryMemory - LangChain Docs](https://python.langchain.com/api_reference/langchain/memory/langchain.memory.summary.ConversationSummaryMemory.html)
- [ConversationSummaryBufferMemory - LangChain Docs](https://python.langchain.com/api_reference/langchain/memory/langchain.memory.summary_buffer.ConversationSummaryBufferMemory.html)
- [Chatbots Memory - LangChain How-To](https://python.langchain.com/docs/how_to/chatbots_memory/)
- [Persistence - LangGraph Docs](https://docs.langchain.com/oss/python/langgraph/persistence)
- [Message Handling and Summarization - LangChain Academy / DeepWiki](https://deepwiki.com/langchain-ai/langchain-academy/5.2-message-handling-and-summarization)
- [Conversational Memory for LLMs with Langchain - Pinecone](https://www.pinecone.io/learn/series/langchain/langchain-conversational-memory/)

### Google Gemini / ADK
- [Introduction to Conversational Context: Session, State, and Memory - ADK Docs](https://google.github.io/adk-docs/sessions/)
- [Context - ADK Docs](https://google.github.io/adk-docs/context/)
- [Building Agents with the ADK and the Interactions API - Google Developers Blog](https://developers.googleblog.com/building-agents-with-the-adk-and-the-new-interactions-api/)
- [Remember This: Agent State and Memory with ADK - Google Cloud Blog](https://cloud.google.com/blog/topics/developers-practitioners/remember-this-agent-state-and-memory-with-adk)
- [Thought Signatures - Gemini API](https://ai.google.dev/gemini-api/docs/thought-signatures)
- [Gemini 3 Developer Guide](https://ai.google.dev/gemini-api/docs/gemini-3)

### Letta / MemGPT
- [MemGPT - Letta Docs](https://docs.letta.com/concepts/memgpt/)
- [Memory Overview - Letta Docs](https://docs.letta.com/guides/agents/memory/)
- [Understanding Memory Management - Letta Docs](https://docs.letta.com/advanced/memory-management/)
- [Research Background - Letta Docs](https://docs.letta.com/concepts/letta/)
- [MemGPT: Towards LLMs as Operating Systems (arXiv:2310.08560)](https://arxiv.org/abs/2310.08560)

### Mem0
- [Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory (arXiv:2504.19413)](https://arxiv.org/abs/2504.19413)
- [Graph Memory - Mem0 Docs](https://docs.mem0.ai/open-source/features/graph-memory)
- [AI Memory Research: 26% Accuracy Boost - Mem0](https://mem0.ai/research)

### Microsoft Agent Framework
- [Introduction to Microsoft Agent Framework - Microsoft Learn](https://learn.microsoft.com/en-us/agent-framework/overview/agent-framework-overview)
- [Microsoft's Agentic Frameworks: AutoGen and Semantic Kernel - AutoGen Blog](https://devblogs.microsoft.com/autogen/microsofts-agentic-frameworks-autogen-and-semantic-kernel/)

### Academic Research
- [Context Rot: How Increasing Input Tokens Impacts LLM Performance - Chroma Research](https://research.trychroma.com/context-rot)
- [The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization - JetBrains Research / NeurIPS 2025](https://blog.jetbrains.com/research/2025/12/efficient-context-management/)
- [Chain of Agents: LLMs Collaborating on Long-Context Tasks - Google Research / NeurIPS 2024](https://research.google/blog/chain-of-agents-large-language-models-collaborating-on-long-context-tasks/)
- [Memory in the Age of AI Agents: A Survey](https://github.com/Shichun-Liu/Agent-Memory-Paper-List)
- [Evaluating Long-Context Reasoning in LLM-Based WebAgents (arXiv:2512.04307)](https://arxiv.org/abs/2512.04307)

### Token Counting & Cost
- [Token Counting Explained: tiktoken, Anthropic, and Gemini (2025 Guide) - Propel](https://www.propelcode.ai/blog/token-counting-tiktoken-anthropic-gemini-guide-2025)
- [How Tiktoken Stops AI Token Costs From Exploding in Production - Galileo](https://galileo.ai/blog/tiktoken-guide-production-ai)

---

## Appendix A: corteX Current Memory Architecture

The existing corteX `MemoryFabric` (in `corteX/engine/memory.py`) provides:

- **WorkingMemory:** Capacity-limited (default: 100 items), importance-based eviction. Scoring formula: `importance - (age_hours * 0.1)`.
- **EpisodicStore:** Stores trajectories with success/failure labels and quality ratings. Capacity: 500 episodes. Semantic search by goal text.
- **SemanticStore:** Factual knowledge with confidence scores. Capacity: 1000 entries. Topic-based search.
- **MemoryFabric:** Unified interface with consolidation (promote important working memories to episodic/semantic). Pluggable backends (InMemory, File).
- **ContextBroker** (in `corteX/memory/manager.py`): Production memory management with Gemini File Search and Context Caching integration.

**What's missing:**
- No conversation history compression or summarization.
- No token counting or budget management.
- No context window packing.
- No progressive summarization (multi-resolution temporal compression).
- No observation masking.
- No structured state extraction.
- No context checkpointing or recovery.
- No domain-aware compression profiles.

The proposed Cortical Context Engine (CCE) extends the existing architecture by adding a context management layer between the MemoryFabric and the LLM provider, using the existing memory systems as storage backends while adding the intelligence needed for 10,000+ step workflows.

---

## Appendix B: Competitive Positioning Summary

```
                    Short Sessions       Medium Sessions      Long Sessions
                    (10-50 steps)        (50-500 steps)       (500-10,000+ steps)
                    ─────────────        ──────────────       ──────────────────

Anthropic           Excellent            Good                 Degrades (flat summary)
OpenAI SDK          Good                 Fair                 Fails
LangChain/Graph     Fair (manual)        Poor (manual)        Fails
Google ADK          Good (big window)    Good                 Fair (huge window helps)
Letta/MemGPT        Good                 Fair (expensive)     Degrades (LLM overhead)
Mem0                Good                 Good                 Not designed for this

corteX (target)     Good                 Excellent            Excellent
```

corteX does not need to win at short sessions (every competitor already handles those). The differentiation is in the 500-10,000+ step range where every existing framework either fails, degrades significantly, or requires extensive custom engineering from the developer.
