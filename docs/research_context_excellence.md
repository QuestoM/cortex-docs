# Context Excellence Research Report

## What Should Go Into Every LLM Call to Make the Model Truly Understand the Situation?

**Date:** 2026-02-15
**Author:** Senior AI Engineer Research
**Scope:** Comprehensive analysis of context compilation strategies for the corteX Enterprise AI Agent SDK
**Status:** Research Only -- No Code Changes

---

## Table of Contents

1. [What the LLM Should "See" in Each Call](#1-what-the-llm-should-see-in-each-call)
2. [Context Window Budget Allocation](#2-context-window-budget-allocation)
3. [Context Quality Metrics](#3-context-quality-metrics)
4. [Competitive Analysis](#4-competitive-analysis)
5. [Gap Analysis of corteX](#5-gap-analysis-of-cortex)
6. [Ranked Recommendations](#6-ranked-recommendations)
7. [Pseudocode for Top Recommendations](#7-pseudocode-for-top-recommendations)
8. [References](#8-references)

---

## 1. What the LLM Should "See" in Each Call

### 1.1 The Anatomy of a Perfect LLM Call

Based on extensive analysis of Claude Code, Cursor, Devin, Anthropic's engineering blog, JetBrains NeurIPS 2025 research, Chroma's Context Rot study, and academic surveys, every LLM call in an agentic system should contain these elements, ordered from most stable (top of context) to most volatile (bottom of context):

#### Layer 1: Identity & Capabilities (Stable -- System Prompt)

```
[IDENTITY]
- Who am I? (agent name, role, specialization)
- What am I capable of? (constraints, permissions)
- What are my behavioral guidelines? (tone, style, safety rules)

[CAPABILITIES]
- Available tools (name, description, parameters, success rates)
- Tool ordering by Bayesian preference (most reliable first)
- Quarantined/disabled tools and why

[POLICIES]
- Enterprise safety policies
- Blocked topics
- Compliance rules
- Data handling requirements
```

**Key insight from Claude Code:** The system prompt is modular, not monolithic. Claude Code uses ~40 "system reminder" fragments activated conditionally based on environment and configuration. This allows the identity layer to be cache-friendly (stable prefix) while still context-sensitive.

**Key insight from Anthropic's context engineering blog:** "Use distinct sections with XML tagging or Markdown headers" -- sections like `<background_information>`, `<instructions>`, and `## Tool guidance` create clear cognitive boundaries for the model.

#### Layer 2: Persistent Context (Semi-Stable -- Changes Infrequently)

```
[GOAL]
- Original goal (full text, always present)
- Current sub-goal (if decomposed)
- Plan summary (completed/pending/failed steps)
- Progress percentage

[BRAIN STATE -- Behavioral Configuration]
- Significant weight signals translated to natural language
  (e.g., "Be concise" not "verbosity: -0.4")
- Active column/specialization mode
- Calibration warnings if confidence is degrading
- Prediction context (surprise levels, expected outcomes)

[USER CONTEXT]
- User preferences (communication style, technical level)
- Session history summary (for returning users)
- Enterprise tenant configuration
```

**Key insight from research:** The goal should appear at BOTH the beginning and end of the context window. This exploits the primacy-recency bias documented in "Lost in the Middle" (Liu et al., 2023) and confirmed by 2025 follow-up research (arXiv 2508.07479). The U-shaped attention curve means information at the boundaries gets the most attention.

#### Layer 3: Working Memory (Dynamic -- Changes Every Turn)

```
[TASK STATE]
- Current step description
- What was just attempted
- What succeeded/failed
- Scratchpad notes (agent's own notes for future turns)

[RELEVANT MEMORIES]
- Episodic: "Last time we tried X, it resulted in Y"
- Semantic: "Known fact about Z"
- Working: Recent conversation highlights

[TOOL RESULTS]
- Recent tool outputs (observation-masked for older ones)
- Error patterns detected
```

#### Layer 4: Recent Conversation (Most Volatile -- Hot Zone)

```
[RECENT MESSAGES]
- Last N user/assistant exchanges (newest at bottom)
- Tool call/result pairs
- Any injected system messages (drift warnings, reflections)

[GOAL RESTATEMENT]
- "REMINDER: Your current goal is: ..."
- Current drift/loop warnings if applicable
```

### 1.2 Meta-Instructions: The "Situation Awareness" Block

The most sophisticated agentic systems include meta-instructions that tell the model WHERE it is in the overall process. This is a critical differentiator between naive and production-grade context:

```
## Situation Awareness
You are on step 47 of approximately 120.
Progress: 38% toward the original goal.
Last action: Wrote authentication middleware (succeeded).
Next planned step: Write unit tests for auth middleware.
Drift score: 0.12 (on track).
Loops detected: 0.
Time elapsed: 4 minutes.
Tokens spent: 23,450 of 100,000 budget.
```

**Why this matters:** Without situational awareness, the model has no sense of temporal position. It cannot judge whether to be thorough (early in a task) or to wrap up (near completion). It cannot detect that it has been going in circles. Multiple studies and production systems (Claude Code, Devin) confirm that explicit progress tracking significantly improves agent behavior.

### 1.3 Tool Presentation: Beyond Simple Listings

Current corteX presents tools as a comma-separated list of names in the system prompt. This is far below best practice. Tools should include:

1. **Name and description** -- What the tool does
2. **Parameters with types** -- JSON Schema format
3. **Usage guidance** -- When to use this tool vs. alternatives
4. **Success rate** -- "This tool has a 94% success rate over the last 20 calls"
5. **Known limitations** -- "Cannot handle files larger than 10MB"
6. **Ordering** -- Tools ordered by Bayesian preference (Thompson Sampling ranking)

**Key insight from Anthropic:** "Teams should ensure human engineers can definitively identify which tool applies in given situations before expecting AI agents to do so." Tools with overlapping functionality create ambiguous decision points that waste context and cause errors.

**Key insight from Spotify's engineering team:** "Deliberately excluded: Code search and documentation tools." Limiting the toolset "prioritizes predictability over capability." Broader tool exposure introduces "more dimensions of unpredictability."

---

## 2. Context Window Budget Allocation

### 2.1 The Optimal Budget Split

Based on analysis of production systems and research:

| Zone | Budget % | Purpose | Stability |
|------|----------|---------|-----------|
| System Prompt (Identity + Capabilities) | 10-15% | Agent identity, tools, policies | Very stable (KV-cache friendly) |
| Persistent Context (Goal + Brain + User) | 8-12% | Goal, plan, brain state, preferences | Semi-stable |
| Working Memory (Task State + Memories) | 25-35% | Current task context, memories, scratchpad | Dynamic |
| Recent Conversation (Hot Zone) | 35-45% | Recent messages, tool results | Most volatile |
| Output Reservation | 5-10% | Reserved for model response | Fixed |

**Critical finding from Chroma's Context Rot research (2025):** Optimal context utilization is 50-70% of the context window, NOT 95%. Performance degrades gradually with context size, with Claude models degrading the slowest. The current corteX default of 80% (`token_budget_ratio: 0.80`) is already reasonable but could benefit from being task-adaptive.

**Critical finding from "Lost in the Middle" (2025 update):** The primacy-recency bias is strongest when inputs occupy up to 50% of the context window. Beyond that, primacy bias weakens while recency bias remains stable. This means:
- Stable context (goals, identity) should go at the TOP (primacy position)
- Recent turns should go at the BOTTOM (recency position)
- The MIDDLE should contain compressed/summarized history (least critical to recall precisely)

### 2.2 Task-Adaptive Budget Allocation

Different task types warrant different allocations:

| Task Type | System | Persistent | Working | Recent | Rationale |
|-----------|--------|------------|---------|--------|-----------|
| Quick Chat | 15% | 5% | 20% | 50% | Conversation-heavy, minimal state |
| Coding | 12% | 10% | 35% | 35% | Need tool results + code context |
| Long Agentic Task | 10% | 12% | 40% | 30% | Need plan + memory + state |
| Research | 10% | 8% | 45% | 30% | Need retrieved knowledge + summaries |
| Debugging | 12% | 10% | 30% | 40% | Need recent errors + stack traces |

**corteX already has profiles** (`CODING_PROFILE`, `RESEARCH_PROFILE`) in `context.py` but they only control compression behavior, NOT budget allocation. This is a gap.

### 2.3 Progressive Summarization Strategy

The transition as context fills up should follow this cascade:

```
0-50% utilization: L0 (Verbatim) -- Keep everything raw
50-70% utilization: L1 (Observation Masking) -- Mask old tool outputs
70-85% utilization: L2 (LLM Summarization) -- Summarize old conversation blocks
85-95% utilization: L3 (Emergency Digest) -- Replace all working memory with structured digest
95%+: Compaction Reset -- Keep only goal + digest + last 2 recent messages
```

**Key insight from JetBrains NeurIPS 2025:** Observation masking (L1) achieves 50%+ cost reduction WITHOUT trajectory elongation. LLM summarization causes ~15% longer trajectories because summaries obscure termination signals. The recommendation: **Always prefer observation masking over summarization. Use summarization only when masking alone is insufficient.**

**Key insight from Anthropic's long-running agents blog:** "Tool result clearing is the safest, lightest-touch compaction form." Always try clearing tool results before summarizing conversation.

### 2.4 What MUST Always Stay in Context vs. What Can Be Summarized

**ALWAYS in context (never summarize away):**
1. The original goal (verbatim, word-for-word)
2. Current plan/step state
3. Active errors and unresolved issues
4. User-specified constraints and preferences
5. Key decisions and their rationale (the "why")
6. The most recent 2-3 conversation turns
7. Brain state signals (drift warnings, loop alerts)

**Safe to summarize:**
1. Old tool outputs (especially verbose ones like file listings, search results)
2. Intermediate reasoning steps that led to already-recorded decisions
3. Successful tool calls (just record: "tool X succeeded, result: Y")
4. Package installation output, build logs
5. Repeated patterns of the same operation

**Safe to drop entirely:**
1. Duplicate information (same tool called twice with same result)
2. Superseded plans (only keep current plan)
3. Resolved errors (unless they recur)
4. Verbose formatting/whitespace in tool outputs

---

## 3. Context Quality Metrics

### 3.1 How to Measure Context Quality

Context quality is the degree to which the assembled context enables the model to produce the correct/desired output. We propose four measurable dimensions:

#### Metric 1: Goal Retention Score (GRS)

```
GRS = cosine_similarity(
    embed(assembled_context),
    embed(original_goal)
)
```

If the goal's semantic signature is diluted in the assembled context, the model is likely to drift. Track GRS across turns; a declining GRS predicts drift before GoalTracker detects it.

**Target:** GRS > 0.3 at all times. Alert if GRS drops below 0.2.

#### Metric 2: Signal-to-Noise Ratio (SNR)

```
SNR = tokens_of_high_importance_items / total_context_tokens
```

Where "high importance" is defined as items with importance score > 0.6. A context filled with low-importance items is noisy.

**Target:** SNR > 0.4. If SNR < 0.25, trigger aggressive compression.

#### Metric 3: Recency Coverage

```
RC = (number_of_recent_5_turns_in_context) / 5
```

If recent turns are being dropped due to budget pressure, the model loses track of the immediate conversation.

**Target:** RC = 1.0 (all 5 most recent turns always present). Alert if RC < 0.8.

#### Metric 4: Decision Preservation Rate (DPR)

```
DPR = (key_decisions_in_context) / (total_key_decisions_made)
```

If important decisions were summarized away, the model may reverse them.

**Target:** DPR > 0.8 for decisions made in the last 50 steps.

### 3.2 Signals That the Model Is "Confused" or "Lost Context"

Detectable signals indicating context quality degradation:

1. **Repeated tool calls with identical arguments** -- The model forgot it already did this (loop)
2. **Contradicting previous decisions** -- "Let's use PostgreSQL" after deciding on MongoDB
3. **Asking questions already answered** -- Re-asking the user for information already provided
4. **Reverting completed work** -- Deleting files it previously created with intent
5. **Sudden topic shift** -- Jumping to unrelated work mid-task
6. **Increasing error rate** -- More tool failures per turn over time
7. **Declining confidence signals** -- If structured output includes confidence, watch the trend
8. **Prediction surprise spikes** -- Sudden high surprise from PredictionEngine
9. **Goal drift acceleration** -- Drift score increasing faster than expected
10. **Output length inflation** -- Responses getting progressively longer (verbosity escape)

### 3.3 Detecting Summarization Loss

When important information is accidentally summarized away:

1. **Decision Reversal Detection:** Compare current decisions against the decision log. If the model makes a decision that contradicts a logged decision, the logged decision may have been summarized away.

2. **Entity Tracking:** Maintain a list of "active entities" (files, variables, API endpoints). If the model references an entity not in the current context but in the entity list, flag a potential summarization loss.

3. **Error Recurrence:** If the same error occurs twice after being "resolved," the resolution was likely summarized away.

4. **Scratchpad Checksums:** Maintain checksums of scratchpad notes. If a note disappears from context between turns, alert.

5. **Post-Compaction Validation:** After any compaction/summarization, run a lightweight check: "Can the model still correctly answer questions about key facts from the pre-compaction context?"

---

## 4. Competitive Analysis

### 4.1 Claude Code (Anthropic)

**Architecture:**
- Modular system prompt with ~40 conditional "system reminder" fragments
- 18+ builtin tools with individual specification documents
- Sub-agent architecture: Plan, Explore, Task agents with isolated context windows
- CLAUDE.md files as persistent memory substrates (loaded at session start)

**Context Management:**
- Compaction: Summarize conversation history + retain 5 most-recently accessed files
- Progressive disclosure: Agents incrementally discover context through exploration
- Just-in-time retrieval: Maintain lightweight identifiers, load data via tools at runtime
- Sub-agents return condensed summaries (1,000-2,000 tokens) to the orchestrator

**Key Pattern -- Prompt Caching:**
Claude Code's system prompt is designed for KV-cache reuse. The stable prefix (identity, tools, policies) is cached, reducing cost by up to 90% for repetitive calls.

**Strengths corteX should learn from:**
1. Conditional system reminders (not one monolithic prompt)
2. Sub-agent isolation (each sub-agent gets a clean, focused context)
3. Persistent memory files (CLAUDE.md) that survive context resets
4. Explicit file path tracking (know what files are "in scope")
5. Compaction that prioritizes tool result clearing over conversation summarization

### 4.2 Cursor AI

**Architecture:**
- Semantic codebase indexing using AST parsing and embeddings
- Vector database (Turbopuffer) for embedding storage and retrieval
- Merkle trees for incremental re-indexing (only changed files)
- Agent Mode for multi-file autonomous task execution

**Context Management:**
- Chunks code at semantic boundaries (between functions, not mid-line)
- File path obfuscation for privacy (paths masked before server transmission)
- Custom retrieval models analyze codebase for relevant files/functions
- ~40% context token utilization efficiency for retrieved context

**Strengths corteX should learn from:**
1. Semantic code chunking (not arbitrary character-based splitting)
2. Incremental indexing via Merkle trees (efficient re-indexing)
3. Dedicated retrieval models for context selection (not just TF-IDF)
4. Privacy-preserving architecture (path obfuscation)

### 4.3 Devin (Cognition)

**Architecture:**
- Full development environment with browser, terminal, editor
- Devin Wiki / Devin Search for codebase understanding
- Multi-session capability with persistent state

**Context Management:**
- Dual-Layer Architecture: Hot Path (recent messages + summarized state) and Cold Path (retrieval from vector store)
- Memory Node that synthesizes what to save after each turn
- Separate write policies for interaction-driven memory

**Strengths corteX should learn from:**
1. Memory Node pattern (dedicated synthesis of what to persist)
2. Separate hot/cold paths with different retrieval strategies
3. Write policies controlling what gets persisted vs. discarded

### 4.4 OpenAI Assistants API

**Architecture:**
- Thread-based conversation management (up to 100K messages per thread)
- Built-in truncation strategies: `auto` and `last_messages`

**Context Management:**
- `auto` mode: drops middle messages to fit context, keeping first and last
- `last_messages` mode: keeps only N most recent messages
- Token budget control: `max_prompt_tokens` and `max_completion_tokens`
- "Smart" truncation: attempts to drop least important messages first

**Strengths corteX should learn from:**
1. User-configurable truncation strategies (auto vs. last_messages)
2. Explicit token budget parameters exposed to developers
3. Middle-dropping strategy (aligned with Lost-in-the-Middle research)

### 4.5 Letta/MemGPT

**Architecture:**
- OS-inspired virtual context management
- LLM itself manages memory through designated memory-editing tools
- Hierarchical memory: Core Memory (always in context) + Archival Memory (searchable store)
- Interrupt-based control flow between agent and user

**Context Management (as of 2026):**
- Context Repositories: programmatic context management with git-based versioning
- Conversations API: shared memory across parallel agent experiences
- The LLM explicitly moves data in/out of context using memory tools

**Strengths corteX should learn from:**
1. Agent-controlled memory editing (the agent decides what to keep/discard)
2. Core Memory concept (information that ALWAYS stays in context)
3. Git-based versioning of context state (rollback capability)
4. Treating context management as an explicit tool the agent can invoke

### 4.6 LangGraph (LangChain)

**Architecture:**
- Graph-based workflow with explicit state schemas (TypedDict + Annotated types)
- Centralized state accessible to all nodes
- Checkpointing with time-travel capability

**Context Management:**
- Database-backed checkpointing (PostgresSaver)
- Cross-thread memory stores (MongoDB, Redis)
- Vector search for semantic retrieval
- File URLs instead of file contents in state (avoid bloat)

**Strengths corteX should learn from:**
1. Database-backed persistent checkpointing (not just in-memory)
2. Time-travel debugging (restore any checkpoint)
3. Storing references instead of full content

---

## 5. Gap Analysis of corteX

### 5.1 What corteX Does Well

1. **Four-zone context architecture** (`context_compiler.py`): System/Persistent/Working/Recent zones with defined budget ratios. This is structurally sound and aligns with best practices.

2. **Goal-at-both-ends pattern** (`context_compiler.py`): The goal appears in the Persistent zone AND as a restatement at the end. This exploits primacy-recency bias correctly.

3. **Brain State Injection** (`brain_state_injector.py`): Translating cognitive signals (weights, calibration, predictions) into natural-language instructions is a unique differentiator. No competitor does this.

4. **Progressive summarization pipeline** (`context.py`, `context_summarizer.py`): L0-L3 compression with observation masking as L1 is aligned with JetBrains NeurIPS 2025 findings.

5. **Importance scoring** (`context.py`, `semantic_scorer.py`): Multi-factor importance scoring (recency, relevance, causal, reference frequency, success correlation, domain) is comprehensive.

6. **Task state tracking** (`context.py`): The `TaskState` dataclass captures goals, sub-goals, decisions, entities, constraints, errors, and tool usage. This is thorough.

7. **Dual context management** (`context.py` + `context_compiler.py`): Having both a detailed CorticalContextEngine and a lightweight ContextCompiler gives flexibility for different execution paths.

8. **Rate-limited summarization** (`context_summarizer.py`): Rate limiting on LLM summarization calls with truncation fallback prevents runaway API costs.

9. **Scratchpad** (`context_compiler.py`): Agent-writable scratchpad notes in working memory. This aligns with Anthropic's recommendation of structured note-taking.

10. **Checkpointing** (`context.py`): Periodic context checkpoints for fault-tolerant recovery.

### 5.2 What corteX Is Missing

#### GAP 1: Tool Presentation is Minimal (CRITICAL)

**Current state:** `context_compiler.py` line 225-226 only includes tool names as a comma-separated list:
```python
parts.append("## Available Tools\n" + ", ".join(
    t.get("name", "?") for t in self._tool_definitions))
```

**Impact:** The model has no information about what each tool does, its parameters, success rates, or when to use it vs. alternatives. This is one of the most impactful gaps.

**Recommendation:** Include full tool definitions with descriptions, parameters, success rates, and usage guidance. Anthropic explicitly states tools should be "self-contained with clear intended use."

#### GAP 2: No Situational Awareness Block (HIGH)

**Current state:** The model receives brain state (weights, calibration, predictions) and goal progress, but there is no unified "situation awareness" block telling the model:
- What step it is on
- How far through the plan it is
- How much token budget remains
- How long the session has been running
- Whether it should be wrapping up

**Impact:** The model cannot self-regulate its depth/breadth tradeoff based on position in the task.

#### GAP 3: No Adaptive Budget Allocation (HIGH)

**Current state:** Zone budgets are fixed ratios:
```python
DEFAULT_ZONE_BUDGETS = {
    "system": 0.12, "persistent": 0.08, "working": 0.40, "recent": 0.40
}
```

**Impact:** A quick chat gets the same allocation as a 1000-step agentic task. Quick chats waste budget on working memory; long tasks starve for it.

**Recommendation:** Implement task-adaptive budget allocation that shifts ratios based on task type and session length.

#### GAP 4: Context Compiler and CorticalContextEngine Are Not Unified (HIGH)

**Current state:** Two parallel context management systems exist:
- `CorticalContextEngine` in `context.py` (detailed, with compression/scoring/checkpointing)
- `ContextCompiler` in `context_compiler.py` (4-zone, KV-cache-oriented)

They operate independently, with the SDK manually syncing between them:
```python
# sdk.py manually adds to both:
self.context_engine.add_user_message(self._turn_count, message)
# AND separately:
self._context_compiler.append_message("user", message, importance=0.7)
```

**Impact:** Dual maintenance, potential state divergence, wasted tokens from redundant tracking.

**Recommendation:** Unify them. The ContextCompiler should be the front-end (manages zone assembly and KV-cache), while CorticalContextEngine provides the backend (compression, scoring, storage). The API should be single-path.

#### GAP 5: No Context Quality Monitoring (MEDIUM-HIGH)

**Current state:** No mechanism to detect:
- Goal Retention Score declining
- Signal-to-Noise Ratio dropping
- Decision reversals after summarization
- Important information being accidentally dropped

**Impact:** Context quality degrades silently. The brain detects drift (GoalTracker) and surprise (PredictionEngine) but does not detect context assembly quality issues.

#### GAP 6: Memory Retrieval Is Not Goal-Aware (MEDIUM)

**Current state:** Memory retrieval in `_retrieve_memory_context` does keyword matching:
```python
context = self.memory.get_relevant_context(query, max_items=5)
```

The query is the raw user message. There is no integration between the SemanticImportanceScorer (which has the goal embedding) and memory retrieval.

**Impact:** Memories are retrieved based on surface similarity to the user's current message, not relevance to the overall goal.

#### GAP 7: No Few-Shot Examples from Past Trajectories (MEDIUM)

**Current state:** The `Trajectory` model exists in `memory/types.py` and `ContextBroker` can store/retrieve trajectories, but trajectories are never injected into the LLM context.

**Impact:** Missing the "few-shot from experience" pattern. Anthropic research shows examples are "pictures worth a thousand words."

#### GAP 8: No Conditional System Reminders (MEDIUM)

**Current state:** The system prompt is built as a single block in `_build_system_content()`. All information is always present.

**Impact:** Wasted tokens on irrelevant instructions. For example, tool guidance is included even when no tools are available. Safety policies are included even for simple chat.

**Recommendation:** Adopt Claude Code's pattern of conditional system reminder fragments.

#### GAP 9: No Cross-Session Context Persistence (MEDIUM)

**Current state:** When a session ends, all context is lost. The `ContextBroker` can persist trajectories to FileSearch, but there is no mechanism to load prior session state when starting a new session.

**Impact:** For long-running tasks that span multiple sessions, the agent starts from zero each time. Compare with Anthropic's initializer/coder agent pattern with `claude-progress.txt`.

#### GAP 10: Observation Masking Threshold Is Not Adaptive (LOW-MEDIUM)

**Current state:** L1 observation masking uses a fixed age threshold:
```python
if (current_step - item.step_number) > 10:  # Fixed at 10 steps
```

**Impact:** In a fast session, 10 steps might mean 30 seconds. In a slow session, it might mean an hour. The masking should be adaptive based on token pressure, not just age.

#### GAP 11: No Sub-Agent Context Isolation (LOW-MEDIUM)

**Current state:** The `SubAgentManager` exists but context is not isolated per sub-agent. All sub-agents share the parent's context window.

**Impact:** Sub-agents are polluted with irrelevant parent context. Anthropic's research shows "many agents with isolated contexts outperformed single-agent" because each sub-agent window can be narrowly focused.

---

## 6. Ranked Recommendations

### Tier 1: Highest Impact (Implement First)

| # | Recommendation | Impact | Effort | Gap |
|---|---------------|--------|--------|-----|
| 1 | **Rich Tool Presentation** -- Include full tool descriptions, parameters, success rates, and usage guidance in context | Very High | Medium | GAP 1 |
| 2 | **Situational Awareness Block** -- Add unified progress/state block to every LLM call | Very High | Low | GAP 2 |
| 3 | **Unify Context Engines** -- Merge ContextCompiler and CorticalContextEngine into a single pipeline | High | High | GAP 4 |
| 4 | **Task-Adaptive Budget Allocation** -- Shift zone ratios based on task type and session length | High | Medium | GAP 3 |

### Tier 2: High Impact (Implement Next)

| # | Recommendation | Impact | Effort | Gap |
|---|---------------|--------|--------|-----|
| 5 | **Context Quality Monitor** -- Implement GRS, SNR, Recency Coverage, Decision Preservation metrics | High | Medium | GAP 5 |
| 6 | **Goal-Aware Memory Retrieval** -- Use SemanticScorer goal embedding to rank retrieved memories | Medium-High | Low | GAP 6 |
| 7 | **Conditional System Reminders** -- Fragment the system prompt into conditional sections | Medium-High | Medium | GAP 8 |
| 8 | **Trajectory Injection** -- Inject relevant past trajectories as few-shot examples | Medium | Medium | GAP 7 |

### Tier 3: Important (Implement When Possible)

| # | Recommendation | Impact | Effort | Gap |
|---|---------------|--------|--------|-----|
| 9 | **Cross-Session Persistence** -- Save/load session state for multi-session continuity | Medium | High | GAP 9 |
| 10 | **Adaptive Observation Masking** -- Trigger masking based on token pressure, not just age | Medium | Low | GAP 10 |
| 11 | **Sub-Agent Context Isolation** -- Give sub-agents clean, focused context windows | Medium | Medium | GAP 11 |
| 12 | **Summarization Loss Detection** -- Detect when critical info was accidentally dropped | Medium | Medium | GAP 5 |

---

## 7. Pseudocode for Top Recommendations

### 7.1 Rich Tool Presentation (Recommendation #1)

```python
def _build_tool_context(self) -> str:
    """Build rich tool presentation for context window."""
    if not self._tool_definitions:
        return ""

    parts = ["## Available Tools\n"]
    for tool in self._tool_definitions:
        name = tool.get("name", "unknown")
        desc = tool.get("description", "No description")
        params = tool.get("parameters", {})
        success_rate = tool.get("success_rate")  # From Bayesian selector
        usage_hint = tool.get("usage_hint", "")

        tool_block = f"### {name}\n{desc}\n"
        if params:
            param_str = ", ".join(
                f"{p}: {info.get('type', 'any')}"
                for p, info in params.get("properties", {}).items()
            )
            tool_block += f"Parameters: {param_str}\n"
        if success_rate is not None:
            tool_block += f"Success rate: {success_rate:.0%} (last 20 calls)\n"
        if usage_hint:
            tool_block += f"When to use: {usage_hint}\n"
        parts.append(tool_block)

    return "\n".join(parts)
```

### 7.2 Situational Awareness Block (Recommendation #2)

```python
def _build_situation_awareness(self) -> str:
    """Build unified situational awareness block."""
    if not self._goal:
        return ""

    parts = ["## Situation Awareness"]

    # Step progress
    if self._plan_state:
        current = self._plan_state.get("current_step", 0)
        total = self._plan_state.get("total_steps", 0)
        progress = self._plan_state.get("progress", 0.0)
        parts.append(
            f"Step {current} of {total} | "
            f"Progress: {progress:.0%}"
        )

    # Goal drift status
    drift = self._brain_digest.get("drift_score", 0.0)
    if drift < 0.2:
        parts.append("Status: On track")
    elif drift < 0.5:
        parts.append(f"Status: Mild drift ({drift:.2f}) -- stay focused")
    else:
        parts.append(f"Status: SIGNIFICANT DRIFT ({drift:.2f}) -- refocus NOW")

    # Loop detection
    loop_count = self._brain_digest.get("loop_count", 0)
    if loop_count > 0:
        parts.append(
            f"WARNING: {loop_count} loop(s) detected. "
            "Try a fundamentally different approach."
        )

    # Token budget
    stats = self.get_stats()
    utilization = stats.get("utilization", 0.0)
    parts.append(
        f"Context utilization: {utilization:.0%} "
        f"({stats.get('total_tokens_estimate', 0):,} / "
        f"{self._max_tokens:,} tokens)"
    )

    # Session time
    if self._session_start_time:
        elapsed = time.time() - self._session_start_time
        parts.append(f"Session duration: {elapsed / 60:.1f} minutes")

    return "\n".join(parts)
```

### 7.3 Task-Adaptive Budget Allocation (Recommendation #4)

```python
TASK_BUDGET_PROFILES = {
    "conversation": {
        "system": 0.15, "persistent": 0.05,
        "working": 0.20, "recent": 0.50,
    },
    "coding": {
        "system": 0.12, "persistent": 0.10,
        "working": 0.35, "recent": 0.35,
    },
    "research": {
        "system": 0.10, "persistent": 0.08,
        "working": 0.45, "recent": 0.30,
    },
    "debugging": {
        "system": 0.12, "persistent": 0.10,
        "working": 0.30, "recent": 0.40,
    },
    "long_agentic": {
        "system": 0.10, "persistent": 0.12,
        "working": 0.40, "recent": 0.30,
    },
}

def adapt_budgets(self, task_type: str, turn_count: int) -> None:
    """Adapt zone budgets based on task type and session length."""
    base = TASK_BUDGET_PROFILES.get(task_type, DEFAULT_ZONE_BUDGETS)

    # Long sessions need more persistent/working, less recent
    if turn_count > 50:
        base = {
            "system": base["system"],
            "persistent": min(0.15, base["persistent"] + 0.03),
            "working": min(0.45, base["working"] + 0.05),
            "recent": max(0.25, base["recent"] - 0.08),
        }

    # Normalize to sum to 1.0
    total = sum(base.values())
    for zone_name, ratio in base.items():
        self._zones[zone_name].budget_ratio = ratio / total
```

### 7.4 Context Quality Monitor (Recommendation #5)

```python
@dataclass
class ContextQualityReport:
    """Metrics on the quality of the assembled context."""
    goal_retention_score: float       # Cosine sim of context vs. goal
    signal_to_noise_ratio: float      # High-importance tokens / total
    recency_coverage: float           # Recent turns present / expected
    decision_preservation_rate: float  # Decisions in context / all decisions
    overall_health: str               # "healthy", "degrading", "critical"
    warnings: List[str]


class ContextQualityMonitor:
    """Monitors and reports on context assembly quality."""

    def __init__(self, scorer: SemanticImportanceScorer):
        self._scorer = scorer
        self._decision_log: List[str] = []
        self._last_report: Optional[ContextQualityReport] = None

    def evaluate(
        self,
        compiled_context: CompiledContext,
        goal: str,
        all_decisions: List[str],
        recent_turn_count: int = 5,
    ) -> ContextQualityReport:
        warnings = []

        # 1. Goal Retention Score
        full_text = " ".join(m["content"] for m in compiled_context.messages)
        grs = self._scorer.score_relevance(full_text, [goal])

        if grs < 0.2:
            warnings.append(
                f"Goal retention critically low ({grs:.2f}). "
                "Context may have drifted from the goal."
            )

        # 2. Signal-to-Noise Ratio
        total_tokens = compiled_context.total_tokens
        # Estimate high-importance tokens from zone usage
        persistent_tokens = compiled_context.zone_usage.get("persistent", 0)
        system_tokens = compiled_context.zone_usage.get("system", 0)
        snr = (persistent_tokens + system_tokens) / max(1, total_tokens)

        if snr < 0.25:
            warnings.append(
                f"Signal-to-noise ratio low ({snr:.2f}). "
                "Consider compressing low-importance items."
            )

        # 3. Recency Coverage
        recent_msgs = [
            m for m in compiled_context.messages[-recent_turn_count:]
            if m["role"] in ("user", "assistant")
        ]
        rc = len(recent_msgs) / max(1, recent_turn_count)

        if rc < 0.8:
            warnings.append(
                f"Recent turn coverage low ({rc:.0%}). "
                "Important recent messages may have been dropped."
            )

        # 4. Decision Preservation Rate
        decisions_in_context = sum(
            1 for d in all_decisions
            if any(d[:50].lower() in m["content"].lower()
                   for m in compiled_context.messages)
        )
        dpr = decisions_in_context / max(1, len(all_decisions))

        if dpr < 0.8:
            warnings.append(
                f"Decision preservation low ({dpr:.0%}). "
                "Key decisions may have been summarized away."
            )

        # Overall health
        scores = [grs, snr, rc, dpr]
        avg = sum(scores) / len(scores)
        if avg > 0.6:
            health = "healthy"
        elif avg > 0.4:
            health = "degrading"
        else:
            health = "critical"

        report = ContextQualityReport(
            goal_retention_score=grs,
            signal_to_noise_ratio=snr,
            recency_coverage=rc,
            decision_preservation_rate=dpr,
            overall_health=health,
            warnings=warnings,
        )
        self._last_report = report
        return report
```

### 7.5 Goal-Aware Memory Retrieval (Recommendation #6)

```python
def _retrieve_memory_context(
    self, query: str, max_items: int = 5
) -> str:
    """Retrieve relevant memories ranked by goal relevance."""
    # Combine user query with goal for richer retrieval
    goal = ""
    if self._goal_tracker:
        goal = self._goal_tracker.original_goal

    # Build composite query: user message + goal context
    composite_query = query
    if goal:
        composite_query = f"{query} [Goal context: {goal[:200]}]"

    context = self.memory.get_relevant_context(
        composite_query, max_items=max_items * 2  # Over-retrieve
    )

    # Re-rank by goal relevance using SemanticScorer
    all_items = []
    for category in ("working", "episodic", "semantic"):
        for item in context.get(category, []):
            content = str(item.get("content", ""))
            relevance = self._semantic_scorer.score_relevance(
                content, [goal] if goal else None
            )
            all_items.append((relevance, category, item))

    # Sort by relevance and take top-k
    all_items.sort(key=lambda x: x[0], reverse=True)
    top_items = all_items[:max_items]

    # Format into context block
    parts = []
    for relevance, category, item in top_items:
        # ... format as before but with relevance ranking ...
        pass

    return formatted_block
```

---

## 8. References

### Industry Engineering Blogs

1. [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) -- Anthropic, September 2025. The definitive guide to context engineering strategies: trimming, isolation, sandboxing, compaction, structured note-taking, sub-agents.

2. [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) -- Anthropic, November 2025. Multi-session agent architecture with initializer/coder pattern and `claude-progress.txt` for cross-session state.

3. [Context Engineering: Background Coding Agents (Part 2)](https://engineering.atspotify.com/2025/11/context-engineering-background-coding-agents-part-2) -- Spotify Engineering, November 2025. Six prompt engineering practices for background agents, tool constraint philosophy.

4. [Context Engineering for Agents](https://blog.langchain.com/context-engineering-for-agents/) -- LangChain Blog, 2025. Integration patterns with LangGraph.

5. [From Artifacts to Organisms: Supercharging Development with Claude Code's Agentic Context Engineering](https://labs.adaline.ai/p/context-engineering-with-claude-code) -- Adaline Labs. Three-layer context structure: intent translation, knowledge synthesis, repository retrieval.

### System Prompt Analysis

6. [Claude Code System Prompts](https://github.com/Piebald-AI/claude-code-system-prompts) -- Piebald AI. Complete decomposition of Claude Code's system prompt: 18+ tools, ~40 system reminders, sub-agent prompts (Plan/Explore/Task).

### Academic Research

7. [The Complexity Trap: Simple Observation Masking Is as Efficient as LLM Summarization for Agent Context Management](https://openreview.net/forum?id=OHVzruJl5k) -- JetBrains Research, NeurIPS 2025 DL4Code Workshop. Key finding: observation masking halves cost while matching or exceeding LLM summarization performance. Summarization causes ~15% trajectory elongation.

8. [Lost in the Middle: How Language Models Use Long Contexts](https://direct.mit.edu/tacl/article/doi/10.1162/tacl_a_00638/119630/) -- Liu et al., 2023 (TACL). Foundational research on primacy-recency bias in LLM context windows.

9. [Positional Biases Shift as Inputs Approach Context Window Limits](https://arxiv.org/abs/2508.07479) -- 2025. Follow-up showing bias is strongest at <50% window utilization and shifts as context grows.

10. [Context Rot: How Increasing Input Tokens Impacts LLM Performance](https://research.trychroma.com/context-rot) -- Chroma Research, 2025. Evaluation of 18 LLMs showing gradual performance degradation with context size. Claude models degrade slowest.

11. [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) -- Packer et al., 2023. OS-inspired virtual context management with hierarchical memory.

12. [Memory in the Age of AI Agents: A Survey](https://github.com/Shichun-Liu/Agent-Memory-Paper-List) -- Comprehensive 2025-2026 survey of memory architectures for LLM agents.

13. [ICLR 2026 Workshop Proposal: MemAgents: Memory for LLM-Based Agentic Systems](https://openreview.net/pdf?id=U51WxL382H) -- 2026 workshop proposal on agent memory systems.

14. [A-MEM: Agentic Memory for LLM Agents](https://arxiv.org/abs/2502.12110) -- January 2026. Unified long-term and short-term memory management.

### Product Documentation

15. [OpenAI Assistants API Deep Dive](https://platform.openai.com/docs/assistants/deep-dive) -- Thread truncation strategies (auto, last_messages), token budget control.

16. [Letta/MemGPT Documentation](https://docs.letta.com/concepts/memgpt/) -- Virtual context management, Context Repositories (February 2026).

17. [How Cursor Actually Indexes Your Codebase](https://towardsdatascience.com/how-cursor-actually-indexes-your-codebase/) -- Semantic indexing, AST parsing, Merkle trees, Turbopuffer vector DB.

18. [Devin's 2025 Performance Review](https://cognition.ai/blog/devin-annual-performance-review-2025) -- Learnings from 18 months of agents at work.

### Context Engineering Guides

19. [Context Window Management: Strategies for Long-Context AI Agents](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots/) -- Comprehensive strategy overview.

20. [LLM Context Management: How to Improve Performance and Lower Costs](https://eval.16x.engineer/blog/llm-context-management-guide) -- Practical guide to context management.

21. [Memory for AI Agents: A New Paradigm of Context Engineering](https://thenewstack.io/memory-for-ai-agents-a-new-paradigm-of-context-engineering/) -- The New Stack, 2025. Dual-layer hot/cold path architecture.

---

## Summary of Key Takeaways

### The Five Commandments of Context Excellence

1. **Context is a finite attention budget, not free storage.** Every token competes for attention. Treat context like premium real estate: every item must earn its place through measured importance.

2. **Exploit the U-shaped attention curve.** Place stable, critical information at the TOP (primacy position) and recent, actionable information at the BOTTOM (recency position). The middle is for compressed history and retrieved knowledge.

3. **Prefer observation masking over summarization.** JetBrains proved that simply hiding old tool outputs is as effective as expensive LLM summarization, at half the cost and without trajectory elongation. Summarize only when masking is insufficient.

4. **Tell the model where it is.** Situational awareness (step number, progress, drift, budget) is the single highest-value, lowest-effort improvement. Without it, the model cannot self-regulate.

5. **Monitor context quality, not just token count.** Token utilization tells you HOW FULL the context is. Goal Retention Score, Signal-to-Noise Ratio, and Decision Preservation Rate tell you HOW GOOD it is. Both matter.

### corteX's Competitive Position

corteX has a **strong foundation** with its four-zone architecture, brain state injection, progressive summarization, and importance scoring. These are architecturally sound and in several cases (brain state injection, dual-process routing) represent unique differentiators not found in competitors.

The most impactful improvements are:
1. **Rich tool presentation** (currently just names, should be full descriptions with success rates)
2. **Situational awareness block** (the model does not know where it is in the task)
3. **Unified context pipeline** (two parallel engines cause maintenance overhead)
4. **Task-adaptive budgets** (one-size-fits-all is suboptimal)
5. **Context quality monitoring** (currently blind to quality degradation)

Implementing recommendations #1 and #2 alone would likely produce a measurable improvement in agent task completion rates, as they are low-effort, high-impact changes that every major competitor already implements.
