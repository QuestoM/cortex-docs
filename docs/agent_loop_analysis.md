# corteX Agent Loop: Deep Analysis & Agentic Engine Design

**Author:** Claude Opus 4.6
**Date:** 2026-02-14
**Scope:** Complete analysis of `Session.run()` in `corteX/sdk.py`, all supporting engine modules, gap identification, and Agentic Engine design.

---

## Table of Contents

1. [Complete Agent Loop Map](#1-complete-agent-loop-map)
2. [Data Flow Diagram](#2-data-flow-diagram)
3. [Component-by-Component Analysis](#3-component-by-component-analysis)
4. [Gap Analysis](#4-gap-analysis)
5. [Agentic Engine Design](#5-agentic-engine-design)
6. [Implementation Roadmap](#6-implementation-roadmap)

---

## 1. Complete Agent Loop Map

### Session.run() Step-by-Step (sdk.py lines 333-886)

The current `Session.run()` is a **single-turn reactive loop** -- it processes one user message and returns one response. It is NOT a multi-step agentic loop. Here is every step:

#### Phase A: Pre-Processing (lines 347-432)

| Step | Lines | What Happens | Input | Output |
|------|-------|-------------|-------|--------|
| A1 | 347-348 | Timer start + turn counter increment | - | `start_time`, `_turn_count` |
| A2 | 350-363 | **Feedback processing** -- Tier1 regex detects implicit signals (frustration, satisfaction, correction) from user message. AdaptationFilter applies habituation (dampens repeated signals). | `message` | `feedback_signals` (with adapted strengths) |
| A3 | 365-368 | **Goal tracker init** -- On first message only, creates GoalTracker with user message as the goal. Sets goal in context engine. | `message` | `_goal_tracker` instance |
| A4 | 370-372 | **History append** -- Adds user LLMMessage to `_messages` list and context engine. | `message` | Updated `_messages`, CCE hot items |
| A5 | 374-380 | **Attention filtering** (P2) -- Classifies task type via keyword heuristic (`_classify_task_type`, lines 1158-1171), then `AttentionSystem.process_turn()` determines priority level (SUBCONSCIOUS, BACKGROUND, FOREGROUND, CRITICAL). | `message`, turn context | `task_type`, `attention_result`, `attention_priority` |
| A6 | 382-385 | **Column selection** (P2) -- `ColumnManager.select_column()` picks the best specialized cortical column for this task. Gets column recommendations (model tier, weight overrides). | `task_type`, `message`, `tool_names` | `active_column`, `column_recs` |
| A7 | 387-388 | **Resource allocation** (P2) -- `ResourceHomunculus.allocate()` returns token budget and model tier for this task type. | `task_type` | `resource_alloc` |
| A8 | 390-393 | **Concept activation** (P3) -- Activates matching concepts in the concept graph based on task type and available tools. Gets recommendations (suggested model tier, related tools). | `task_type`, `tool_names` | `active_concepts`, `concept_recs` |
| A9 | 395-409 | **Cross-modal enrichment** (P1) -- Looks up learned associations from the CrossModalAssociator. If associations exist, injects them as context into the last user message (appended as `[Context associations: ...]`). | `task_type`, `tool_names` | Modified last message in `_messages` |
| A10 | 411-415 | **Reactive prediction** -- `PredictionEngine.predict()` generates a prediction for this turn (expected outcome, latency, quality) based on historical tool/model stats. | `message[:100]`, model name | `pred` (Prediction object) |
| A11 | 417-422 | **Proactive prediction** (P0) -- Predicts what the user will do NEXT turn. Schedules pre-warming. Feeds reactive surprise into proactive engine. | Conversation trajectory | `proactive_pred` |
| A12 | 424-433 | **Dual-process routing** -- Builds `EscalationContext` from surprise, population agreement, safety level, error history, goal drift. `DualProcessRouter.route()` returns SYSTEM1 (fast) or SYSTEM2 (slow). | Multi-signal context | `process_type` (SYSTEM1/SYSTEM2) |
| A13 | 435-452 | **Targeted modulation** (P3) + **Tool filtering** -- Modulator ticks time. Reputation system filters quarantined tools. Modulator silences/activates tools. ToolExecutor builds filtered definitions. | All tools, modulation context | `tool_defs` (filtered), `available_tools` |

#### Phase B: LLM Generation (lines 457-518)

| Step | Lines | What Happens | Input | Output |
|------|-------|-------------|-------|--------|
| B1 | 458-471 | **Role selection** -- Multi-signal voting: AttentionalPriority, ProcessType, ResourceHomunculus, ConceptGraph, ColumnManager all contribute to choosing "orchestrator" (expensive/quality) vs "worker" (cheap/fast). | attention_priority, process_type, resource_alloc, concept_recs, column_recs | `role` |
| B2 | 473-488 | **Brain state injection** -- `_build_brain_augmented_prompt()` (lines 1173-1197) collects `BrainSnapshot` from all live components (behavioral weights, column mode, goal state, attention changes, calibration alerts, prediction surprise, active concepts, proactive predictions). `BrainStateInjector.compile_brain_context()` translates snapshot into structured markdown injected into system prompt. Then `StructuredOutputInjector` appends self-assessment instruction asking LLM to emit confidence/difficulty/escalation signals. | All brain state | `system_with_brain` (augmented system prompt) |
| B3 | 490-497 | **Brain parameter resolution** -- `_resolve_brain_parameters()` (lines 1298-1383) feeds task_type, process_type, attention_priority, column, resource alloc, surprise, calibration health, creativity, verbosity into `BrainParameterResolver.resolve()`. Returns `LLMParameterBundle` with temperature, top_p, top_k, max_tokens, frequency_penalty, presence_penalty, thinking_budget. | All cognitive signals | `param_bundle` (LLMParameterBundle) |
| B4 | 499-518 | **LLM call** -- `router.generate()` is called with messages, tools, role, system_instruction, and all brain-resolved parameters. On failure, returns error Response. | `_messages`, `tool_defs`, `role`, `system_with_brain`, `param_bundle` | `llm_response` (LLMResponse) |

#### Phase C: Tool Execution Loop (lines 520-591)

| Step | Lines | What Happens | Input | Output |
|------|-------|-------------|-------|--------|
| C1 | 521-523 | **Tool loop init** -- `max_tool_rounds = 5`, `tool_round = 0`. | - | Loop control vars |
| C2 | 525-526 | **Loop condition** -- While `llm_response.tool_calls` and `tool_round < max_tool_rounds`. | `llm_response` | Continue/exit decision |
| C3 | 528-535 | **Append assistant message** -- Adds assistant message (with tool_calls) to history and context engine. | `llm_response` | Updated `_messages`, CCE |
| C4 | 537-580 | **Execute each tool call** -- For each tool call: extracts name/args, calls `_tool_executor.execute_tool_call()`, records in WeightEngine (Bayesian + EMA), ReputationSystem (trust), CalibrationEngine (Platt-scaled prediction vs actual), ContextEngine (tool result), and appends tool result to `_messages`. | `tc` (tool call dict) | `result` (ToolExecutionResult), updated weights/reputation/calibration/messages |
| C5 | 582-591 | **Follow-up LLM call** -- Calls `router.generate()` again with updated messages. **NOTE: does NOT pass brain-resolved parameters** -- uses only messages, tools, and role. Temperature, top_p, etc. are not forwarded. | `_messages`, `tool_defs`, `role` | New `llm_response` |

#### Phase D: Post-Processing (lines 593-886)

| Step | Lines | What Happens | Input | Output |
|------|-------|-------------|-------|--------|
| D1 | 593-597 | **Final assistant message** -- Appends to history and context engine. | `llm_response.content` | Updated `_messages`, CCE |
| D2 | 599-602 | **Token tracking** -- Sums usage from response, adds to total, records in CCE. | `llm_response.usage` | `tokens_this_turn`, `_total_tokens` |
| D3 | 604-608 | **Quality estimation** -- `PopulationQualityEstimator.estimate()` computes quality vector from response content. Stores agreement for next turn's dual-process routing. | `llm_response.content` | `estimated_quality`, `quality_vector.agreement` |
| D4 | 610-616 | **Surprise signal** -- `prediction.compare()` matches prediction to actual outcome (success/failure, latency, quality). Computes surprise magnitude, direction, learning signal. | `pred`, actual outcome/latency/quality | `surprise` (SurpriseSignal) |
| D5 | 618-637 | **Structured signal parsing** -- `StructuredOutputParser.parse()` extracts self-assessed signals from LLM response. Strips signal block from user-facing content. `SignalAggregator.aggregate()` combines LLM self-assessment, prediction surprise, and population quality into composite confidence. Refines quality estimate. | `llm_response.content`, `surprise`, `estimated_quality` | `structured_signals`, refined `estimated_quality` |
| D6 | 639-659 | **Shapley attribution** -- For multi-tool turns, computes fair credit allocation across tools using Shapley values. Applies attribution to weight engine. | `tools_called`, `estimated_quality`, tool outcomes | Updated tool weights |
| D7 | 661-667 | **Plasticity** -- `PlasticityManager.on_step_complete()` fires Hebbian, LTP, LTD rules. Critical period modulates learning rates. Surprise amplifies learning. | model, success, quality, surprise | Plasticity events, updated weights |
| D8 | 669-684 | **Goal verification** -- `GoalTracker.verify_step()` checks if the step advanced the goal. Computes alignment score, drift delta, progress delta. Recommends action (continue/adjust/replan/abort). **NOTE: verification result is NOT acted upon -- only progress/drift/loop values are read.** | `message`, `llm_response.content[:500]` | `goal_progress`, `drift_score`, `loop_detected` |
| D9 | 686-694 | **Memory consolidation** -- Stores turn summary in working memory (key, content, importance). | user[:200], assistant[:200], quality | Working memory entry |
| D10 | 696-709 | **Calibration recording** -- Records model quality prediction accuracy for continuous calibration. | `estimated_quality` | Updated calibration tracker |
| D11 | 711-733 | **Proactive + Cross-modal recording** -- Records turn in proactive prediction engine (trajectory learning). Records co-occurrence in cross-modal associator (Hebbian binding). | Turn data | Updated proactive engine, cross-modal associations |
| D12 | 735-744 | **Metacognition check** -- Every 10 turns, runs calibration cycle. If alerts found (oscillation, stagnation, degradation), adjusts learning rate multiplier. | Calibration state | `_lr_multiplier` update |
| D13 | 746-820 | **Component learning** -- Records outcome in: active column, resource map, attention system, concept graph, map reorganizer. Every 25 turns: decay columns, reorganize resources, prune weak columns, concept maintenance, cortical reorganization. | Quality, success, latency, tools used | Updated P2/P3 components |
| D14 | 822-847 | **Nash routing** -- Records utility for model-task combo. Periodically runs Nash equilibrium iteration and applies scores to weight engine. | Model, task_type, quality, latency | Updated Nash routing strategy |
| D15 | 849-865 | **Semantic scorer + Summarization check** -- Feeds content to TF-IDF vocabulary builder. Checks if L2/L3 summarization threshold reached (but only logs, does not execute summarization). | Message, response content | Updated vocabulary, log entry |
| D16 | 867-886 | **Build Response** -- Constructs `ResponseMetadata` and `Response` object. Returns to caller. | All computed values | `Response` |

---

## 2. Data Flow Diagram

```
User Message
     |
     v
[A2: Feedback] --> updates WeightEngine (behavioral weights)
     |
     v
[A3: GoalTracker init] --> creates GoalTracker (first msg only)
     |
     v
[A4: History] --> appends to _messages + CCE
     |
     v
[A5-A8: Brain Pre-Processing]
  |-- AttentionSystem --> attention_priority (SUBCONSCIOUS..CRITICAL)
  |-- ColumnManager --> active_column (specialization, weight overrides)
  |-- ResourceHomunculus --> resource_alloc (token budget, model tier)
  |-- ConceptGraph --> active_concepts, concept_recs
     |
     v
[A9: Cross-Modal] --> enriches user message with learned associations
     |
     v
[A10-A11: Prediction] --> pred (expected outcome), proactive_pred (next turn)
     |
     v
[A12: DualProcess] --> SYSTEM1 or SYSTEM2 (7 escalation criteria)
     |
     v
[B1: Role Selection] --> "orchestrator" or "worker"
     |              (multi-signal vote from attention, dual-process,
     |               resource, concept, column)
     v
[B2: Brain Injection] --> system prompt + brain context markdown
     |                     + structured output instruction
     v
[B3: Parameter Resolution] --> temperature, top_p, max_tokens, etc.
     |
     v
[B4: LLM Call] --> router.generate(messages, tools, role, brain params)
     |
     v
[C: Tool Loop] --> up to 5 rounds
  |  |-- execute tool --> record in weights, reputation, calibration, CCE
  |  |-- follow-up LLM call (NOTE: no brain params in follow-ups!)
  |  v
[D: Post-Processing]
  |-- Quality estimation (population coding)
  |-- Surprise signal (prediction error)
  |-- Structured signal parsing (LLM self-assessment)
  |-- Shapley attribution (multi-tool credit)
  |-- Plasticity (Hebbian, LTP, LTD, homeostasis)
  |-- Goal verification (alignment, drift, loops)
  |-- Memory consolidation (working memory)
  |-- Calibration recording
  |-- Proactive + cross-modal recording
  |-- Metacognition check
  |-- Component learning (columns, resources, attention, concepts, reorganizer)
  |-- Nash routing optimization
  |-- Semantic scorer update
     |
     v
Response(content, metadata)
```

---

## 3. Component-by-Component Analysis

### 3.1 GoalTracker (engine/goal_tracker.py)

**What it does well:**
- State hashing with coarse MD5 for loop detection (line 198-211)
- Dual-mode alignment: heuristic keyword overlap + LLM verification fallback (lines 150-162)
- Drift scoring with continuous update (lines 164-177)
- Stall detection after 5 turns without progress (line 25, 295)
- Recommends actions: continue, adjust, replan, abort (lines 289-303)

**Critical gap:** GoalTracker's `recommended_action` is NEVER acted upon by `Session.run()`. The verification runs at line 676, but only `_progress`, `_drift_score`, and `_loop_detected` are read. No replanning, adjusting, or aborting ever happens.

### 3.2 CorticalContextEngine (engine/context.py)

**What it does well:**
- Three-tier memory hierarchy (hot/warm/cold) with configurable budgets
- Progressive compression (L0 verbatim -> L1 observation masking -> L2 summary -> L3 digest)
- Importance scoring with 6 weighted factors (recency, relevance, causal, reference, success, domain)
- Context window packing with primacy/recency bias optimization
- Checkpointing for fault-tolerant recovery
- Domain-specific compression profiles (coding, research)

**Critical gap:** The CCE is fully built but `Session.run()` never calls `get_context_window()`. The messages sent to the LLM are the raw `_messages` list (line 500-501), not the packed context. The CCE only records data; it never filters or compresses what the LLM actually sees. This means for long sessions, the context window will overflow.

### 3.3 WeightEngine (engine/weights.py)

**What it does well:**
- 7-tier weight system (behavioral, tool, model, goal, user insights, enterprise, global)
- Bayesian posteriors with Beta/Gamma distributions for principled uncertainty
- Prospect-theoretic asymmetric updates (loss aversion)
- Thompson Sampling for exploration/exploitation
- Momentum-based updates with homeostatic regulation
- Consolidation (sleep-like weight cleanup)

**No critical gaps** -- the weight engine is the most complete component.

### 3.4 FeedbackEngine (engine/feedback.py)

**What it does well:**
- 4-tier feedback system (direct, user insights, enterprise, global)
- Regex-based implicit signal detection (correction, frustration, satisfaction, brevity, speed, detail)
- Accumulates signals across sessions for user profiling
- Translates signals into weight updates

**Gap:** Feedback only processes the incoming user message. It does not analyze the user's response TO the agent's previous output (which is the more valuable signal). The feedback should compare what the agent said vs. what the user said next.

### 3.5 PredictionEngine (engine/prediction.py)

**What it does well:**
- Generates predictions before each action (outcome, latency, quality)
- Computes surprise signal (magnitude, direction) from prediction error
- Non-linear learning signal via tanh (small surprises suppressed, large amplified)
- Running statistics with EMA for continuous calibration
- Tool/model-specific prediction histories

**No critical gaps** -- well-designed, though currently only predicts at the turn level, not at the tool-call level within a turn.

### 3.6 LLMRouter (core/llm/router.py)

**What it does well:**
- Multi-provider support (OpenAI, Gemini, Local)
- Task-type-aware temperature selection with per-provider overrides
- Weight-based model selection with speed/latency/failure scoring
- Fallback to alternative models on failure
- Orchestrator/worker role-based default model selection

**Gap:** The router does not use the WeightEngine's `ModelSelectionWeights` or the `NashRoutingOptimizer` results for actual routing decisions. It has its own separate `_model_weights` dict that is never populated by the brain system. The brain learns optimal routing (Nash equilibrium, Shapley attribution) but the router ignores it.

### 3.7 BrainStateInjector (engine/brain_state_injector.py)

**What it does well:**
- Translates behavioral weights into natural language instructions
- Compiles goal state (progress, drift, loops, stalls) into actionable warnings
- Includes calibration warnings, prediction context, active concepts, proactive predictions
- Token budget enforcement (500 token max with smart truncation)

**No critical gaps** -- well-designed bridge between brain state and LLM prompt.

### 3.8 BrainParameterResolver (engine/parameter_resolver.py)

**What it does well:**
- Maps cognitive state to concrete LLM API parameters
- Provider-aware (OpenAI vs Gemini vs Local parameter support)
- Priority: Modulator CLAMP > Column override > Brain computation > Task default
- Full observability via get_stats()
- Delegated to focused sub-resolvers (temperature, token, penalty, helpers)

**No critical gaps** -- clean design.

---

## 4. Gap Analysis

### GAP 1: No Planning Engine

**Current state:** There is NO task decomposition or planning. `Session.run()` sends the user message directly to the LLM (line 500). The `GoalTracker.set_plan()` method exists (line 98) but is never called. No plan steps are ever created.

**What's missing:**
- Pre-execution planning: decompose user goal into sub-tasks
- Plan representation: ordered list of steps with dependencies
- Plan execution: iterate through steps, executing each
- Plan revision: when GoalTracker recommends "replan", actually generate a new plan
- Plan persistence: survive across turns

**How it should work:**
1. On first message (or when goal changes), call LLM to generate a plan
2. Store plan in GoalTracker via `set_plan()`
3. Execute plan steps one by one, verifying each
4. On "replan" recommendation, regenerate the remaining plan
5. Report plan progress to user

**Priority:** CRITICAL
**Complexity:** HIGH -- requires new module + significant `Session.run()` restructuring

### GAP 2: No Reflection/Self-Verification Loop

**Current state:** GoalTracker's `verify_step()` runs (line 676) AFTER the response is already finalized. The `recommended_action` (continue/adjust/replan/abort) is completely ignored. The verification happens too late and is never acted upon.

**What's missing:**
- Pre-response verification: "Is this answer actually correct?"
- Post-tool verification: "Did the tool do what I expected?"
- Quality gate: if quality estimate is below threshold, retry
- Self-correction: if response seems wrong, revise before sending to user

**How it should work:**
1. After LLM generates response (line 518), check quality estimate
2. If below threshold (e.g., 0.4), ask a different model to verify
3. If verification fails, regenerate with adjusted prompt
4. Limit retries to prevent infinite loops
5. GoalTracker's recommended_action should be checked BEFORE finalizing response

**Priority:** CRITICAL
**Complexity:** MEDIUM -- mostly control flow changes in `Session.run()`

### GAP 3: No Multi-Step Agentic Loop

**Current state:** `Session.run()` processes exactly ONE user message and returns ONE response. It is a chat completion wrapper, not an agent loop. There is no mechanism for the agent to take multiple autonomous steps toward a goal.

**What's missing:**
- Autonomous execution: agent takes N steps without user input
- Inner loop: plan -> act -> observe -> verify -> continue/stop
- Step budget: max steps before requiring user confirmation
- Intermediate output: report progress to user during execution
- Graceful interruption: user can stop mid-execution

**How it should work:**
```python
async def run_agentic(self, goal: str, max_steps: int = 20) -> Response:
    plan = await self._generate_plan(goal)
    for step in plan:
        result = await self._execute_step(step)
        verification = await self._verify_step(step, result)
        if verification.recommended_action == "abort":
            break
        if verification.recommended_action == "replan":
            plan = await self._replan(goal, completed_steps)
    return self._compile_final_response()
```

**Priority:** CRITICAL
**Complexity:** HIGH -- this is fundamentally a new execution mode

### GAP 4: No Context Window Management in LLM Calls

**Current state:** The `CorticalContextEngine` (context.py, 1111 lines) is a fully built context management system with three-tier memory, progressive compression, importance scoring, and context packing. However, `Session.run()` sends raw `self._messages` to the LLM (line 500), completely bypassing the CCE's packing capabilities.

**What's missing:**
- Integration point: replace `messages=self._messages` with `messages=self.context_engine.get_context_window()`
- Message format conversion: CCE returns `ContextItem` list, router expects `LLMMessage` list
- Dynamic budget: adjust context window based on model being used
- L2 summarization execution: CCE flags when L2 is needed but never actually calls the LLM to summarize

**How it should work:**
1. Before each LLM call, call `context_engine.get_context_window(model_context_window)`
2. Convert packed `ContextItem` list to `LLMMessage` list
3. Use this compressed message list instead of raw history
4. When L2 summarization is triggered, call worker model to generate summary
5. This enables 10,000+ step sessions without context overflow

**Priority:** CRITICAL
**Complexity:** MEDIUM -- CCE is already built, just needs integration

### GAP 5: No User Interaction Manager (Timeout + Autonomous Decisions)

**Current state:** There is no mechanism for the agent to ask the user a question, wait for response, or make autonomous decisions when the user doesn't respond. The API is pure request/response.

**What's missing:**
- Question generation: agent identifies when it needs clarification
- Timeout system: if user doesn't respond within T seconds, proceed autonomously
- Autonomous decision-making: use LLM to decide "what would the user likely choose?"
- Escalation rules: some decisions MUST have user input (e.g., deleting data)
- Interaction budget: limit how many times agent asks before deciding alone

**How it should work:**
1. Agent detects uncertainty during planning/execution
2. Generates a question with options (A, B, C)
3. Waits for user response (configurable timeout)
4. If timeout expires, uses orchestrator model to predict user's likely choice
5. Proceeds with predicted choice, noting it was autonomous
6. Enterprise config controls which decisions can be autonomous

**Priority:** HIGH
**Complexity:** HIGH -- requires async event loop, callback system, or webhook pattern

### GAP 6: Follow-Up LLM Calls Lack Brain Parameters

**Current state:** The first LLM call (line 500) gets full brain-resolved parameters (temperature, top_p, max_tokens, etc.). But follow-up calls in the tool loop (line 584) only pass messages, tools, and role. No temperature, no brain injection, no parameter resolution.

**What's missing at line 584:**
```python
llm_response = await self.agent.engine.router.generate(
    messages=self._messages,
    tools=tool_defs,
    role=role,
    # MISSING: system_instruction=system_with_brain,
    # MISSING: temperature=param_bundle.temperature,
    # MISSING: top_p=param_bundle.top_p,
    # MISSING: top_k=param_bundle.top_k,
    # MISSING: max_tokens=param_bundle.max_tokens,
    # MISSING: frequency_penalty=param_bundle.frequency_penalty,
    # MISSING: presence_penalty=param_bundle.presence_penalty,
    # MISSING: thinking_budget=param_bundle.thinking_budget,
)
```

**Priority:** HIGH
**Complexity:** LOW -- just pass the existing variables

### GAP 7: No Error Recovery / Backtracking

**Current state:** If the LLM call fails (line 513), `Session.run()` returns an error string. If a tool fails, it just appends the error to messages and lets the LLM try to recover. There is no systematic error recovery strategy.

**What's missing:**
- Retry with different parameters (higher temperature, different model)
- Backtracking: undo the failed step and try alternative approach
- Error classification: transient (retry) vs. permanent (replan) vs. fatal (abort)
- Exponential backoff for rate limits
- Context recovery from checkpoints when context gets corrupted

**How it should work:**
1. Classify error: transient, permanent, fatal
2. Transient: retry with backoff (up to 3 retries)
3. Permanent: replan (generate alternative approach)
4. Fatal: return error to user with explanation
5. If N consecutive errors, restore from checkpoint and try fresh

**Priority:** HIGH
**Complexity:** MEDIUM

### GAP 8: No Intelligent Model Routing from Brain

**Current state:** The `NashRoutingOptimizer` computes optimal model-task assignment (game_theory.py). The `ModelSelectionWeights` track which models work best for which tasks (weights.py). But the `LLMRouter._select_model()` uses its own `_model_weights` dict (router.py line 203) which is never populated from the brain system.

**What's missing:**
- Bridge: feed Nash routing results into router's model selection
- Bridge: feed ModelSelectionWeights into router's scoring
- Dynamic routing: as the brain learns which model is best for coding vs. planning, the router should follow
- Cost-aware routing: consider API cost in routing decisions

**How it should work:**
1. After Nash iteration, update `router._model_weights` from Nash strategy
2. Before each `router.generate()`, optionally override model based on brain's learned preferences
3. Track cost per model per task type, include in routing score

**Priority:** MEDIUM
**Complexity:** MEDIUM

### GAP 9: No Proactive Tool Pre-Warming

**Current state:** `ProactivePredictionEngine.schedule_pre_warming_sync()` is called (line 419) but the method likely only logs or marks tools for pre-warming. No actual pre-warming (e.g., preparing tool state, loading models) occurs.

**What's missing:**
- Tool initialization: pre-load expensive tools before they're needed
- Model pre-warming: send keep-alive pings to prevent cold starts
- Context pre-computation: pre-build context window for predicted next turn
- Cache population: pre-fetch data the user is likely to need

**Priority:** LOW (optimization, not correctness)
**Complexity:** MEDIUM

### GAP 10: No Streaming Support for Tool Calls

**Current state:** `run_stream()` (lines 888-1061) does NOT support tool calls. It only streams text responses. If the LLM wants to call a tool during streaming, this is not handled.

**What's missing:**
- Tool call detection in stream chunks
- Tool execution mid-stream
- Resume streaming after tool execution
- Proper post-processing pipeline for streamed tool-using responses

**Priority:** MEDIUM
**Complexity:** HIGH

### GAP 11: Goal Tracker Not Integrated Into Decision Loop

**Current state:**
- GoalTracker is initialized on first message (line 367)
- `set_plan()` is never called (no planning)
- `verify_step()` runs after response is finalized (line 676)
- `recommended_action` is never checked
- `reset_loop_detection()` is never called after replanning (because replanning never happens)
- Goal drift is injected into brain state (BrainStateInjector) but only as informational text, not as a control signal

**What's missing:**
- Pre-response check: verify alignment BEFORE sending response to user
- Action on recommendations: actually replan/adjust/abort based on GoalTracker
- Plan integration: GoalTracker needs a plan to track progress against
- LLM-based verification: `llm_verify_fn` parameter is passed as None (line 676 - no llm_verify_fn)

**Priority:** HIGH
**Complexity:** MEDIUM

### GAP 12: No Memory Retrieval for Long Sessions

**Current state:** MemoryFabric stores working memory entries (line 688). But no memory is ever RETRIEVED and injected into the LLM context. Memory is write-only.

**What's missing:**
- Episodic memory retrieval: "What did we do last time we encountered this error?"
- Semantic memory search: find relevant past interactions
- Memory-augmented prompting: inject retrieved memories into context
- Cross-session persistence: remember user preferences across sessions

**Priority:** MEDIUM
**Complexity:** MEDIUM

### GAP 13: L2/L3 Summarization Never Executes

**Current state:** At line 862, the code checks `self._summarizer.should_summarize_l2(entry_count)` and logs a debug message, but never actually calls the LLM to generate a summary. L1 compression (observation masking) works, but L2 (LLM summarization) and L3 (structured digest) are placeholders.

**What's missing:**
- Call worker model to generate L2 summaries of old context
- Compact L3 digests for very old context
- Integrate summarized content back into CCE warm/cold tiers

**Priority:** MEDIUM (only matters for very long sessions)
**Complexity:** MEDIUM

---

## 5. Agentic Engine Design

### 5.1 Architecture Overview

The current `Session.run()` is a single-turn reactive processor. The agentic engine transforms it into a multi-step agentic engine with three execution modes:

1. **Chat Mode** (current behavior, improved) -- single turn, one message in, one response out
2. **Agent Mode** (new) -- multi-step autonomous execution toward a goal
3. **Streaming Agent Mode** (new) -- agent mode with real-time progress streaming

```
                     User Goal
                        |
                        v
                 [Planning Engine]
                   |         |
                   v         v
            [Plan Steps]  [Plan DAG]
                   |
        +----------+----------+
        |          |          |
        v          v          v
    [Step 1]   [Step 2]   [Step N]
        |          |          |
        v          v          v
  [Reflection] [Reflection] [Reflection]
        |          |          |
        v          v          v
  [Quality Gate]   ...      ...
        |
        v
  [Continue / Replan / Ask User / Done]
```

### 5.2 Planning Engine

**New file: `corteX/engine/planner.py`**

```python
class PlanStep:
    """A single step in an execution plan."""
    step_id: str
    description: str          # What to do
    expected_outcome: str     # What success looks like
    tools_needed: List[str]   # Which tools this step likely needs
    dependencies: List[str]   # step_ids that must complete first
    status: str               # pending, running, completed, failed, skipped
    retry_count: int
    max_retries: int

class ExecutionPlan:
    """A structured plan for achieving a goal."""
    goal: str
    steps: List[PlanStep]
    current_step_index: int
    created_at: float
    revised_count: int

class PlanningEngine:
    """Decomposes goals into executable plans."""

    async def create_plan(
        self,
        goal: str,
        available_tools: List[str],
        context: str,
        router: LLMRouter,
    ) -> ExecutionPlan:
        """Use orchestrator model to decompose goal into steps."""

    async def revise_plan(
        self,
        original_plan: ExecutionPlan,
        completed_steps: List[PlanStep],
        failure_reason: str,
        router: LLMRouter,
    ) -> ExecutionPlan:
        """Revise remaining plan based on what happened so far."""

    def get_next_step(self, plan: ExecutionPlan) -> Optional[PlanStep]:
        """Get the next executable step (respecting dependencies)."""
```

**Integration with Session.run():**
- On first message or `run_agentic()`, call `planner.create_plan()`
- Pass plan steps to `GoalTracker.set_plan()`
- Execute steps one at a time
- On GoalTracker "replan" recommendation, call `planner.revise_plan()`

### 5.3 Reflection Loop

**New file: `corteX/engine/reflection.py`**

```python
class ReflectionResult:
    """Result of self-verification."""
    passes_quality_gate: bool
    quality_score: float
    issues_found: List[str]
    suggested_fix: Optional[str]
    should_retry: bool
    retry_guidance: str

class ReflectionEngine:
    """Post-generation self-verification."""

    async def reflect(
        self,
        goal: str,
        step_description: str,
        response: str,
        tool_results: List[ToolExecutionResult],
        router: LLMRouter,
        quality_threshold: float = 0.5,
    ) -> ReflectionResult:
        """
        Use a different model (or the same model with verification prompt)
        to check if the response actually achieves the step goal.
        """

    async def suggest_improvement(
        self,
        response: str,
        issues: List[str],
        router: LLMRouter,
    ) -> str:
        """Generate an improved version of the response."""
```

**Integration with Session.run():**
- After line 593 (final assistant message), before post-processing
- If `reflection.passes_quality_gate` is False and retries remain, go back to B4
- Limit to 2 reflection rounds to prevent infinite loops

### 5.4 Context Window Manager

**Changes to Session.run():**

Replace raw message passing with CCE-managed context:

```python
# BEFORE (line 500):
messages=self._messages

# AFTER:
packed_context = self.context_engine.get_context_window(
    model_context_window=self._get_model_context_window(role)
)
messages = self._context_items_to_messages(packed_context)
```

**New method: `_context_items_to_messages()`**
```python
def _context_items_to_messages(self, items: List[ContextItem]) -> List[LLMMessage]:
    """Convert CCE packed items to LLMMessage list for router."""
    messages = []
    for item in items:
        if item.item_type == ContextItemType.USER_MESSAGE:
            messages.append(LLMMessage(role="user", content=item.content))
        elif item.item_type == ContextItemType.ASSISTANT_MESSAGE:
            messages.append(LLMMessage(role="assistant", content=item.content))
        elif item.item_type == ContextItemType.TOOL_RESULT:
            messages.append(LLMMessage(role="tool", content=item.content))
        elif item.item_type == ContextItemType.TASK_STATE:
            messages.append(LLMMessage(role="system", content=item.content))
        elif item.item_type == ContextItemType.SUMMARY:
            messages.append(LLMMessage(role="system", content=item.content))
    return messages
```

**L2 Summarization execution:**
```python
async def _execute_l2_summarization(self):
    """Call worker model to summarize old context."""
    items_to_summarize = self.context_engine.get_warm_items_for_summary()
    summary_prompt = self._build_summarization_prompt(items_to_summarize)
    summary = await self.agent.engine.router.generate(
        messages=[LLMMessage(role="user", content=summary_prompt)],
        role="worker",  # Use cheap model
    )
    self.context_engine.apply_l2_summary(summary.content, items_to_summarize)
```

### 5.5 Client Interaction Manager

**New file: `corteX/engine/interaction.py`**

```python
class InteractionType(str, Enum):
    INFORMATION = "information"     # "What file should I modify?"
    CONFIRMATION = "confirmation"   # "Should I delete this?"
    CHOICE = "choice"              # "Option A or B?"
    PERMISSION = "permission"      # "This requires admin access"

class InteractionRequest:
    """A question the agent needs answered."""
    interaction_type: InteractionType
    question: str
    options: Optional[List[str]]
    default_choice: Optional[str]
    timeout_seconds: float
    can_auto_decide: bool          # Enterprise config controls this
    auto_decision_prompt: str      # Prompt for LLM to decide autonomously

class InteractionManager:
    """Manages agent-to-user interactions with timeout."""

    async def ask_user(
        self,
        request: InteractionRequest,
        callback: Optional[Callable],
    ) -> str:
        """
        Ask the user a question.
        If user doesn't respond within timeout:
        1. If can_auto_decide, use LLM to predict user's choice
        2. If not, wait indefinitely (pause execution)
        """

    async def auto_decide(
        self,
        request: InteractionRequest,
        router: LLMRouter,
    ) -> str:
        """Use orchestrator model to predict what user would choose."""
```

**Integration with Agent Mode:**
- During plan execution, if agent encounters ambiguity
- Generate InteractionRequest
- Call `interaction_manager.ask_user()`
- Continue with response or auto-decision

### 5.6 Model Router

**Changes to LLMRouter:**

```python
class LLMRouter:
    def set_brain_weights(self, brain_weights: Dict[str, Dict[str, float]]):
        """
        Accept learned routing weights from the brain system.
        Called after Nash iteration or weight updates.
        """
        self._model_weights = brain_weights

    def _select_model(self, task_type, required_model, prefer_speed):
        # EXISTING: weight-based selection
        # NEW: also consult brain's model selection weights
        brain_scores = self._brain_weights.get(task_type, {})
        for ptype, provider in self._providers.items():
            for model_info in provider.available_models:
                # Blend brain score with existing score
                brain_score = brain_scores.get(model_info.model_id, 0.5)
                final_score = 0.6 * weight_score + 0.4 * brain_score
```

**Integration point:** After Nash iteration (line 838-845), call:
```python
self.agent.engine.router.set_brain_weights(
    self.weights.models.to_dict()
)
```

### 5.7 Error Recovery Engine

**New file: `corteX/engine/recovery.py`**

```python
class ErrorClass(str, Enum):
    TRANSIENT = "transient"      # Rate limit, network timeout
    PERMANENT = "permanent"      # Tool not found, invalid args
    CONTEXT = "context"          # Context corrupted, need checkpoint recovery
    FATAL = "fatal"              # No provider available, key revoked

class RecoveryStrategy:
    error_class: ErrorClass
    max_retries: int
    backoff_seconds: float
    alternative_model: Optional[str]
    should_replan: bool
    should_restore_checkpoint: bool

class RecoveryEngine:
    """Intelligent error recovery with backtracking."""

    def classify_error(self, error: Exception) -> ErrorClass:
        """Classify error type for recovery strategy selection."""

    def get_strategy(self, error_class: ErrorClass, retry_count: int) -> RecoveryStrategy:
        """Get recovery strategy based on error class and retry count."""

    async def execute_recovery(
        self,
        strategy: RecoveryStrategy,
        session: Session,
        failed_step: Optional[PlanStep],
    ) -> bool:
        """Execute recovery strategy. Returns True if recovered."""
```

### 5.8 Complete Agent Loop

Here is the redesigned `Session.run_agentic()`:

```python
async def run_agentic(
    self,
    goal: str,
    max_steps: int = 20,
    quality_threshold: float = 0.5,
    auto_decide_timeout: float = 30.0,
) -> Response:
    """
    Multi-step agentic execution toward a goal.
    """
    # PHASE 1: PLAN
    plan = await self.planner.create_plan(
        goal=goal,
        available_tools=[t.name for t in self.agent.tools],
        context=self._get_relevant_context(),
        router=self.agent.engine.router,
    )
    self._goal_tracker = GoalTracker(goal)
    self._goal_tracker.set_plan([s.description for s in plan.steps])

    steps_completed = 0
    all_results = []

    # PHASE 2: EXECUTE
    while steps_completed < max_steps:
        step = self.planner.get_next_step(plan)
        if step is None:
            break  # All steps completed

        # 2a. Predict
        pred = self.prediction.predict(
            action=step.description,
            tool=step.tools_needed[0] if step.tools_needed else "",
        )

        # 2b. Execute step (uses the current single-turn logic)
        step_response = await self._execute_single_step(step)

        # 2c. Reflect
        reflection = await self.reflection.reflect(
            goal=goal,
            step_description=step.description,
            response=step_response.content,
            tool_results=step_response.tool_results,
            router=self.agent.engine.router,
            quality_threshold=quality_threshold,
        )

        # 2d. Quality gate
        if not reflection.passes_quality_gate:
            if step.retry_count < step.max_retries:
                step.retry_count += 1
                continue  # Retry step

        # 2e. Verify goal alignment
        verification = await self._goal_tracker.verify_step(
            step_description=step.description,
            step_output=step_response.content[:500],
        )

        # 2f. Act on verification
        if verification.recommended_action == "replan":
            plan = await self.planner.revise_plan(
                original_plan=plan,
                completed_steps=all_results,
                failure_reason=verification.reasoning,
                router=self.agent.engine.router,
            )
            self._goal_tracker.reset_loop_detection()
            continue

        if verification.recommended_action == "abort":
            break

        # 2g. Record completion
        step.status = "completed"
        steps_completed += 1
        all_results.append(step_response)

        # 2h. Check if we need user input
        if self._needs_user_input(step_response):
            interaction = self._build_interaction_request(step_response)
            user_response = await self.interaction_manager.ask_user(
                interaction, timeout=auto_decide_timeout
            )
            # Incorporate user response into next step

    # PHASE 3: COMPILE
    return self._compile_agentic_response(all_results, plan)
```

---

## 6. Implementation Roadmap

### Phase 1: Critical Fixes (1-2 days)

| # | Task | Complexity | Impact |
|---|------|-----------|--------|
| 1.1 | **Fix follow-up LLM calls** -- pass brain params (temperature, top_p, etc.) to follow-up calls in tool loop (line 584) | LOW | HIGH |
| 1.2 | **Integrate CCE into LLM calls** -- replace `self._messages` with CCE-packed context | MEDIUM | CRITICAL |
| 1.3 | **Act on GoalTracker recommendations** -- check `recommended_action` before finalizing response | MEDIUM | HIGH |
| 1.4 | **Bridge brain weights to router** -- feed ModelSelectionWeights/Nash results into LLMRouter | MEDIUM | MEDIUM |

### Phase 2: Reflection & Recovery (3-5 days)

| # | Task | Complexity | Impact |
|---|------|-----------|--------|
| 2.1 | **Build ReflectionEngine** -- post-generation quality verification | MEDIUM | CRITICAL |
| 2.2 | **Build RecoveryEngine** -- error classification + retry strategies | MEDIUM | HIGH |
| 2.3 | **Integrate reflection loop** into Session.run() | MEDIUM | HIGH |
| 2.4 | **Execute L2/L3 summarization** -- wire up SummarizationPipeline to actually call LLM | MEDIUM | MEDIUM |

### Phase 3: Planning & Agent Mode (5-8 days)

| # | Task | Complexity | Impact |
|---|------|-----------|--------|
| 3.1 | **Build PlanningEngine** -- goal decomposition via LLM | HIGH | CRITICAL |
| 3.2 | **Build `run_agentic()`** -- multi-step execution loop | HIGH | CRITICAL |
| 3.3 | **Plan-GoalTracker integration** -- feed plan to tracker, replan on drift | MEDIUM | HIGH |
| 3.4 | **Memory retrieval** -- retrieve relevant memories for context augmentation | MEDIUM | MEDIUM |

### Phase 4: Interaction & Streaming (5-8 days)

| # | Task | Complexity | Impact |
|---|------|-----------|--------|
| 4.1 | **Build InteractionManager** -- user questions with timeout | HIGH | HIGH |
| 4.2 | **Autonomous decision-making** -- LLM-based prediction of user choice | MEDIUM | HIGH |
| 4.3 | **Streaming agent mode** -- real-time progress reporting during agentic execution | HIGH | MEDIUM |
| 4.4 | **Tool calls in streaming** -- handle tool execution during streamed responses | HIGH | MEDIUM |

### Summary of Current State

**What works well:**
- 20+ brain-inspired engine components (weights, prediction, feedback, plasticity, calibration, etc.)
- Brain state injection into LLM prompts
- Brain parameter resolution (cognitive state -> LLM params)
- Dual-process routing (System 1/2)
- Tool reputation with quarantine
- Game theory (Nash routing, Shapley attribution, minimax safety)
- Concept graph, cortical columns, attention, cross-modal associations
- Comprehensive post-processing pipeline (14+ recording/learning steps)

**What is fundamentally missing:**
1. **Planning** -- no task decomposition at all
2. **Reflection** -- no self-verification before sending response
3. **Multi-step execution** -- no autonomous agent loop
4. **Context management** -- CCE built but never used for actual LLM calls
5. **User interaction** -- no mechanism to ask user questions with timeout
6. **Error recovery** -- no systematic backtracking or retry
7. **Goal tracker action** -- verifies but never acts on recommendations

The SDK has world-class brain-inspired LEARNING infrastructure (weights, plasticity, prediction, calibration). What it lacks is the EXECUTION infrastructure that turns those learned signals into autonomous, goal-driven behavior. The brain can observe and learn, but it cannot yet plan, reflect, recover, or act autonomously.

---

*This document is the definitive reference for the Agentic Engine rebuild. All line numbers reference `corteX/sdk.py` as of 2026-02-14.*
