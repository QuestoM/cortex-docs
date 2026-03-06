# Deep Research: State-of-the-Art Agentic Systems

## Comprehensive Analysis for corteX Agentic Engine

**Date:** 2026-02-14
**Scope:** Architecture analysis of leading agentic systems, extraction of common patterns, and actionable recommendations for corteX Agentic Engine.
**Focus Areas:** Context window management, LLM call orchestration, goal verification, client interaction, long-running tasks, autonomous decision-making.

---

## Table of Contents

1. [Claude Code Analysis](#1-claude-code-analysis)
2. [OpenClaw Analysis](#2-openclaw-analysis)
3. [CUGA Analysis](#3-cuga-analysis)
4. [Cursor Analysis](#4-cursor-analysis)
5. [Other Systems Analysis](#5-other-systems-analysis)
6. [Common Patterns Across All Systems](#6-common-patterns-across-all-systems)
7. [Key Insights for corteX](#7-key-insights-for-cortex)
8. [Recommended Architecture for corteX Agentic Engine](#8-recommended-architecture-for-cortex-agentic-engine)

---

## 1. Claude Code Analysis

### 1.1 Core Architecture: The Master Loop (nO)

Claude Code's architecture is centered on a **single-threaded master loop** (internally codenamed "nO"). The design philosophy is radical simplicity: `while(tool_call) -> execute tool -> feed results -> repeat`. The loop terminates when Claude produces plain text without tool calls.

**Key architectural decisions:**

- **Single-threaded, not multi-agent.** One flat list of messages. No competing agent personas, no swarm logic. This makes the system debuggable, predictable, and auditable.
- **Sub-agent spawning is strictly limited** to one at a time via the `dispatch_agent` tool (I2A/Task Agent), preventing recursive explosion while enabling parallel exploration.
- **The harness IS the agent.** Claude Code serves as the "agentic harness" -- it provides the tools, context management, and execution environment that turn a language model into a capable agent. The LLM does the reasoning; the harness provides the agency.

### 1.2 The Agentic Loop: Three Phases

Claude Code's loop blends three phases that are NOT strictly sequential but intermixed based on task needs:

1. **Gather Context** -- Search files, read code, explore codebase structure
2. **Take Action** -- Edit files, run commands, create artifacts
3. **Verify Results** -- Run tests, check outputs, validate changes

The loop adapts: a question about a codebase might only need context gathering. A bug fix cycles through all three phases repeatedly. A refactor might involve extensive verification. Claude decides what each step requires based on what it learned from the previous step, chaining dozens of actions together and course-correcting along the way.

### 1.3 Real-Time Steering (h2A Queue)

An asynchronous dual-buffer queue enables **mid-task user interjection** without full restart. Users can inject constraints or redirect Claude's approach dynamically. The system seamlessly adjusts plans mid-execution, creating truly interactive streaming conversations rather than batch processing.

**This is a critical pattern.** The agent doesn't wait for the user to respond at defined checkpoints -- the user can interrupt the agent at ANY point during execution.

### 1.4 Context Window Management

**Compaction (wU2):**
- Triggers automatically at ~92-95% context window usage
- Summarizes conversation history while preserving key information
- Uses the same model for summarization (no separate summarizer)
- The compaction block replaces all prior content -- everything before it is dropped
- Custom instructions can control what gets preserved during compaction
- CLAUDE.md files serve as persistent memory that survives compaction

**API-Level Compaction (New, 2026):**
- `context_management.edits` parameter with `compact_20260112` strategy
- Configurable trigger threshold (minimum 50,000 tokens, default 150,000)
- `pause_after_compaction` option lets the harness inject additional content before continuing
- Compaction blocks can be cached with `cache_control` for cost optimization
- Default summarization prompt: "Write a summary... including the state, next steps, learnings. You must wrap your summary in `<summary></summary>` tags."

**What survives compaction:**
- System prompt (always present)
- CLAUDE.md instructions (injected every turn)
- The compaction summary itself
- Recent messages (if using pause_after_compaction to preserve them)

**What is lost:**
- Detailed conversation history
- Early instructions not in CLAUDE.md
- Tool output details

### 1.5 Planning: TodoWrite

Claude Code uses a simple but effective planning mechanism:
- **TodoWrite** creates structured JSON task lists with IDs, content, status, and priority levels
- The UI renders these as interactive checklists
- System reminders inject the current TODO state after tool uses, preventing model drift
- This is essentially a "scratchpad" that keeps the plan in the model's recent attention

### 1.6 Tool Architecture

Tools follow a consistent JSON-call-to-plaintext-result pattern:

| Category | Tools | Design Philosophy |
|----------|-------|-------------------|
| Reading | View, LS, Glob | View defaults to ~2000 lines |
| Search | GrepTool | Regex-powered (ripgrep); chosen OVER vector search |
| Editing | Edit, Write/Replace | Diff-based, tracked and reviewable |
| Execution | Bash | Persistent sessions; risk classification; injection filtering |
| Specialized | WebFetch, NotebookEdit | URL restrictions; JSON parsing |

**Critical design choice:** Claude Code chose regex/ripgrep search over embeddings/vector search. The reasoning: simpler, more transparent, no index to maintain, no stale data, and the LLM is smart enough to construct good search queries.

### 1.7 Long-Running Task Architecture

Anthropic's published approach for long-running agents uses a **two-part solution**:

1. **Initializer Agent** (first session): Sets up the environment, creates `init.sh`, creates `claude-progress.txt` for sequential logging, makes initial git commit, generates a comprehensive feature list as JSON (200+ requirements, all initially "failing")

2. **Coding Agent** (subsequent sessions): Each session makes incremental progress, works on single features sequentially, commits changes to git, updates progress files before session end, leaves code in "main-branch-ready" condition

**State management across sessions:**
- Run `pwd` to establish working directory
- Read git logs and progress files to understand recent work
- Identify highest-priority incomplete features
- Execute basic end-to-end tests before starting new work

**Key insight:** JSON format for feature lists is more resistant to model drift than Markdown. Success depends on **externalized memory artifacts** (progress files, git history) that remain consistent across context window transitions.

### 1.8 Sub-Agents

Sub-agents get their own fresh context window, completely separate from the main conversation. Their work doesn't bloat the main context. When done, they return a summary. This isolation is what makes sub-agents valuable for long sessions.

### 1.9 Safety Model

- Permission system requires explicit allow/deny for writes, risky Bash commands, and external tools
- Command sanitization includes risk classification and safety notes
- Checkpoints before every file edit (snapshots for revert)
- Diffs-first workflow for transparency

### 1.10 Key Takeaways from Claude Code

| Principle | Implementation |
|-----------|---------------|
| Simplicity over complexity | Single loop, flat message list, no swarms |
| Tools over knowledge | Grep over embeddings, agentic search over pre-loaded context |
| Externalized memory | CLAUDE.md, git, progress files survive context resets |
| Compaction over expansion | Summarize and compress, don't try to keep everything |
| User-in-the-loop | Interruptible at any point, not just checkpoints |
| Verify by doing | Run tests, check outputs, don't just reason about correctness |

---

## 2. OpenClaw Analysis

### 2.1 Core Architecture: Hub-and-Spoke Gateway

OpenClaw is an open-source (MIT) personal AI assistant platform that uses a **local-first Gateway WebSocket control plane** as the single hub for sessions, channels, tools, and events.

**Architecture layers:**
- **Gateway** (`ws://127.0.0.1:18789`): Node.js 22+ WebSocket server, single source of truth
- **Agent Runtime** (`piembeddedrunner.ts`): RPC mode with tool streaming and block streaming
- **Channel Adapters**: WhatsApp (Baileys), Telegram (grammY), Discord (discord.js), etc.
- **Tool System**: First-class tools including browser (CDP), Canvas, cron, webhooks

### 2.2 Agent Loop: Four-Phase Cycle

1. **Session Resolution**: Determines which session handles the message (main, DM, or group)
2. **Context Assembly**: Loads conversation history, builds system prompts, retrieves relevant memories
3. **Model Invocation**: Streams requests to configured LLM providers with failover
4. **Tool Execution & State Persistence**: Intercepts tool calls during streaming, executes them (potentially sandboxed), streams results back to model

The runtime **watches for tool calls during streaming** and intercepts them, incorporating results into ongoing generation before persisting updated session state.

### 2.3 Context Management: Layered Composition

Context is assembled through explicit layering:
- **Workspace configuration files**: `AGENTS.md` (agent instructions), `SOUL.md` (personality), `TOOLS.md` (tool conventions)
- **Session history**: Persisted JSON files
- **Memory search**: SQLite with vector embeddings for semantic retrieval
- **Skills**: `skills/<skill>/SKILL.md` injected selectively to avoid prompt bloat
- **Tool definitions**: Auto-generated from built-in and plugin sources

### 2.4 Memory System: Hybrid Search

OpenClaw's memory combines **vector similarity** (semantic matching) with **BM25 keyword relevance**:
- Storage: `~/.openclaw/memory/<agentId>.sqlite` using SQLite
- Memory files: `MEMORY.md` (long-term facts) and `memory/YYYY-MM-DD.md` (daily logs)
- Embedding provider selection: Local models first, fallback to OpenAI/Gemini APIs
- Auto-reindexing with file watcher (1.5s debounce)

### 2.5 Multi-Agent Communication

Agents communicate through the Gateway's session abstraction:
- `sessions_list`: Discover active sessions/agents
- `sessions_history`: Fetch session transcripts
- `sessions_send`: Message other sessions with optional reply-back

### 2.6 Security: Layered Sandboxing

- **Main session**: Tools execute natively on host
- **DM/Group sessions**: Tools run in ephemeral Docker containers by default
- **Policy precedence**: Tool Profile > Provider Profile > Global Policy > Provider Policy > Agent Policy > Group Policy > Sandbox Policy

### 2.7 Key Takeaways from OpenClaw

| Principle | Implementation |
|-----------|---------------|
| Local-first | Everything runs on your machine, no cloud dependency |
| Layered context | Configuration files, not monolithic prompts |
| Hybrid memory | Vector + BM25 keyword search |
| Channel-agnostic | Same agent serves WhatsApp, Telegram, Discord, etc. |
| Sandboxed by default | Docker isolation for untrusted sessions |
| File-based personality | SOUL.md, AGENTS.md -- behavior via config, not code |

---

## 3. CUGA Analysis

### 3.1 Core Architecture: Enterprise Generalist Agent

CUGA (by IBM Research) is an open-source enterprise generalist agent with a **modular, multi-layer, multi-agent system** designed for complex, long-horizon tasks across web and API environments. Released under Apache 2.0.

### 3.2 Agent Architecture: Plan Controller + Plan-Execute Agents

**Central component: Plan Controller Agent**
- Decomposes user intents into structured sub-tasks
- Tracks execution states through a **dynamic task ledger**
- Supports re-planning when tasks fail or conditions change

**Specialized Plan-Execute Agents:**
- **Browser Agents**: UI interactions via Playwright automation
- **API Agents**: REST, OpenAPI, MCP server calls
- **Custom Agents**: Domain-specific executors

Each agent is equipped with:
- Short-term memory
- Reflection mechanisms
- Variable management (prevents hallucination)

### 3.3 Task Decomposition Flow

```
User Message -> Chat Layer (intent interpretation) -> Goal Construction
     -> Task Planning & Control (structured subtask decomposition)
     -> Dynamic Task Ledger (programmatic tracking)
     -> Plan-Execute Agents (browser/API/custom)
     -> Re-planning if needed
```

### 3.4 Configurable Reasoning Modes

CUGA supports **multiple reasoning modes** that balance performance vs. cost/latency:
- Fast heuristics (cheap, fast)
- Deep planning (expensive, thorough)
- Hybrid approaches

### 3.5 Policy System (5 Types)

A distinguishing enterprise feature:

| Policy Type | Purpose |
|-------------|---------|
| Intent Guard | Blocks unauthorized actions |
| Playbook | Standardizes workflows |
| Tool Approval | Enables human oversight gates |
| Tool Guide | Provides domain knowledge |
| Output Formatter | Controls response generation |

### 3.6 Save-and-Reuse (Experimental)

CUGA can **capture and reuse successful execution paths** (plans, code, trajectories) for faster and consistent behavior in recurring tasks. This is essentially "learning from experience" at the trajectory level.

### 3.7 Key Takeaways from CUGA

| Principle | Implementation |
|-----------|---------------|
| Enterprise-first | Policy system with 5 guardrail types |
| Hybrid execution | Browser + API in single workflows |
| Dynamic re-planning | Task ledger supports mid-execution replanning |
| Composability | Agent can be exposed as a tool to other agents |
| Variable management | Structured variable tracking prevents hallucination |
| Trajectory reuse | Learn from successful past executions |

---

## 4. Cursor Analysis

### 4.1 Core Architecture: Agent-First IDE

Cursor is a VS Code fork that treats AI as a core architectural component, not a plugin. Because it owns the entire editor state, it can open files, run terminal commands, apply multi-file diffs, and manage browser interactions with lower latency than a plugin working through an API layer.

### 4.2 Context Management: Hierarchical RAG

Cursor's primary competitive advantage is its **sophisticated context handling**:

**Semantic Chunking Process:**
- Tree-sitter for language-agnostic parsing
- Files split into semantic chunks (100-500 tokens)
- Overlapping windows maintain context
- 1536-dimensional embedding vectors
- Similarity search: `vector_db.similarity_search(query_embedding, top_k=20)`

**Hierarchical Context Compression:**

| Level | Content | Token Budget |
|-------|---------|-------------|
| 1 | Current file | ~2K |
| 2 | Imported files | ~3K |
| 3 | Related files (by embedding) | ~2K |
| 4 | Project structure | ~1K |

**Compression Pipeline:** Full codebase (10M tokens) -> Find relevant files -> Top 100 files (500K) -> Score by relevance -> Top 20 files (50K) -> Final context (~8K tokens)

### 4.3 Agent Mode: Autonomous Execution Loop

Cursor's Agent (`Cmd+.`) executes multi-step tasks through:

1. **Intent Classification**: Understands the goal
2. **Task Planning**: Breaks into executable steps
3. **Tool Selection**: Chooses appropriate actions
4. **Execution Loop**: Runs commands, reads/writes files, searches codebase
5. **Verification**: Checks goal achievement
6. **Self-Correction**: Fixes issues independently

### 4.4 Multi-Model Routing

**Fast Path (Autocomplete):**
- Custom fine-tuned models (1-7B parameters)
- Quantized INT8 on A100/H100 GPUs
- Target: <100ms p50 latency
- Dynamic batching for throughput

**Slow Path (Complex Tasks):**
- Routes to GPT-4, Claude Opus 4.1, Gemini 2.5 Pro
- Load balancing across providers
- Fallback mechanisms for failures
- Target: 2-5s for first token

### 4.5 @-Mention Context Control

Users explicitly guide context inclusion:
- `@Files` -- specific files
- `@Folders` -- entire directories
- `@Code` -- specific functions/blocks
- `@Docs` -- external documentation
- `@Web` -- real-time web search
- `@Definitions` -- jump to symbol definitions

### 4.6 Persistent Knowledge

- **Memories**: Remember facts from conversations, apply in future sessions
- **Rules** (`.cursorrules`): Pin instructions to project/user/team -- naming conventions, patterns, testing standards, security policies
- **RLHF Loop**: Collects acceptance rates of suggestions to improve quality over time

### 4.7 Key Takeaways from Cursor

| Principle | Implementation |
|-----------|---------------|
| Hierarchical RAG | 10M tokens -> 8K tokens through progressive filtering |
| Dual-model routing | Fast models for autocomplete, frontier for complex tasks |
| User-guided context | @-mentions give explicit control over what's in context |
| Embedding-based search | Tree-sitter + vector DB for semantic code understanding |
| Persistent rules | .cursorrules files survive across sessions |
| IDE integration | Owning the editor gives lower latency than plugin approach |

---

## 5. Other Systems Analysis

### 5.1 Devin (Cognition AI)

**Architecture:**
- Full cloud-based IDE per agent instance (shell, code editor, browser in sandbox)
- Multi-agent operation: one AI agent can dispatch tasks to other AI agents
- Self-assessed confidence evaluation: asks for clarification when not confident enough
- Automatic repository indexing every couple hours (creates architecture diagrams, wikis)

**Task Execution Pattern:**
```
Plan -> Implement chunk -> Test -> Fix -> Checkpoint review -> Next chunk
```

**Key innovations:**
- Structured checkpoints for multi-part tasks
- Fresh-start pattern: if struggling with feedback, start over rather than iterate
- Confidence-based escalation to humans
- Auto-generated repository wikis for context

### 5.2 SWE-Agent

**Architecture:**
- **DefaultAgent**: Minimal control loop (~100 lines of Python) coordinating Model and Environment
- **SWEEnv**: Thin wrapper around Docker containers with shell sessions
- **HistoryProcessor**: Compresses conversation history for optimal context window usage
- **ACI (Agent-Computer Interface)**: Custom tools configured via YAML

**Key Pattern -- ReAct Loop:**
```python
while not done:
    action = agent.forward()  # Model decides action
    observation = env.execute(action)  # Environment executes
    agent.history.append(observation)  # History updated
    done = check_completion(observation)
```

**Live-SWE-Agent (2025 evolution):**
- Dynamically evolves its own implementation at runtime
- Starts from minimal shell tools
- Can **invent new tools** during problem-solving and immediately use them
- Autonomously modifies its own agent structure

### 5.3 Open SWE (LangChain)

**Multi-agent architecture with 4 dedicated agents:**
1. **Planner**: Researches codebase to form robust strategy first
2. **Coder**: Implements changes
3. **Reviewer**: Checks for common errors, runs tests and formatters
4. **Orchestrator**: Coordinates the pipeline

Built on LangGraph, each agent has its own state and unique inputs/outputs.

### 5.4 AutoGPT (2025-2026)

**Recursive goal-pursuit loop:**
1. Receive goal
2. Generate task list
3. Execute top task
4. Evaluate result
5. Update task list
6. Repeat

**Improvements in 2025-2026:**
- Step limits and human-in-the-loop feedback to prevent API spirals
- Visual builders for agent configuration
- Multimodal support (image analysis, complex coding)

### 5.5 BabyAGI

**Cognitive sequencing loop:**
1. **Task Execution**: Complete current task
2. **Task Creation**: Formulate new tasks based on outcomes
3. **Task Prioritization**: Rearrange tasks contextually

Tasks stored in a queue alongside relevant context in a vector database. Lightweight, research-inspired, great for experimentation.

### 5.6 Microsoft Agent Framework (AutoGen v2 + Semantic Kernel)

**Architecture evolution:**
- AutoGen v0.4: Asynchronous, event-driven architecture
- Agents communicate through asynchronous messages (event-driven + request/response)
- Graph-based Workflow APIs replacing event-driven models
- Agents-as-tools pattern: agents can be used as tools by other agents

**Enterprise features:**
- Session-based state management
- Type safety, filters, telemetry
- Model Context Protocol (MCP) support
- Agent-to-Agent communication protocol

### 5.7 LangGraph

**Core: Stateful graph architecture**
- Agents as nodes in a directed graph
- Explicit, reducer-driven state schemas using TypedDict
- Checkpointing for persistent memory and safe parallel execution
- Time-travel debugging (roll back to any state)

**Execution patterns:**
- Scatter-gather (distribute + consolidate)
- Pipeline parallelism (sequential stages concurrently)
- Human-in-the-loop (pause at specific nodes for review)

**Production features:**
- Automated retries, per-node timeouts
- Pause/resume at specific nodes
- Time-travel debugging

### 5.8 Manus AI

**Architecture: Multi-agent with Planner-Executor-Verifier:**

1. **Planner Agent**: Breaks problems into manageable sub-tasks
2. **Executor Agent**: Executes one tool action per iteration (strict limit)
3. **Verifier Agent**: Reviews outputs

**Context Engineering Innovations (Major Contributions):**

| Strategy | Description |
|----------|-------------|
| KV-Cache optimization | 10x cost difference between cached (0.30/MTok) and uncached (3.00/MTok) tokens |
| Append-only context | Never modify prior actions/observations -- cache invalidation is devastating |
| Dynamic tool masking | Mask token logits instead of removing tools -- preserves KV-cache |
| File system as memory | Unlimited, persistent, agent-operable -- externalizes long-term state |
| Todo.md recitation | Constantly rewrite objectives into context end -- fights "lost-in-the-middle" |
| Error preservation | Keep failed actions in context -- model learns to avoid repeating mistakes |
| Context diversity | Vary serialization templates -- prevents pattern mimicry degradation |
| Reversible compression | Drop content but preserve URLs/paths -- information recoverable when needed |

**Single most important metric:** KV-cache hit rate. With Claude Sonnet, cached tokens cost 0.30 USD/MTok vs 3 USD/MTok uncached -- a 10x difference.

---

## 6. Common Patterns Across All Systems

### 6.1 The Universal Agent Loop

Every system implements a variant of the same fundamental loop:

```
OBSERVE (gather context)
   |
THINK (reason about next action)
   |
ACT (execute tool/action)
   |
VERIFY (check result against goal)
   |
LEARN (update state, memory, weights)
   |
REPEAT (until goal achieved or budget exhausted)
```

**Variations:**
- Claude Code: Think-Act-Observe (no explicit verify phase; verification is an act)
- Manus: One action per iteration (strict)
- SWE-Agent: Forward()-Execute-Append (minimal)
- CUGA: Plan-Decompose-Execute-Reflect (hierarchical)

### 6.2 Context Management: The Four Strategies

Anthropic's framework (validated across all systems studied) identifies four core strategies:

**1. WRITE Context (Scratchpad/Notes)**
- Claude Code: TodoWrite, CLAUDE.md
- OpenClaw: MEMORY.md, daily logs
- Manus: todo.md constantly rewritten
- Cursor: Memories system
- CUGA: Dynamic task ledger

**2. SELECT Context (Retrieval)**
- Claude Code: Grep/glob (agentic search)
- OpenClaw: Vector + BM25 hybrid search
- Cursor: Hierarchical RAG with embedding
- CUGA: Structured variable management

**3. COMPRESS Context (Summarization)**
- Claude Code: Compaction at 92-95% capacity
- OpenClaw: Session compaction
- Cursor: 10M -> 8K token hierarchical compression
- SWE-Agent: HistoryProcessor
- Google ADK: Asynchronous LLM-driven summarization

**4. ISOLATE Context (Sub-agents)**
- Claude Code: Sub-agents with fresh context windows
- Manus: Planner/Executor/Verifier agents
- Open SWE: 4 specialized agents
- CUGA: Plan Controller + Plan-Execute agents

### 6.3 Planning Approaches

| System | Planning Style | Replanning? |
|--------|---------------|-------------|
| Claude Code | TodoWrite (dynamic checklist) | Yes, model-driven |
| Manus | Planner agent creates upfront plan | Yes, via re-planning loop |
| CUGA | Plan Controller with task ledger | Yes, dynamic re-planning |
| Devin | Checkpoint-based (plan-implement-test-fix) | Yes, at checkpoints |
| SWE-Agent | Minimal (model decides step by step) | No explicit replanning |
| LangGraph | Graph-based (state machine) | Yes, via graph transitions |
| Open SWE | Dedicated Planner agent | Yes, Reviewer can trigger |

### 6.4 Error Recovery Patterns

| Pattern | Description | Used By |
|---------|-------------|---------|
| Retry with context | Re-attempt with error message in context | All systems |
| Fresh start | Abandon current approach, start over | Devin, Claude Code |
| Backtracking | Revert to known-good state | LangGraph (time-travel), Claude Code (checkpoints) |
| Error preservation | Keep failed actions visible in context | Manus (deliberate strategy) |
| Confidence-based escalation | Ask human when confidence is low | Devin, CUGA |
| Self-reflection | Critique own output before finalizing | CUGA (reflection mechanisms), Reflexion pattern |

### 6.5 Tool Selection Intelligence

| System | Strategy |
|--------|----------|
| Claude Code | Model selects from tool list; bias toward independent discovery |
| Cursor | Provider API handles tool-calling syntax; model selects |
| Manus | Dynamic tool masking via token logit manipulation |
| CUGA | Policy-governed tool access with human approval gates |
| SWE-Agent | YAML-configured tool sets; Live-SWE can invent tools |

### 6.6 When to Ask User vs. Decide Autonomously

| Level | Role | When to Use | Example |
|-------|------|-------------|---------|
| L1 | Operator | User controls everything | Simple chatbot |
| L2 | Collaborator | Agent suggests, user confirms | Code review with suggestions |
| L3 | Consultant | Agent acts, asks on key decisions | "I found 3 approaches, which do you prefer?" |
| L4 | Approver | Agent pre-selects, user rubber-stamps | "I'll use approach A unless you object" |
| L5 | Observer | Fully autonomous, user monitors | Background task execution |

**Best practice (from Devin and Claude Code):**
- Default to L3-L4 for most tasks
- Escalate to L2 when confidence is low
- Use L5 only for well-scoped, reversible tasks
- ALWAYS escalate for irreversible actions (deployments, database changes, external API calls)

### 6.7 Long-Running Task Patterns

| Pattern | Description | Used By |
|---------|-------------|---------|
| Session-based progress | Progress files + git history survive context resets | Claude Code |
| Checkpoint-commit | Git commit at each milestone | Claude Code, Devin |
| Feature list tracking | JSON/structured list of requirements with status | Claude Code |
| Todo.md recitation | Constantly rewrite plan into recent context | Manus |
| Sub-agent isolation | Delegate sub-tasks to agents with fresh context | Claude Code, Open SWE |
| Graph checkpointing | Checkpoint state at every graph node | LangGraph |
| Session compaction | Summarize and restart with compressed context | All modern systems |

---

## 7. Key Insights for corteX

### 7.1 The CORE Insight: Simplicity Beats Complexity in the Agent Loop

The most successful agentic systems (Claude Code, Manus, SWE-Agent) share a counterintuitive truth: **the agent loop itself should be as simple as possible.** The intelligence comes from the LLM, not from the harness.

Claude Code's entire loop is: `while(tool_call) -> execute -> feed back -> repeat`.
SWE-Agent's control loop is ~100 lines of Python.
Manus explicitly says their biggest performance gains came from **removing things**, not adding them.

**What this means for corteX:** The current 15-step pipeline in `Session.run()` may be OVER-ENGINEERED for the agent loop itself. The brain-inspired modules (weights, plasticity, prediction, game theory) are valuable for LEARNING AND ADAPTATION, but they should not be in the critical path of every agent turn. They should inform the context that goes into the LLM call, not gate the loop itself.

### 7.2 Context Engineering is the Real Battleground

Every system studied has converged on the same conclusion: **the #1 bottleneck in agent systems is context management, not model capability.**

Manus states: "The biggest performance gains came not from adding complex RAG pipelines or fancy routing logic, but from removing things."

Google ADK: "The biggest bottleneck in multi-agent systems stopped being the LLM itself and became context management."

**The four-strategy framework (Write/Select/Compress/Isolate) is universal.** Every production system implements all four. corteX's CCE (Contextual Context Engine) already has compression (L0-L3) and some selection, but needs stronger Write (scratchpad/notes) and Isolate (sub-agent) capabilities.

### 7.3 KV-Cache Awareness is a Production Necessity

Manus's insight about KV-cache hit rates is profound and largely missing from the corteX architecture:

- **Stable prompt prefixes**: Even a single-token difference invalidates the cache from that point onward
- **Append-only context**: Never modify prior actions/observations
- **Don't add/remove tools mid-iteration**: Tool definitions are near the front of context; changing them invalidates everything
- **10x cost difference**: Cached vs uncached tokens

**For corteX:** The brain state injection (BrainStateInjector) and dynamic parameter resolution may be HARMING KV-cache performance if they produce slightly different system prompts on each turn. The system prompt should be deterministic and stable.

### 7.4 The File System IS External Memory

Manus and Claude Code both use the file system as the agent's primary external memory:
- Unlimited in size
- Persistent by nature
- Directly operable by the agent
- Survives context window resets

**For corteX:** Instead of complex in-memory hierarchies, consider teaching agents to write and read files (progress logs, scratchpads, notes) as their primary memory mechanism. The memory system should facilitate this, not replace it.

### 7.5 "Lost in the Middle" is Real and Must Be Addressed

The transformer attention mechanism creates n-squared pairwise relationships for n tokens. As context grows, attention stretches thin. Critical information placed in the middle of long contexts is reliably missed.

**Solutions seen across systems:**
- Manus: Constantly rewrite objectives (todo.md) into the END of context
- Claude Code: CLAUDE.md is injected at the BEGINNING; recent messages at the END
- Cursor: Hierarchical context puts most relevant code at position extremes
- Google ADK: Narrative casting reframes context to maintain coherence

**For corteX:** The brain state injection should place the MOST IMPORTANT information (current goal, next steps, critical constraints) at the BEGINNING and END of the context window, never buried in the middle.

### 7.6 Verification Through Action, Not Reasoning

Claude Code doesn't have a separate "verify" module -- it verifies by DOING (running tests, checking outputs, reading files). This is more reliable than asking the LLM "is this correct?"

**For corteX:** The quality estimation (population coding) and prediction engine are valuable for LEARNING signals, but verification should primarily happen through tool execution (run tests, check outputs, validate with external systems).

### 7.7 Sub-Agents Are Essential for Long Tasks

Every mature system uses sub-agents to manage context pressure:
- Fresh context window per sub-agent
- Only the summary returns to the parent
- Enables parallel work without context bloat
- 1,000-2,000 token summaries from sub-agents are optimal

**For corteX:** The current architecture lacks proper sub-agent support. This is a critical gap for long-running tasks.

### 7.8 Human Interaction Should Be Interrupt-Based, Not Checkpoint-Based

Claude Code's h2A queue allows mid-task interruption. This is fundamentally different from the typical "plan, show plan, get approval, execute" pattern.

**For corteX:** The "smart timeout" requirement (if user doesn't respond, decide autonomously) should be combined with an interrupt-based model. The agent proceeds autonomously at L4 (pre-selects decisions), but the user can interrupt at any point. If the user doesn't respond within timeout, the agent continues with its best judgment.

### 7.9 Tool Design is as Important as Prompt Design

Anthropic's principle: invest tool documentation and testing effort EQUAL to human-computer interface design. Tools should:
- Use absolute paths (eliminates whole class of errors)
- Return token-efficient information
- Be self-contained and unambiguous
- Have comprehensive documentation with examples and edge cases

### 7.10 Enterprise Policy is a First-Class Concern

CUGA's 5-policy system is the most sophisticated enterprise guardrail pattern found:
- Intent Guard (blocks unauthorized actions)
- Playbook (standardizes workflows)
- Tool Approval (human oversight gates)
- Tool Guide (domain knowledge)
- Output Formatter (controls response generation)

**For corteX:** The enterprise weight tier already exists, but should be expanded to a proper policy engine that can be configured by the SaaS developer's enterprise customers.

---

## 8. Recommended Architecture for corteX Agentic Engine

### 8.1 Design Principles

Based on all research, the agentic engine should follow these principles:

1. **Simple loop, rich context**: The agent loop is thin; intelligence is in context engineering
2. **Append-only context**: Never modify prior messages; KV-cache awareness by default
3. **Four-strategy context management**: Write, Select, Compress, Isolate -- all four are required
4. **Externalized memory**: File system and structured artifacts are the primary memory
5. **Interrupt-based interaction**: User can inject at any point, not just checkpoints
6. **Brain modules inform context, not gate the loop**: Weights, plasticity, prediction operate OUTSIDE the critical path
7. **Sub-agents for isolation**: Fresh context windows for delegated work
8. **Verification through action**: Run tests and check outputs, don't just reason
9. **Enterprise policies as first-class**: Policy engine configurable per tenant
10. **Provider-agnostic with model routing**: Fast model for simple tasks, frontier for complex ones

### 8.2 Proposed Architecture

```
+------------------------------------------------------------------+
|                        corteX Agentic Engine                      |
+------------------------------------------------------------------+
|                                                                   |
|  [1. INPUT ROUTER]                                               |
|  User message -> Intent classification -> Autonomy level select  |
|  (L1-L5 based on task risk, user preferences, confidence)        |
|                                                                   |
|  [2. CONTEXT COMPILER]                                           |
|  Assembles the context window for each LLM call:                 |
|  +----------------------------------------------------------+   |
|  | SYSTEM PROMPT (stable, cached)                            |   |
|  |   - Agent identity + capabilities                         |   |
|  |   - Enterprise policies (from Policy Engine)              |   |
|  |   - Tool definitions (stable set, never changed mid-task) |   |
|  |   - User preferences (from Insight Store)                 |   |
|  +----------------------------------------------------------+   |
|  | PERSISTENT CONTEXT (CLAUDE.md equivalent)                 |   |
|  |   - Goal statement + progress summary                     |   |
|  |   - Brain state digest (from Brain Modules)               |   |
|  |   - Active constraints + guardrails                       |   |
|  +----------------------------------------------------------+   |
|  | WORKING MEMORY (selected + compressed)                    |   |
|  |   - Relevant conversation history (compressed)            |   |
|  |   - Retrieved context (from tools/search)                 |   |
|  |   - Scratchpad/notes (agent-written)                      |   |
|  +----------------------------------------------------------+   |
|  | RECENT CONTEXT (raw, uncompressed)                        |   |
|  |   - Last N messages (hot context)                         |   |
|  |   - Current task state + next steps                       |   |
|  |   - Goal restatement (fights "lost in the middle")        |   |
|  +----------------------------------------------------------+   |
|                                                                   |
|  [3. AGENT LOOP] (thin, simple)                                  |
|  while True:                                                     |
|      context = context_compiler.compile()                        |
|      response = llm.generate(context)                            |
|      if response.has_tool_calls:                                 |
|          for tool_call in response.tool_calls:                   |
|              result = tool_executor.execute(tool_call)            |
|              context_compiler.append(tool_call, result)          |
|          # Check interrupt queue                                  |
|          if interrupt_queue.has_message():                        |
|              context_compiler.inject(interrupt_queue.pop())       |
|          # Check budget (token budget, time budget, step budget) |
|          if budget_exceeded():                                    |
|              trigger_wrap_up()                                    |
|      else:                                                        |
|          # No tool calls = response complete                      |
|          break                                                    |
|  post_turn_learning()  # Brain modules update AFTER the turn     |
|                                                                   |
|  [4. CONTEXT MANAGEMENT STRATEGIES]                              |
|  +----------------------------------------------------------+   |
|  | WRITE: Scratchpad, progress notes, todo lists             |   |
|  | SELECT: Agentic search (grep/glob), semantic search       |   |
|  | COMPRESS: Compaction (summarize when >80% capacity)        |   |
|  | ISOLATE: Sub-agent spawning (fresh context windows)        |   |
|  +----------------------------------------------------------+   |
|                                                                   |
|  [5. MODEL ROUTER]                                               |
|  Routes to appropriate model based on:                           |
|  - Task complexity (simple -> fast model, complex -> frontier)   |
|  - Token budget remaining                                         |
|  - Model weights from Brain (learned preferences)                |
|  - Fallback chain (primary -> secondary -> tertiary)             |
|                                                                   |
|  [6. TOOL EXECUTOR]                                              |
|  - Permission-gated execution                                    |
|  - Reputation tracking (from Brain)                              |
|  - Timeout management                                             |
|  - Result truncation (token-efficient outputs)                   |
|  - Sandboxing for untrusted operations                           |
|                                                                   |
|  [7. BRAIN MODULES] (operate OUTSIDE the critical loop path)     |
|  Asynchronous learning and adaptation:                           |
|  - Weight updates (tool, model, behavioral)                     |
|  - Plasticity (Hebbian, LTP/LTD)                                |
|  - Prediction engine (surprise signals)                          |
|  - Goal tracker (drift detection)                                |
|  - Calibration engine                                            |
|  - Game theory (when multiple tools/strategies compete)          |
|  These INFORM the context compiler, don't GATE the loop.         |
|                                                                   |
|  [8. INTERRUPT QUEUE]                                            |
|  - User messages during execution                                |
|  - Timeout events (auto-decide if user doesn't respond)          |
|  - System events (memory pressure, error cascades)               |
|  - Priority queue (critical interrupts skip the queue)           |
|                                                                   |
|  [9. POLICY ENGINE]                                              |
|  Enterprise-configurable guardrails:                             |
|  - Intent guard (block unauthorized actions)                     |
|  - Tool approval gates (require human approval)                  |
|  - Output formatting rules                                       |
|  - Compliance constraints                                        |
|  - Budget limits (token, cost, time)                             |
|                                                                   |
|  [10. SUB-AGENT MANAGER]                                         |
|  - Spawn sub-agents with fresh context windows                   |
|  - Pass goal + minimal context to sub-agent                      |
|  - Collect 1-2K token summaries back                             |
|  - Parallel execution support                                    |
|  - Timeout and cancellation                                      |
|                                                                   |
|  [11. SESSION PERSISTENCE]                                       |
|  - Progress files (JSON, not Markdown -- resistant to drift)     |
|  - Git-based checkpoints                                         |
|  - Scratchpad files                                               |
|  - Cross-session memory (insight store)                          |
|                                                                   |
+------------------------------------------------------------------+
```

### 8.3 Context Compiler: Detailed Design

The Context Compiler is the most critical component. It assembles the context window for EVERY LLM call, following these rules:

**Rule 1: Stable prefix, variable suffix**
```
[System prompt + tools]  <- NEVER changes (maximizes KV-cache hits)
[Persistent context]     <- Changes rarely (goal, preferences, policies)
[Working memory]         <- Changes per-turn (compressed history, retrieved context)
[Recent context]         <- Changes per-turn (latest messages, current state)
[Goal restatement]       <- ALWAYS at the end (fights lost-in-the-middle)
```

**Rule 2: Token budgets per zone**

| Zone | Budget | Notes |
|------|--------|-------|
| System prompt + tools | 10-15% | Stable, cached |
| Persistent context | 5-10% | Goal, preferences, brain digest |
| Working memory | 30-40% | Compressed history, retrieved context |
| Recent context | 30-40% | Last N messages, raw tool outputs |
| Goal restatement | 1-2% | Current goal + next steps |
| Reserve | 10% | Output token budget |

**Rule 3: Compaction triggers**
- At 80% capacity: Clear old tool outputs (lightweight compaction)
- At 90% capacity: Summarize working memory (full compaction)
- At 95% capacity: Emergency compaction (summarize everything, keep only goal + recent)

**Rule 4: Goal always at both extremes**
- Goal statement in persistent context (near the beginning)
- Goal restatement at the very end of context
- This ensures the model NEVER loses sight of what it's trying to achieve

### 8.4 Agent Loop: Detailed Design

```python
class AgentLoop:
    async def run(self, goal: str) -> AgentResult:
        self.context_compiler.set_goal(goal)
        self.brain.goal_tracker.initialize(goal)

        step = 0
        while step < self.max_steps:
            # 1. Compile context (brain modules inform this)
            context = self.context_compiler.compile(
                brain_digest=self.brain.get_digest(),
                model_params=self.brain.get_model_params(),
            )

            # 2. Check if compaction needed
            if context.token_count > self.compaction_threshold:
                context = await self.context_compiler.compact(context)

            # 3. Route to appropriate model
            model = self.model_router.select(
                task_complexity=self.brain.estimate_complexity(),
                token_budget=self.budget.remaining(),
                model_weights=self.brain.model_weights,
            )

            # 4. Generate response
            response = await model.generate(context)

            # 5. Handle tool calls
            if response.has_tool_calls():
                for tool_call in response.tool_calls:
                    # Policy check
                    if not self.policy_engine.approve(tool_call):
                        result = ToolResult(error="Blocked by policy")
                    else:
                        result = await self.tool_executor.execute(tool_call)
                    self.context_compiler.append(tool_call, result)

                # 6. Check interrupt queue
                while self.interrupt_queue.has_message():
                    msg = self.interrupt_queue.pop()
                    self.context_compiler.inject_interrupt(msg)

                # 7. Budget check
                if self.budget.exceeded():
                    await self.wrap_up()
                    break

            else:
                # No tool calls = turn complete
                break

            step += 1

        # 8. Post-turn learning (ASYNC, non-blocking)
        asyncio.create_task(self.brain.post_turn_update(
            goal=goal,
            steps=step,
            context_summary=self.context_compiler.get_summary(),
        ))

        return AgentResult(response=response, steps=step)
```

### 8.5 Smart User Interaction: Detailed Design

The autonomy level system handles the "if user doesn't respond" problem:

```python
class AutonomyManager:
    """Manages when to ask user vs decide autonomously."""

    async def decide_or_ask(self, decision: Decision) -> Choice:
        level = self.get_autonomy_level(decision)

        if level == AutonomyLevel.AUTONOMOUS:
            # L5: Just do it
            return self.best_choice(decision)

        elif level == AutonomyLevel.APPROVER:
            # L4: Present decision, wait with timeout
            choice = self.best_choice(decision)
            response = await self.ask_user_with_timeout(
                message=f"I'll proceed with {choice} unless you say otherwise.",
                timeout=self.timeout_for(decision),
                default=choice,  # If timeout, use our choice
            )
            return response or choice

        elif level == AutonomyLevel.CONSULTANT:
            # L3: Present options, wait with timeout
            options = self.rank_choices(decision)
            response = await self.ask_user_with_timeout(
                message=f"I found {len(options)} approaches. Which do you prefer?",
                options=options,
                timeout=self.timeout_for(decision),
                default=options[0],  # If timeout, use best option
            )
            return response or options[0]

        else:
            # L2/L1: Must wait for user
            return await self.ask_user(decision)

    def get_autonomy_level(self, decision: Decision) -> AutonomyLevel:
        """Determine autonomy level based on risk, reversibility, confidence."""
        if decision.is_irreversible:
            return AutonomyLevel.COLLABORATOR  # Must confirm
        if decision.risk > self.risk_threshold:
            return AutonomyLevel.CONSULTANT  # Present options
        if self.brain.confidence < 0.7:
            return AutonomyLevel.CONSULTANT  # Not sure, ask
        if decision.risk < 0.3 and self.brain.confidence > 0.9:
            return AutonomyLevel.AUTONOMOUS  # Safe and confident
        return AutonomyLevel.APPROVER  # Default: pre-select, seek approval

    def timeout_for(self, decision: Decision) -> float:
        """Smart timeout based on urgency and risk."""
        base = 30.0  # 30 seconds base
        if decision.is_time_sensitive:
            base = 10.0
        if decision.risk > 0.5:
            base *= 2  # More time for risky decisions
        return base
```

### 8.6 Sub-Agent Architecture

```python
class SubAgentManager:
    """Manages sub-agents with isolated context windows."""

    async def spawn(self, task: str, context_hint: str = "") -> SubAgentResult:
        """Spawn a sub-agent for a focused task."""
        sub_agent = AgentLoop(
            context_compiler=ContextCompiler(fresh=True),
            brain=self.brain.fork(),  # Lightweight brain copy
            model_router=self.model_router,
            tool_executor=self.tool_executor.scoped(),
            max_steps=self.sub_agent_max_steps,
            budget=self.budget.allocate_sub_budget(),
        )

        # Give minimal context: task + hint + relevant tools
        result = await sub_agent.run(
            goal=task,
            initial_context=context_hint,
        )

        # Return condensed summary (1-2K tokens max)
        return SubAgentResult(
            summary=self.summarize(result, max_tokens=2000),
            artifacts=result.artifacts,
            success=result.success,
        )

    async def spawn_parallel(self, tasks: list[str]) -> list[SubAgentResult]:
        """Spawn multiple sub-agents in parallel."""
        return await asyncio.gather(*[
            self.spawn(task) for task in tasks
        ])
```

### 8.7 How Brain Modules Integrate (Outside the Critical Path)

The brain modules from v1 are PRESERVED but their role changes:

```
BEFORE (v1): Brain modules are IN the loop
  message -> feedback -> goal_track -> context -> columns -> resources
  -> concepts -> cross_modal -> prediction -> proactive -> dual_process
  -> LLM -> tools -> quality -> surprise -> plasticity -> memory

AFTER (agentic): Brain modules INFORM the context compiler
  message -> context_compiler.compile(brain_digest) -> LLM -> tools

  # Brain modules run ASYNCHRONOUSLY:
  post_turn: brain.update(turn_data)
    -> weights.update()
    -> plasticity.apply()
    -> prediction.compare()
    -> goal_tracker.check()
    -> calibration.update()

  # Brain digest is a SMALL summary injected into context:
  brain_digest = {
    "confidence": 0.85,
    "goal_progress": "67% complete",
    "tool_recommendations": ["grep", "edit"],
    "model_recommendation": "gemini-3-pro",
    "risk_level": "low",
    "user_preferences": {"verbosity": "concise", "autonomy": "high"},
  }
```

### 8.8 What Changes from v1 to Agentic Architecture

| Component | v1 Status | Agentic Change |
|-----------|-----------|-----------|
| Agent Loop | 15-step pipeline, synchronous | Thin loop: compile-generate-execute-repeat |
| Context Engine (CCE) | 3-temperature hierarchy | 4-zone Context Compiler with KV-cache awareness |
| Brain Modules | In critical path | Asynchronous, inform context compiler |
| Goal Tracker | In critical path | Runs post-turn, injects digest into context |
| Sub-Agents | Not implemented | New: SubAgentManager with isolated contexts |
| Compaction | L0-L3 compression | Add API-level compaction + scratchpad notes |
| User Interaction | Not implemented | New: AutonomyManager with smart timeouts |
| Model Routing | NashRoutingOptimizer | Simplified: complexity-based routing with brain weights |
| Tool Selection | Reputation system | Preserved, but tool set is STABLE per session |
| Policy Engine | Enterprise weights | New: 5-policy system (Intent, Playbook, Approval, Guide, Format) |
| Interrupt Queue | Not implemented | New: h2A-style interrupt queue |
| Session Persistence | Memory modules | Add: progress files, scratchpad, git checkpoints |
| KV-Cache Awareness | Not implemented | New: stable prefix, append-only context |

### 8.9 Implementation Priority

**Phase 1: Core Loop Refactor (Critical)**
1. Simplify agent loop to thin compile-generate-execute pattern
2. Implement Context Compiler with 4-zone architecture
3. Move brain modules to post-turn async processing
4. Add compaction with configurable triggers

**Phase 2: Interaction & Isolation (High)**
5. Implement Interrupt Queue for mid-task user interaction
6. Implement AutonomyManager with smart timeouts
7. Implement SubAgentManager with isolated context windows

**Phase 3: Enterprise & Production (Medium)**
8. Implement Policy Engine (5 policy types)
9. Add KV-cache optimization (stable prefixes, append-only)
10. Add session persistence (progress files, scratchpad)

**Phase 4: Intelligence Enhancement (Enhancement)**
11. Model Router with learned weights
12. Enhanced tool reputation with Bayesian updates
13. Cross-session learning and trajectory reuse (like CUGA)

---

## Appendix A: Source References

### Claude Code
- [How Claude Code Works](https://code.claude.com/docs/en/how-claude-code-works) -- Official documentation
- [Claude Code: Behind the Master Agent Loop](https://blog.promptlayer.com/claude-code-behind-the-scenes-of-the-master-agent-loop/) -- Technical deep dive
- [Effective Harnesses for Long-Running Agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) -- Anthropic Engineering
- [Building Agents with the Claude Agent SDK](https://claude.com/blog/building-agents-with-the-claude-agent-sdk) -- Anthropic Engineering
- [Claude Agent SDK Python](https://github.com/anthropics/claude-agent-sdk-python) -- GitHub
- [Compaction API Documentation](https://platform.claude.com/docs/en/build-with-claude/compaction) -- Official API docs
- [Claude Code: A Simple Loop That Produces High Agency](https://medium.com/@aiforhuman/claude-code-a-simple-loop-that-produces-high-agency-814c071b455d) -- Analysis
- [Claude Code Agent Architecture](https://www.zenml.io/llmops-database/claude-code-agent-architecture-single-threaded-master-loop-for-autonomous-coding) -- ZenML

### OpenClaw
- [OpenClaw GitHub Repository](https://github.com/openclaw/openclaw) -- Source code
- [OpenClaw Architecture Overview](https://ppaolo.substack.com/p/openclaw-system-architecture-overview) -- Technical analysis
- [OpenClaw Complete Guide](https://milvus.io/blog/openclaw-formerly-clawdbot-moltbot-explained-a-complete-guide-to-the-autonomous-ai-agent.md) -- Milvus Blog

### CUGA
- [CUGA Agent GitHub](https://github.com/cuga-project/cuga-agent) -- Source code
- [IBM Research: Introducing CUGA](https://research.ibm.com/blog/cuga-agent-framework) -- IBM Research Blog
- [CUGA on Hugging Face](https://huggingface.co/blog/ibm-research/cuga-on-hugging-face) -- Hugging Face

### Cursor
- [Cursor AI Technical Architecture Deep Dive](https://collabnix.com/cursor-ai-deep-dive-technical-architecture-advanced-features-best-practices-2025/) -- CollabiNix
- [Cursor Agent System Prompt (March 2025)](https://gist.github.com/sshh12/25ad2e40529b269a88b80e7cf1c38084) -- GitHub Gist
- [Cursor 2.0 Agent-First Architecture](https://www.digitalapplied.com/blog/cursor-2-0-agent-first-architecture-guide) -- Digital Applied
- [Cursor AI Review 2025: Agent Mode](https://skywork.ai/blog/cursor-ai-review-2025-agent-refactors-privacy/) -- Skywork

### Devin
- [Devin 2025 Performance Review](https://cognition.ai/blog/devin-annual-performance-review-2025) -- Cognition AI
- [Coding Agents 101](https://devin.ai/agents101) -- Devin Documentation

### SWE-Agent
- [SWE-Agent Architecture](https://swe-agent.com/latest/background/architecture/) -- Official Documentation
- [SWE-Agent GitHub](https://github.com/SWE-agent/SWE-agent) -- Source code
- [Open SWE](https://blog.langchain.com/introducing-open-swe-an-open-source-asynchronous-coding-agent/) -- LangChain Blog

### Context Engineering
- [Effective Context Engineering for AI Agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) -- Anthropic Engineering
- [Context Engineering: Lessons from Building Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus) -- Manus Blog
- [Google ADK Context Architecture](https://developers.googleblog.com/architecting-efficient-context-aware-multi-agent-framework-for-production/) -- Google Developers Blog

### Agent Design Patterns
- [Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) -- Anthropic Research
- [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/overview/agent-framework-overview) -- Microsoft Learn
- [LangGraph Architecture](https://latenode.com/blog/ai-frameworks-technical-infrastructure/langgraph-multi-agent-orchestration/) -- Latenode

### Academic Research
- [Agentic AI: Comprehensive Survey of Architectures](https://arxiv.org/html/2510.25445v1) -- arXiv
- [Agentic AI Frameworks: Architectures, Protocols, Design Challenges](https://arxiv.org/html/2508.10146v1) -- arXiv
- [Agentic Context Engineering](https://arxiv.org/abs/2510.04618) -- arXiv
- [Agent Design Pattern Catalogue](https://arxiv.org/html/2405.10467v2) -- arXiv
- [BacktrackAgent: Error Detection in GUI Agents](https://aclanthology.org/2025.emnlp-main.212.pdf) -- EMNLP 2025
- [ReAgent: Reversible Multi-Agent Reasoning](https://aclanthology.org/2025.emnlp-main.202.pdf) -- EMNLP 2025

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **Agent Loop** | The core think-act-observe-repeat cycle that drives agent behavior |
| **Compaction** | Summarizing conversation history to free context window space |
| **Context Compiler** | Component that assembles the optimal context window for each LLM call |
| **Context Engineering** | The discipline of curating optimal token sets for LLM inference |
| **KV-Cache** | Key-Value cache in transformer inference; reuses computation for unchanged prefixes |
| **L1-L5 Autonomy** | Levels of agent autonomy from human-controlled (L1) to fully autonomous (L5) |
| **Lost in the Middle** | LLM tendency to miss information placed in the middle of long contexts |
| **Scratchpad** | Agent-writable notes that persist during task execution |
| **Sub-Agent** | Agent instance with its own context window for delegated work |
| **Token Budget** | Allocation of context window tokens across different content zones |
