# Memory Survival Through Compaction: Research Report

**Date:** 2026-02-15
**Author:** Senior AI Engineer, corteX R&D
**Status:** Research Complete -- Implementation Pending

---

## Executive Summary

This report addresses the most critical challenge in long-running AI agent systems: **how to keep agent memory "alive" through dozens of compaction cycles, across different agents and tasks, for thousands of steps.** After analyzing corteX's current memory system, studying 15+ competing approaches (Claude Code, Manus, Letta/MemGPT, Mem0, SimpleMem, MemOS, OpenAI Agents SDK, Devin, GAM, and academic research from EMNLP 2025, NeurIPS 2025, and ICLR 2026), and synthesizing best practices from Anthropic's context engineering guide, this report presents a concrete architecture for making corteX the best-in-class memory system for enterprise AI agents.

**Key finding:** The state of the art has converged on a clear pattern -- **externalized structured state files + tiered memory with LLM-driven consolidation + append-only context with selective compaction** -- but no single system implements all pieces well. corteX has a strong foundation (3-tier memory, cold storage, progressive compression) but is missing several critical components that would make it truly compaction-proof.

---

## Table of Contents

1. [Current System Analysis](#1-current-system-analysis)
2. [Compaction Survival Strategies](#2-compaction-survival-strategies)
3. [Memory Architecture for 10,000+ Step Tasks](#3-memory-architecture-for-10000-step-tasks)
4. [Per-User Personalization Across Sessions](#4-per-user-personalization-across-sessions)
5. [Memory Prioritization](#5-memory-prioritization)
6. [Practical Patterns from Production Systems](#6-practical-patterns-from-production-systems)
7. [Gap Analysis: Current vs. Required](#7-gap-analysis-current-vs-required)
8. [Recommended Architecture](#8-recommended-architecture)
9. [Implementation Priority List](#9-implementation-priority-list)

---

## 1. Current System Analysis

### What corteX Has (Strong Foundation)

**corteX/engine/memory.py -- MemoryFabric (3-Tier)**
- Working Memory (Prefrontal Cortex): Limited capacity, fast access, volatile
- Episodic Memory (Hippocampus): Past trajectories, few-shot learning
- Semantic Memory (Neocortex): Factual knowledge, cross-session

**corteX/engine/cold_storage.py -- ColdStorage**
- Archival retrieval for evicted items
- Importance decay (0.5x) on archive
- Automatic promotion back to working memory on relevance match

**corteX/engine/context.py -- CorticalContextEngine (CCE)**
- 3-temperature hierarchy (Hot/Warm/Cold)
- Progressive compression (L0 Verbatim -> L1 Observation Masking -> L2 LLM Summary -> L3 Digest)
- ImportanceScorer with 6-factor weighted scoring
- Observation masking (JetBrains NeurIPS 2025 research)
- Checkpointing for fault-tolerant recovery
- TaskState tracking (goals, decisions, errors, entities, progress)

**corteX/engine/context_compiler.py -- ContextCompiler (4-Zone)**
- System (12%) / Persistent (8%) / Working (40%) / Recent (40%)
- 3-tier compaction thresholds (80/90/95%)
- Goal restatement at end of context (attention bias)
- Scratchpad for agent notes

**corteX/engine/context_summarizer.py -- SummarizationPipeline**
- L2 batch summarization with LLM
- L3 structured digest extraction (JSON: decisions, tools, errors, progress, questions)
- Digest merging (old + new with deduplication)
- Rate limiting with truncation fallback

**corteX/memory/persistence.py -- MemoryPersistenceManager**
- Crash-safe JSON serialization
- Atomic writes (write .tmp, then rename)
- Auto-save every N operations

### What corteX Is Missing (Critical Gaps)

| Gap | Severity | Description |
|-----|----------|-------------|
| **No Compaction-Proof State File** | CRITICAL | No MEMORY.md equivalent -- nothing survives compaction automatically |
| **No Self-Editing Memory** | CRITICAL | Agent cannot update its own memory proactively |
| **No Memory Anchors** | HIGH | No mechanism to mark information as "must survive every compaction" |
| **No Cross-Agent Memory** | HIGH | Sub-agents cannot share memories or inherit parent state |
| **No Memory Gap Detection** | HIGH | No way to detect "I forgot something critical" |
| **No User Profile Persistence** | MEDIUM | UserProfile exists in types.py but is not wired into sessions |
| **No Graph-Based Memory** | MEDIUM | Knowledge graph types exist but are unused |
| **No Audit-Compatible Memory** | MEDIUM | Memory and audit trail are separate systems |
| **No Memory Consolidation Triggers** | MEDIUM | Consolidation only runs on session end, not during long tasks |
| **Keyword-Only Search** | LOW | No embedding/semantic search for memory retrieval |

---

## 2. Compaction Survival Strategies

### 2.1 How Production Systems Survive Compaction

#### Claude Code (Anthropic)
Claude Code uses a three-layer approach:
1. **MEMORY.md files** -- Persistent markdown files loaded into the system prompt at session start (first 200 lines). These contain project-level instructions, key decisions, and standing orders.
2. **Auto Memory** -- Claude writes notes for itself during sessions, stored in `~/.claude/projects/<project>/memory/`. Background summarization means compaction is instant.
3. **Session Memory** -- Continuous background summarization of conversation state. When `/compact` fires, the pre-written summary replaces the full context.

**Key insight:** The information that must survive compaction is externalized to the filesystem, not stored in context. Compaction becomes a non-event because the critical state is always retrievable.

#### Manus AI
Manus treats the **file system as unlimited external memory**:
1. A **todo.md** file is continuously rewritten by the agent, serving as a "recitation" of objectives into the model's recent attention span.
2. All important artifacts are saved to files -- if a URL is preserved, the web page content can be dropped from context.
3. Errors are deliberately kept in context (not summarized away) because they shift the model's priors.
4. Sub-agents get **isolated context windows** -- they communicate results, not raw state.

**Key insight:** Compression must be restorable. If you can reconstruct the information from preserved references (file paths, URLs, keys), you can drop the content itself.

#### Letta/MemGPT
MemGPT implements a virtual memory system:
1. **Core Memory** (in-context) -- editable blocks for agent persona + user information
2. **Archival Memory** (out-of-context) -- vector database for long-term storage
3. **Recall Memory** (out-of-context) -- searchable conversation history
4. The agent **edits its own memory** via tool calls (`memory_insert`, `memory_replace`)
5. **Sleep-time compute** -- asynchronous memory consolidation between turns

**Key insight:** Self-editing memory is the breakthrough. The agent decides what to remember and actively manages its own state, rather than relying on passive summarization.

#### Mem0
Mem0 uses a dual-phase approach:
1. **Extraction Phase** -- LLM identifies salient information from conversation + rolling summary + recent messages
2. **Update Phase** -- Each extracted fact is compared against top-10 similar existing memories. LLM chooses: ADD / UPDATE / DELETE / NOOP
3. **Graph Memory (Mem0^g)** -- Entities as nodes, relationships as edges. Obsolete relationships marked invalid (not deleted) for temporal reasoning.

**Key insight:** Memory is not append-only. Active conflict resolution (update/delete/noop) prevents memory bloat and contradiction accumulation.

### 2.2 What MUST Survive Every Compaction

Based on analysis of all systems, the following information categories must be treated as **compaction-proof anchors**:

```
COMPACTION-PROOF STATE (Always Preserved)
=========================================
1. IDENTITY
   - User ID, preferences, communication style
   - Agent role and capabilities
   - Tenant/enterprise context

2. GOAL STATE
   - Original goal (verbatim)
   - Current sub-goals and their status
   - Progress percentage
   - Success criteria

3. DECISION LOG (Append-Only)
   - Key decisions made and rationale
   - Step number when decision was made
   - Impact on subsequent steps

4. ERROR JOURNAL (Append-Only)
   - Critical errors encountered
   - Resolution status
   - Patterns identified (to avoid repetition)

5. ACTIVE ENTITIES
   - File paths being worked on
   - API endpoints discovered
   - Variable names / schemas in use
   - External resource references (URLs, IDs)

6. TOOL STATE
   - Which tools have been used
   - Tool-specific accumulated state
   - Tools that failed (and why)

7. CONSTRAINTS
   - Rules that must not be violated
   - Enterprise policies in effect
   - User-imposed restrictions

8. OPEN QUESTIONS
   - Unresolved ambiguities
   - Information still needed
   - Blocked dependencies
```

### 2.3 Optimal Format for Compaction-Resistant State

Research and production evidence strongly favors **structured markdown with key-value sections** over pure narrative or pure JSON:

| Format | Pros | Cons | Used By |
|--------|------|------|---------|
| Structured Markdown | Human-readable, LLM-friendly, editable | Slightly verbose | Claude Code, Manus |
| JSON | Machine-parseable, precise | Hard to read in context, LLM may corrupt | Mem0, L3 digest |
| Narrative | Natural for LLM to produce | Lossy, hard to extract specifics from | OpenAI trimming |
| Key-Value | Compact, precise | Too rigid for complex state | SimpleMem |

**Recommendation:** Use structured markdown as the primary format with embedded JSON blocks for machine-critical data (like tool state). This matches what the most successful production systems use.

---

## 3. Memory Architecture for 10,000+ Step Tasks

### 3.1 The Compaction Timeline Problem

For a 10,000-step task with 128K context window:
- Average 500 tokens per step (message + tool call + result)
- Context fills at ~250 steps
- **~40 compaction cycles** needed to complete the task
- Each compaction risks losing 10-30% of nuanced context (Chroma research)
- After 40 compactions: cumulative information loss can be catastrophic

```
Step 0          Step 250        Step 500        Step 10,000
|---FILL------->|---COMPACT---->|---FILL------->|
                     |                               |
              Info Loss ~15%                 Cumulative Loss: ???
```

### 3.2 Recommended Multi-Layer Architecture

```
+================================================================+
|                    COMPACTION-PROOF LAYER                       |
|  (Externalized State Files -- NEVER in context window)          |
|                                                                |
|  state.md          decisions.md      errors.md                 |
|  - Goal             - Decision log    - Error journal          |
|  - Progress          (append-only)    - Patterns               |
|  - Sub-goals                          - Resolutions            |
|  - Entities                                                    |
|  - Constraints       user_profile.md  tool_state.md            |
|  - Open questions    - Preferences    - Tool results cache     |
|                      - Style           - Failed tools          |
|                      - Domain knowledge                        |
+================================================================+
           |                    |                    |
           v                    v                    v
+==================+  +==================+  +==================+
|  WORKING MEMORY  |  |  EPISODIC MEMORY  |  |  SEMANTIC MEMORY |
|  (In-Context)     |  |  (Persistent DB)  |  |  (Persistent DB) |
|                  |  |                    |  |                  |
|  Current task     |  |  Past trajectories |  |  Domain facts    |
|  Recent messages  |  |  Success patterns  |  |  Learned rules   |
|  Active reasoning |  |  Failure patterns  |  |  Entity knowledge|
|  Scratchpad       |  |  Similar goals     |  |  Tool best prax  |
|                  |  |                    |  |                  |
|  Capacity: 100    |  |  Capacity: 500     |  |  Capacity: 1000  |
|  TTL: session     |  |  TTL: permanent    |  |  TTL: permanent  |
+==================+  +==================+  +==================+
           |                    |                    |
           v                    v                    v
+================================================================+
|                      COLD STORAGE                               |
|  (Archival -- search + promote when relevant)                   |
|                                                                |
|  Evicted working items, old episodic memories, aged facts       |
|  Importance decayed 0.5x, promoted back on relevance match      |
|  Capacity: 2000+ (file-backed)                                  |
+================================================================+
           |
           v
+================================================================+
|                   KNOWLEDGE GRAPH                               |
|  (Entity-Relationship Index for Associative Retrieval)          |
|                                                                |
|  Entities: files, APIs, schemas, users, tools, concepts         |
|  Edges: uses, depends-on, caused-by, related-to                 |
|  Used for: memory gap detection, relevance scoring              |
+================================================================+
```

### 3.3 Memory Consolidation Strategy

**When to consolidate (not just on session end):**

| Trigger | Action | Rationale |
|---------|--------|-----------|
| Every 20 steps | L1 observation masking | Cost reduction without quality loss (JetBrains) |
| Every 50 steps | L2 summarization + state file update | Capture decisions and progress |
| Every 100 steps | L3 digest + episodic snapshot | Create "episode boundary" |
| Every 200 steps | Full consolidation cycle | Promote working -> episodic/semantic |
| On compaction trigger | Emergency state file write | Ensure nothing lost |
| On sub-goal completion | Episode recording | Natural boundary for memory chunk |
| On error resolution | Error pattern learning | Prevent repetition |
| On user feedback | Preference/correction learning | Update user profile |

### 3.4 Preventing Memory Drift

Memory drift occurs when important early context is gradually lost through successive compactions. Three mechanisms prevent this:

**1. Anchor Pinning**
Mark certain memories as `importance=1.0, pinned=True`. Pinned items are NEVER summarized or evicted. They occupy a fixed budget (e.g., 8% of context = "persistent zone" in ContextCompiler).

**2. Periodic Re-Grounding**
Every N steps, the agent re-reads its state files and verifies alignment:
- "Is my current action still serving the original goal?"
- "Have I deviated from recorded constraints?"
- "Are there open questions I should address?"

**3. Decision Chain Integrity**
Maintain a cryptographic hash chain of decisions. Each decision references the previous one. If a gap is detected (missing decision reference), the agent knows it has lost context and can retrieve from cold storage.

### 3.5 Multi-Agent Memory Sharing

Following Manus's principle: **"Share memory by communicating, don't communicate by sharing memory."**

```
+------------------+     +------------------+     +------------------+
| Orchestrator     |     | Sub-Agent A      |     | Sub-Agent B      |
| (Parent Agent)   |     | (Specialist)     |     | (Specialist)     |
|                  |     |                  |     |                  |
| Shared State:    |---->| Inherited:       |     | Inherited:       |
| - Goal           |     | - Goal slice     |<----| - Goal slice     |
| - Constraints    |     | - Constraints    |     | - Constraints    |
| - User profile   |     | - User profile   |     | - User profile   |
|                  |     |                  |     |                  |
| Receives:        |<----| Returns:         |     | Returns:         |
| - Summaries      |     | - Result summary |---->| - Result summary |
| - Decisions      |     | - Decisions made |     | - Decisions made |
| - Artifacts      |     | - Artifacts      |     | - Artifacts      |
+------------------+     +------------------+     +------------------+
         |                        |                        |
         +------------------------+------------------------+
         |           SHARED PERSISTENCE LAYER              |
         |   (State files, Episodic DB, Semantic DB)       |
         +-------------------------------------------------+
```

**Rules:**
1. Parent agent writes shared state files before spawning sub-agents
2. Sub-agents inherit read-only copies of parent state (goal, constraints, user profile)
3. Sub-agents return compressed results (1,000-2,000 tokens max)
4. Sub-agents record their own decisions to a namespaced area of the decision log
5. Parent agent merges sub-agent results into its own memory

### 3.6 Memory Gap Detection

A critical missing capability. When the agent suspects it has forgotten something:

**Detection signals:**
- Agent references an entity not in current context or state files
- Agent attempts to use a tool it previously marked as failed (without error awareness)
- Decision chain has gaps (missing step references)
- Agent asks a question it already answered (detected by decision log scan)
- Confidence score drops suddenly (calibration engine detects)

**Recovery protocol:**
1. Scan cold storage for relevant items (keyword + semantic search)
2. Scan episodic memory for similar past goals
3. Read state files for the gap area
4. If still unable to recover: **ask the user** (honest uncertainty > hallucinated continuity)

---

## 4. Per-User Personalization Across Sessions

### 4.1 What to Persist About Users

| Category | Examples | Storage | Privacy Level |
|----------|----------|---------|---------------|
| Communication preferences | Language, verbosity, formality | UserProfile | Low risk |
| Domain expertise | "Expert in Python", "Beginner in k8s" | SemanticMemory | Low risk |
| Tool preferences | "Prefers pytest over unittest" | WeightEngine | Low risk |
| Past interactions summary | "Built 3 REST APIs together" | EpisodicMemory | Medium risk |
| Feedback patterns | "Prefers concise code comments" | FeedbackEngine | Low risk |
| Organizational context | Department, role, projects | UserProfile | Medium risk |
| **NEVER PERSIST** | API keys, passwords, tokens, PII | --- | BLOCKED |

### 4.2 Enterprise Privacy Constraints

```
+-----------------------------------------------------+
|            PRIVACY-AWARE MEMORY TIERS                |
+-----------------------------------------------------+
|                                                     |
|  TIER 1: Session-Only (data_retention="none")       |
|  - Working memory cleared on session end             |
|  - No cross-session state                            |
|  - Audit log may still be retained per compliance    |
|                                                     |
|  TIER 2: User-Scoped (data_retention="session")     |
|  - Preferences persist across sessions               |
|  - Episodic memory retained per-user                 |
|  - User can request full deletion (GDPR Art. 17)     |
|  - Namespace isolation between tenants               |
|                                                     |
|  TIER 3: Enterprise-Shared (data_retention="persistent")|
|  - Semantic knowledge shared within tenant            |
|  - Episodic patterns shared (anonymized)             |
|  - Admin controls what gets shared                   |
|  - Retention policies enforced (HIPAA: 6 years)      |
|                                                     |
+-----------------------------------------------------+
```

### 4.3 How Competitors Handle Cross-Session Memory

| System | Approach | Strengths | Weaknesses |
|--------|----------|-----------|------------|
| Claude Code | MEMORY.md + Auto Memory files | Simple, transparent, user-editable | No semantic search, flat structure |
| Mem0 | Graph + vector DB per user | Rich associations, conflict resolution | Heavy infrastructure, latency |
| Letta | Core memory blocks (editable) | Agent self-manages, lightweight | Limited capacity, no enterprise scoping |
| OpenAI | Session + Long-Term Memory Notes | Clean API, well-integrated | Opaque, limited customization |
| Devin | Project context + corrections | Learns from feedback within project | Limited cross-project transfer |

**Recommendation for corteX:** Combine Claude Code's simplicity (structured files) with Mem0's intelligence (LLM-driven extraction + conflict resolution) and Letta's self-editing pattern (agent manages its own memory). Add enterprise-grade namespace isolation and GDPR compliance.

---

## 5. Memory Prioritization

### 5.1 The Importance Scoring Challenge

Not all memories are equal, and importance changes over time. A file path mentioned at step 10 may seem trivial until step 500 when the agent needs to modify that exact file.

**Current corteX scoring (ImportanceScorer in context.py):**
```
importance = w_r * recency + w_v * relevance + w_c * causal +
             w_f * ref_frequency + w_s * success_corr + w_d * domain
```

This is a good foundation but is missing two critical dimensions:

### 5.2 Enhanced Importance Model

```
importance(item, t) =
    0.15 * recency(t)              // Exponential decay with half-life
  + 0.20 * relevance(goal)         // Semantic similarity to current goal
  + 0.15 * causal_weight            // Is this a decision/error/constraint?
  + 0.10 * reference_frequency      // How often was this referenced?
  + 0.10 * success_correlation      // Does keeping this correlate with success?
  + 0.10 * domain_weight            // Domain-specific importance rules
  + 0.10 * sleeper_potential         // NEW: Could this become critical later?
  + 0.05 * connection_density        // NEW: How connected in knowledge graph?
  + 0.05 * surprise_value            // NEW: How unexpected was this information?
```

### 5.3 Sleeper Memory Handling

"Sleeper" memories are items that seem unimportant now but may become critical later. Examples:
- A file path mentioned once but relevant 500 steps later
- A constraint the user mentioned casually
- An error pattern that recurs intermittently

**Strategy for sleeper memories:**

1. **Never fully delete** -- Move to cold storage, not /dev/null
2. **Tag potential sleepers** -- Items that mention specific entities (file paths, API endpoints, schema names, user names) get a `sleeper_potential` boost
3. **Periodic cold scan** -- Every N steps, query cold storage with current context keywords. Promote matches back to working memory
4. **Surprise-based retention** -- Information that was surprising (high prediction error from PredictionEngine) should be retained longer, as surprising info is more likely to be important later (this mirrors how human hippocampus preferentially encodes surprising events)
5. **Reference chain tracking** -- If item A references item B, and item B is still in hot context, item A should not decay to cold storage

### 5.4 Academic Research on Memory Prioritization

Key papers from 2025-2026:

- **"Memory in the Age of AI Agents" (arXiv:2512.13564, Jan 2026)** -- Comprehensive survey proposing taxonomy of factual/experiential/working memory. Identifies memory formation, evolution, and retrieval as three lifecycle phases.

- **SimpleMem (arXiv:2601.02553, Jan 2026)** -- Semantic Lossless Compression achieving 26.4% F1 improvement with 30x token reduction. Three-stage pipeline: filter noise, recursive consolidation, adaptive retrieval.

- **MemR3 (arXiv:2512.20237, Dec 2025)** -- Evidence-gap state tracking (E, G) that drives query refinement. The agent maintains explicit knowledge of what it knows vs. what it is missing.

- **GAM (2025)** -- Dual-agent architecture: "memorizer" creates full records + concise memos; "recaller" retrieves on demand. Outperforms long-context LLMs on multi-session tasks.

- **MemOS (EMNLP 2025 Oral)** -- Memory Operating System unifying parameterized, activation, and plaintext memory under a MemCube abstraction with lifecycle governance.

- **ACT-R-Inspired Memory (HAI 2025)** -- Human-like remembering and forgetting using cognitive architecture. Demonstrates that controlled forgetting improves performance.

---

## 6. Practical Patterns from Production Systems

### 6.1 Pattern: State File Recitation (Manus)

The most effective pattern for surviving compaction is **continuous state file recitation**: the agent rewrites its state file at every significant step, and the state file is loaded into the persistent zone of the context window at every turn.

```
On every turn:
  1. Read state.md into persistent zone (8% of context)
  2. Execute step
  3. Update state.md with new progress/decisions/errors
  4. State.md is automatically included in next turn
```

This creates a **self-healing loop**: even after aggressive compaction, the agent can reconstruct its full understanding from the state file.

### 6.2 Pattern: Append-Only Decision Log (Claude Code)

```markdown
## decisions.md

### Step 12: Chose FastAPI over Flask
- Rationale: User needs async support, FastAPI is faster
- Impact: All subsequent routing code uses FastAPI patterns

### Step 45: Switched from SQLite to PostgreSQL
- Rationale: User mentioned need for concurrent access
- Impact: Connection pooling added, migration scripts created

### Step 230: Reverted PostgreSQL, back to SQLite
- Rationale: User clarified this is a dev-only tool
- Impact: Removed connection pooling, simplified setup
```

This log is append-only and never summarized. It provides an immutable record of decision reasoning that the agent can reference even after 40 compactions.

### 6.3 Pattern: Error Preservation (Manus + Claude Code)

Counter-intuitively, **keeping errors in context is more valuable than removing them**. When the model sees a failed action and its stack trace, it implicitly updates its beliefs and avoids repeating the mistake.

```markdown
## errors.md

### ERROR-001 (Step 15): ImportError - pandas not installed
- Resolution: Added to requirements.txt, pip install
- Pattern: Always check imports before using a library

### ERROR-002 (Step 67): 403 Forbidden on /api/admin
- Resolution: User provided admin token
- Pattern: Check auth before calling admin endpoints
- STATUS: RESOLVED

### ERROR-003 (Step 120): Test timeout on test_large_dataset
- Resolution: UNRESOLVED -- need to investigate
- Pattern: Large dataset tests need increased timeout
```

### 6.4 Pattern: Goal Sandwiching (ContextCompiler)

corteX already implements this: the goal appears in both the persistent zone (early in context) and as a restatement at the end. This exploits both **primacy bias** (early tokens are well-recalled) and **recency bias** (recent tokens guide generation).

### 6.5 Pattern: KV-Cache-Friendly Compaction

Manus discovered that **KV-cache hit rate is the single most important metric** for production AI agents. The ratio of cached to uncached tokens can be 10x in cost.

**Implications for compaction:**
- Keep the context prefix stable (system prompt, policies, persistent zone)
- Only append to working memory; never reorder
- When compacting, replace a contiguous block with a summary
- Use session IDs for distributed cache routing

### 6.6 Pattern: Enterprise Audit + Memory Alignment

For HIPAA/SOC2/GDPR compliance, the memory system and audit trail must be aligned:

```
+--------------------+     +--------------------+
|   MEMORY LAYER     |     |   AUDIT LAYER      |
+--------------------+     +--------------------+
| Working Memory     |---->| Turn-level log     |
| State Files        |---->| State change log   |
| Decision Log       |---->| Decision audit     |
| Error Journal      |---->| Error audit        |
| User Profile       |---->| Consent log        |
|                    |     |                    |
| RETENTION:         |     | RETENTION:         |
| Configurable       |     | Mandatory (6yr     |
| per enterprise     |     | HIPAA, 5yr SOC2)   |
+--------------------+     +--------------------+
```

The audit layer is an immutable superset of the memory layer. Memory can be deleted (GDPR right to erasure), but the audit log must note that deletion occurred.

---

## 7. Gap Analysis: Current vs. Required

### Priority 1: CRITICAL (Must have for compaction survival)

| # | Gap | Current State | Required State | Effort |
|---|-----|--------------|----------------|--------|
| 1 | **Compaction-Proof State File** | No externalized state | Agent maintains state.md auto-loaded into persistent zone | Medium |
| 2 | **Self-Editing Memory Tools** | Agent cannot modify memory | Agent has `memory_write`, `memory_update`, `memory_delete` tools | Medium |
| 3 | **Decision Log** | Tracked in TaskState but lost on compaction | Append-only file, never summarized, always in persistent zone | Small |
| 4 | **Error Journal** | Errors tracked in L3 digest only | Append-only file with pattern learning | Small |
| 5 | **Consolidation During Long Tasks** | Only on session end | Triggered every 50-200 steps + on sub-goal completion | Medium |

### Priority 2: HIGH (Required for 1000+ step tasks)

| # | Gap | Current State | Required State | Effort |
|---|-----|--------------|----------------|--------|
| 6 | **Memory Gap Detection** | None | Evidence-gap tracking (inspired by MemR3) | Medium |
| 7 | **Cross-Agent Memory Inheritance** | None | Parent state -> child read-only copy, child returns summary | Medium |
| 8 | **Sleeper Memory Scoring** | Not tracked | `sleeper_potential` factor in ImportanceScorer | Small |
| 9 | **Memory Conflict Resolution** | None | ADD/UPDATE/DELETE/NOOP pattern (Mem0-style) | Medium |
| 10 | **KV-Cache-Aware Compaction** | Naive compaction | Append-only context, prefix-stable, cache breakpoints | Medium |

### Priority 3: MEDIUM (Required for enterprise-grade)

| # | Gap | Current State | Required State | Effort |
|---|-----|--------------|----------------|--------|
| 11 | **User Profile Persistence** | UserProfile type exists, unused | Wired into session, loaded from persistence | Small |
| 12 | **Privacy-Tier Memory** | Single mode | 3 tiers: session-only / user-scoped / enterprise-shared | Medium |
| 13 | **Audit-Memory Alignment** | Separate systems | Audit layer mirrors memory operations | Medium |
| 14 | **Knowledge Graph Activation** | EntityNode exists, unused | Active entity-relationship tracking for associative retrieval | Large |
| 15 | **Semantic Search for Memory** | Keyword-only | Embedding-based similarity search (on-prem compatible) | Large |

### Priority 4: FUTURE (Competitive differentiation)

| # | Gap | Current State | Required State | Effort |
|---|-----|--------------|----------------|--------|
| 16 | **Sleep-Time Compute** | None | Async consolidation between turns (Letta pattern) | Large |
| 17 | **Memory Evolution (MemRL)** | None | RL-based memory optimization from episodic outcomes | Very Large |
| 18 | **MemCube Abstraction** | None | Unified memory type spanning in-context + persistent + weights | Very Large |

---

## 8. Recommended Architecture

### 8.1 The "Cortical Memory Stack" -- Target Architecture

```
+=================================================================+
|                    CONTEXT WINDOW (LLM Input)                    |
+=================================================================+
|                                                                 |
|  SYSTEM ZONE (12%)                                               |
|  +----------------------------------------------------------+  |
|  | System Prompt (stable, cached)                             |  |
|  | Policies + Constraints                                     |  |
|  | Tool Definitions                                           |  |
|  +----------------------------------------------------------+  |
|                                                                 |
|  PERSISTENT ZONE (8%) -- COMPACTION-PROOF                        |
|  +----------------------------------------------------------+  |
|  | ## Goal: [original goal, verbatim]                         |  |
|  | ## Progress: [X% complete, current sub-goal]               |  |
|  | ## Key Decisions: [last 5 from decision log]               |  |
|  | ## Active Errors: [unresolved from error journal]          |  |
|  | ## Active Entities: [files, APIs, schemas in play]         |  |
|  | ## User: [preferences, expertise, communication style]     |  |
|  | ## Brain State: [confidence, weights, calibration]         |  |
|  | ## Open Questions: [what we still need to know]            |  |
|  +----------------------------------------------------------+  |
|                                                                 |
|  WORKING ZONE (40%)                                              |
|  +----------------------------------------------------------+  |
|  | Scratchpad notes (agent-writable)                          |  |
|  | Summarized older conversation (L2 summaries)               |  |
|  | Working memory items (current session)                     |  |
|  +----------------------------------------------------------+  |
|                                                                 |
|  RECENT ZONE (40%)                                               |
|  +----------------------------------------------------------+  |
|  | Last N tool calls + results (raw, unccompressed)           |  |
|  | Last N messages (user + assistant)                         |  |
|  | Current task state                                         |  |
|  | [REMINDER] Your current goal: [goal restatement]           |  |
|  +----------------------------------------------------------+  |
|                                                                 |
+=================================================================+

+=================================================================+
|                 EXTERNALIZED PERSISTENCE LAYER                   |
|              (Filesystem + Database -- SURVIVES ALL)             |
+=================================================================+
|                                                                 |
|  STATE FILES (Markdown, auto-read/write by agent)                |
|  +------------------+  +------------------+  +--------------+  |
|  | state.md         |  | decisions.md     |  | errors.md    |  |
|  | - Goal           |  | - Append-only    |  | - Append-only|  |
|  | - Progress       |  | - Step + Reason  |  | - Pattern    |  |
|  | - Sub-goals      |  | - Impact note    |  | - Status     |  |
|  | - Entities       |  |                  |  | - Resolution |  |
|  | - Constraints    |  |                  |  |              |  |
|  +------------------+  +------------------+  +--------------+  |
|                                                                 |
|  MEMORY FABRIC (3-Tier + Cold Storage)                           |
|  +------------------+  +------------------+  +--------------+  |
|  | Working Memory   |  | Episodic Memory  |  | Semantic Mem |  |
|  | 100 items        |  | 500 trajectories |  | 1000 facts   |  |
|  | Session-scoped   |  | Permanent        |  | Permanent    |  |
|  +------------------+  +------------------+  +--------------+  |
|           |                                                     |
|           v                                                     |
|  +----------------------------------------------------------+  |
|  | Cold Storage (2000+ items, file-backed)                    |  |
|  | + Knowledge Graph (entity-relationship index)              |  |
|  +----------------------------------------------------------+  |
|                                                                 |
|  USER PROFILES (Per-user, per-tenant)                            |
|  +----------------------------------------------------------+  |
|  | user_profile.json: preferences, expertise, feedback hist   |  |
|  | Namespace isolated per tenant                              |  |
|  | GDPR-compliant: exportable + deletable                     |  |
|  +----------------------------------------------------------+  |
|                                                                 |
|  AUDIT LOG (Immutable, compliance-grade)                         |
|  +----------------------------------------------------------+  |
|  | Every memory write/delete logged with timestamp            |  |
|  | Retention: configurable per compliance regime              |  |
|  | Mirrors memory operations for full traceability            |  |
|  +----------------------------------------------------------+  |
|                                                                 |
+=================================================================+
```

### 8.2 Compaction Flow (After Implementation)

```
Step 1-249: Normal operation
  - Agent reads state.md at each turn (persistent zone)
  - Agent writes scratchpad notes as needed
  - L1 masking runs every 20 steps
  - State.md updated every 50 steps

Step 250: Context 80% full -> LIGHT COMPACTION
  1. Write current state to state.md
  2. Append any new decisions to decisions.md
  3. L2 summarize oldest 60% of working zone
  4. Keep recent 40% raw (last ~100 steps)
  5. Load state.md into persistent zone
  Result: Context drops to ~45% utilization

Step 250-490: Normal operation continues
  - Agent operates with summarized history + raw recent + state files
  - No information lost because state files have everything critical

Step 490: Context 80% full -> LIGHT COMPACTION (2nd cycle)
  1. Update state.md with progress since step 250
  2. L2 summarize the L2 summary from step 250 + new steps
  3. Create L3 digest from accumulated L2 summaries
  4. Episodic snapshot: record steps 250-490 as an episode
  Result: Context drops to ~45%, state files have complete history

... (repeat for 40 cycles to reach step 10,000)

At ANY point, if agent is confused:
  1. Read state.md -> full goal, progress, decisions, errors
  2. Read decisions.md -> all decisions ever made (append-only)
  3. Search cold storage for relevant dormant memories
  4. Query episodic memory for similar past experiences
  5. If still stuck: ask the user (honest uncertainty)
```

### 8.3 Self-Editing Memory Tools

New tools the agent gains access to:

```python
# These are tools the AGENT calls to manage its own memory

@tool
async def memory_write(key: str, value: str, importance: float = 0.5,
                       memory_type: str = "working") -> str:
    """Write a memory item to the specified memory tier.
    Use for important facts, decisions, or observations."""

@tool
async def memory_update(key: str, new_value: str) -> str:
    """Update an existing memory item.
    Use when information has changed."""

@tool
async def memory_search(query: str, memory_type: str = "all") -> str:
    """Search memories across all tiers.
    Use when you need to recall something."""

@tool
async def state_update(section: str, content: str) -> str:
    """Update a section of the persistent state file.
    Sections: goal, progress, entities, constraints, questions."""

@tool
async def decision_record(decision: str, rationale: str) -> str:
    """Record a key decision to the append-only decision log.
    Use for every significant choice."""

@tool
async def error_record(error: str, resolution: str = "",
                       pattern: str = "") -> str:
    """Record an error to the error journal.
    Use for every significant error encountered."""
```

---

## 9. Implementation Priority List

### Phase 1: Compaction-Proof Foundation (1-2 weeks)

1. **Create `StateFileManager`** -- Read/write state.md, decisions.md, errors.md
   - Location: `corteX/engine/state_files.py`
   - Auto-loads state.md into ContextCompiler's persistent zone
   - Append-only writes for decisions and errors
   - Integrates with MemoryPersistenceManager

2. **Wire state files into ContextCompiler** -- Persistent zone always includes state
   - Modify `ContextCompiler._build_persistent_content()` to read from state files
   - Modify compaction flow to update state files before compacting

3. **Add memory self-editing tools** -- `memory_write`, `memory_search`, `state_update`, `decision_record`, `error_record`
   - Location: `corteX/tools/memory_tools.py`
   - Register as built-in tools in ToolExecutor

4. **Periodic consolidation triggers** -- Not just on session end
   - Modify `Session.run()` to call consolidation every 50-200 steps
   - Add sub-goal completion as a consolidation trigger

### Phase 2: Intelligent Memory Management (2-3 weeks)

5. **Enhanced ImportanceScorer** -- Add sleeper_potential, connection_density, surprise_value
   - Modify `corteX/engine/context.py` ImportanceScorer
   - Integrate with PredictionEngine for surprise signal

6. **Memory conflict resolution** -- ADD/UPDATE/DELETE/NOOP pattern
   - New class `MemoryConflictResolver` in `corteX/engine/memory_resolver.py`
   - LLM-powered comparison of new facts vs. existing memories

7. **Memory gap detection** -- Evidence-gap state tracking
   - New class `MemoryGapDetector` in `corteX/engine/memory_gap.py`
   - Monitors for: entity references without context, repeated questions, decision chain gaps
   - Triggers cold storage retrieval + user question on gap detection

8. **Cross-agent memory inheritance** -- Parent -> child state passing
   - Modify SubAgentManager to pass state file snapshots
   - Sub-agents return compressed results that parent merges

### Phase 3: Enterprise & Personalization (2-3 weeks)

9. **User profile persistence** -- Wire UserProfile into sessions
   - Load from disk on session start
   - Save on session end
   - Include in persistent zone of context

10. **Privacy-tier memory** -- 3 tiers based on EnterpriseConfig.data_retention
    - session-only: clear on session end
    - user-scoped: persist per user, GDPR-deletable
    - enterprise-shared: shared within tenant, admin-controlled

11. **Audit-memory alignment** -- Mirror memory writes to audit log
    - Every `memory_write/update/delete` generates an audit entry
    - Immutable audit log with configurable retention

12. **KV-cache optimization** -- Prefix-stable context, append-only working zone
    - Modify ContextCompiler to track cache breakpoints
    - Add session ID routing for distributed cache

### Phase 4: Advanced Capabilities (3-4 weeks)

13. **Knowledge graph activation** -- Use EntityNode/GraphSlice from types.py
    - Entity extraction from tool results and decisions
    - Relationship tracking (uses, depends-on, caused-by)
    - Graph-based memory retrieval (spreading activation)

14. **Embedding-based memory search** -- On-prem vector similarity
    - Local embedding model (e.g., sentence-transformers)
    - Replace keyword search in InMemoryBackend with hybrid search
    - No cloud dependency (stays on-prem)

15. **Sleep-time compute** -- Async consolidation between turns
    - Background thread for memory consolidation
    - Runs during user think-time
    - Updates state files, promotes working -> long-term

---

## Sources

### Research Papers
- [Memory in the Age of AI Agents (arXiv:2512.13564)](https://arxiv.org/abs/2512.13564)
- [Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory (arXiv:2504.19413)](https://arxiv.org/abs/2504.19413)
- [SimpleMem: Efficient Lifelong Memory for LLM Agents (arXiv:2601.02553)](https://arxiv.org/abs/2601.02553)
- [MemOS: A Memory OS for AI System (arXiv:2507.03724)](https://huggingface.co/papers/2507.03724)
- [Memory OS of AI Agent (EMNLP 2025)](https://aclanthology.org/2025.emnlp-main.1318.pdf)
- [MemR3: Memory Retrieval via Reflective Reasoning for LLM Agents (arXiv:2512.20237)](https://arxiv.org/pdf/2512.20237)
- [ICLR 2026 Workshop: MemAgents](https://openreview.net/pdf?id=U51WxL382H)
- [Context Rot: How Increasing Input Tokens Impacts LLM Performance (Chroma Research)](https://research.trychroma.com/context-rot)

### Production Systems
- [Anthropic: Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Manus: Context Engineering for AI Agents](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- [Letta/MemGPT: Agent Memory](https://www.letta.com/blog/agent-memory)
- [Letta: Context Engineering](https://docs.letta.com/guides/agents/context-engineering/)
- [Claude Code: Memory Management](https://code.claude.com/docs/en/memory)
- [Claude Code Compaction](https://stevekinney.com/courses/ai-development/claude-code-compaction)
- [OpenAI Agents SDK: Session Memory](https://cookbook.openai.com/examples/agents_sdk/session_memory)
- [Devin AI Performance Review 2025](https://cognition.ai/blog/devin-annual-performance-review-2025)
- [Redis: AI Agent Memory](https://redis.io/blog/ai-agent-memory-stateful-systems/)
- [AWS AgentCore Long-Term Memory](https://aws.amazon.com/blogs/machine-learning/building-smarter-ai-agents-agentcore-long-term-memory-deep-dive/)

### Enterprise & Privacy
- [AI Agents and Memory: Privacy in the MCP Era](https://www.newamerica.org/oti/briefs/ai-agents-and-memory/)
- [Engineering GDPR Compliance in the Age of Agentic AI (IAPP)](https://iapp.org/news/a/engineering-gdpr-compliance-in-the-age-of-agentic-ai)
- [Agent Compliance Layer](https://www.agentcompliancelayer.com/)
- [GAM: Dual-Agent Memory Architecture (VentureBeat)](https://venturebeat.com/ai/gam-takes-aim-at-context-rot-a-dual-agent-memory-architecture-that)

---

## Appendix A: Comparison Matrix

| Feature | corteX (Current) | Claude Code | Manus | Letta/MemGPT | Mem0 | SimpleMem |
|---------|------------------|-------------|-------|-------------|------|-----------|
| 3-Tier Memory | Yes | No | No | Yes (2-tier) | No | No |
| Cold Storage | Yes | No | File system | Archival memory | No | No |
| Progressive Compression | L0-L3 | Summarize | Summarize+files | Sliding window | No | Semantic |
| Self-Editing Memory | No | Auto Memory | File I/O | Tool-based | API-based | No |
| State File Survival | No | MEMORY.md | todo.md + files | Core memory | Rolling summary | Cross-session |
| Decision Log | In TaskState | Not explicit | todo.md | Not explicit | Graph edges | Not explicit |
| Error Preservation | L3 digest | In context | In context | Not explicit | Not explicit | Not explicit |
| Cross-Agent Memory | No | No | Sub-agent isolation | Multi-agent API | No | No |
| Memory Gap Detection | No | No | No | No | No | No |
| User Profile Persistence | Type only | MEMORY.md | No | Core memory | User-scoped | No |
| Knowledge Graph | Types only | No | No | No | Mem0^g | No |
| Enterprise Audit | Separate | No | No | No | No | No |
| On-Prem Capable | Yes | No (cloud) | No (cloud) | Yes | Optional | Yes |
| Importance Scoring | 6-factor | No | No | No | Semantic sim | Density gate |
| Observation Masking | Yes (L1) | No | No | No | No | Yes |

**Key takeaway:** corteX has the strongest *engine-level* foundation (importance scoring, progressive compression, observation masking, brain-inspired design) but is missing the *operational patterns* (state files, self-editing, decision logs) that make systems like Claude Code and Manus effective in practice.

---

## Appendix B: Glossary

| Term | Definition |
|------|-----------|
| **Compaction** | Summarizing conversation history to fit within context window limits |
| **Context Rot** | Gradual degradation of LLM accuracy as context grows (Chroma 2025) |
| **KV-Cache** | Key-Value cache in transformer attention; reusing it saves 10x cost |
| **Observation Masking** | Replacing tool outputs with placeholders (JetBrains NeurIPS 2025) |
| **Memory Drift** | Gradual loss of early context through successive compactions |
| **Sleeper Memory** | Information that seems unimportant now but may be critical later |
| **State File** | Externalized structured file that always survives compaction |
| **Evidence-Gap State** | Explicit tracking of what the agent knows vs. what it is missing |
| **Memory Consolidation** | Promoting working memory to long-term storage |
| **Self-Editing Memory** | Agent's ability to create/update/delete its own memories via tools |
