# Brain-to-Behavior Loop: Zero Dead Code Paths

## Overview

As of corteX v0.9, **every brain computation drives real agent decisions**. This is the key differentiator: while other agent frameworks (LangChain, CrewAI, AutoGen) have flat agent loops, corteX has a fully closed brain-to-behavior loop where 14 cognitive components actively control execution.

The [Brain-LLM Bridge](brain-llm-bridge.md) solves the *prompt gap* (brain state -> LLM parameters). The behavior loop solves the *execution gap* (brain state -> runtime decisions like "how many steps to take", "should we abort", "is this response good enough").

## The 14 Components

### Execution Control (Wave 118)

| Component | What It Controls | Before | After |
|-----------|-----------------|--------|-------|
| **AdaptiveBudget** | How many tool rounds per turn | Hardcoded `max_tool_rounds=5` | Velocity-based dynamic budgeting |
| **GoalTracker Enforcement** | Whether to abort or replan | Logged and ignored | Actually stops execution |
| **Weight Persistence** | Cross-session learning | Lost on restart | Save/load with 0.9 exponential decay |
| **Quality Gate** | Response quality threshold | All responses passed through | pass/warn/retry with automatic retry |

### Cognitive Enrichment (Wave 119)

| Component | What It Controls | Before | After |
|-----------|-----------------|--------|-------|
| **Spreading Activation** | Concept priming in prompts | Never called | Top 5 related concepts injected |
| **Resource Allocation** | Per-task budgets | Computed but unused | retry_budget floors tool rounds; verification_depth tightens quality |
| **Modulator CLAMP/AMPLIFY** | Behavioral weight modification | Adjustments computed, not applied | Directly modifies behavioral weights |
| **Proactive Prediction** | Model tier routing | Predictions unused | High-confidence complex tasks route to orchestrator tier |
| **Task Profiles** | Cross-session budget learning | Lost on restart | Exported/imported alongside weight files |
| **Enterprise Feedback** | Tier 3 rule evaluation | Context was None, rules never fired | Passes task_type and turn count |
| **Concept-Suggested Tools** | Tool selection bias | Tool recommendations ignored | Thompson Sampling alpha boosted for concept tools |
| **Cross-Modal Enrichment** | Multi-modal context in prompts | Truncated to 300 chars in user message | Full text in system prompt |
| **Surprise Routing** | Careful vs fast processing | High surprise not persisted | Sets flag forcing System 2 across turns |
| **Column Weight Consistency** | Brain snapshot accuracy | Snapshot used unmodified weights | Merges column overrides before injection |

## How It Works

### AdaptiveBudget: Dynamic Step Control

Instead of a fixed number of tool rounds, the brain computes a budget based on task complexity and historical velocity:

```python
# Before: hardcoded
max_tool_rounds = 5  # Same for trivial and complex tasks

# After: brain-controlled
budget = self._adaptive_budget.compute(task_type, complexity)
max_tool_rounds = max(budget.steps, resource_alloc.retry_budget)
# Simple lookup: 2 steps. Complex multi-step: 15+ steps.
```

Task profiles persist across sessions, so the budget learns from experience. If a "code review" task typically completes in 3 steps, the budget adapts.

### GoalTracker Enforcement: Real Abort/Replan

The GoalTracker's `verify_step()` produces recommendations (abort, replan, continue). These now actually control execution:

```python
# In the chat loop, BEFORE recompilation:
if self._goal_abort_requested:
    break  # Stop immediately

if self._goal_replan_requested:
    self._planning_engine.replan()  # Decompose again
    self._goal_replan_requested = False
```

When goal drift exceeds 0.5, aggressive weight bumps (0.3 instead of 0.15) push the agent back on track. Critical drift triggers `[GOAL DRIFT CRITICAL]` warnings in the brain snapshot.

### Quality Gate: Response Verification

Every LLM response passes through a quality gate before delivery:

```python
verdict = self._evaluate_quality(response, context)
# verdict.action: "pass" (>= 0.6), "warn" (0.4-0.6), "retry" (< 0.4)

if verdict.action == "retry" and retries < 1:
    response = await self._retry_with_feedback(verdict.feedback)
```

The gate caps retries at 1 to prevent infinite loops. The `verification_depth` from ResourceHomunculus tightens the threshold for important tasks.

### Weight Persistence: Synaptic Homeostasis

Weights save to `~/.cortex/weights/<agent>.json` with schema versioning:

```python
# On session close
weight_persistence.save(agent_name, weights.snapshot())

# On session start
if data := weight_persistence.load(agent_name):
    weights.load_merged(data)  # 0.9 decay factor
```

The 0.9 exponential decay prevents weights from drifting too far from baseline over many sessions - this is analogous to synaptic homeostasis in biological neural networks.

### Spreading Activation: Concept Priming

When a task involves certain concepts, the ConceptGraph performs BFS to find related concepts and injects them into the system prompt:

```python
# In prepare.py step 3e1
activated = concept_graph.spreading_activation(seed_concepts, depth=2)
top_concepts = sorted(activated, key=lambda c: c.activation, reverse=True)[:5]

# Injected into system prompt via prepare_finalize.py
"Primed concepts: [security, vulnerability, CVE, compliance, incident-response]"
```

This gives the LLM awareness of related domains it might not otherwise consider.

### Surprise -> System 2 Routing

High surprise (> 0.6) triggers a persistent flag that forces careful processing:

```python
# In learning_signals.py
if surprise > 0.6:
    self.weights.behavioral.weights["_high_surprise_flag"] = 1.0

# In prepare.py, dual-process routing
if weights.get("_high_surprise_flag", 0) > 0 or live_surprise > 0.6:
    process_type = ProcessType.SYSTEM2  # Force careful processing
```

The flag persists across turns until surprise drops below the threshold, ensuring the agent stays careful through an entire unexpected situation.

## Verification

The full brain pipeline is tested end-to-end:

```bash
pytest tests/test_brain_pipeline_e2e.py -v
```

This test simulates a multi-turn conversation exercising all 14 components and verifying each one influences the next turn's behavior.

## Why This Matters

No other agent SDK has a brain that actually drives decisions:

- **LangChain**: Flat prompt -> LLM -> tool loop. No adaptive budgets, no goal enforcement, no quality gates.
- **CrewAI**: Role-based agents but no cognitive state. No learning across sessions.
- **AutoGen**: Multi-agent conversation but no brain-controlled execution parameters.

corteX's closed brain-to-behavior loop means agents genuinely get smarter over time, abort when they detect drift, retry low-quality responses, and allocate resources proportional to task complexity - all automatically, without developer intervention.
