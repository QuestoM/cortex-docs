# Architecture

A detailed look at how `session.run()` processes a single user message through 14 steps.

---

## The 14-step pipeline

Every call to `session.run(message)` executes this sequence:

```text
User message
    |
    v
[1] Feedback processing + sensory adaptation
    |
[2] Goal tracker initialization (first turn only)
    |
[3] Context integration
    |--- [3b] Attentional filtering (P2)
    |--- [3c] Column selection (P2)
    |--- [3d] Resource allocation (P2)
    |--- [3e] Concept activation (P3)
    |--- [3f] Cross-modal enrichment (P1)
    |
[4] Prediction + proactive pre-warming
    |
[5] Dual-process routing (System 1/2)
    |--- [5b] Targeted modulation overrides (P3)
    |
[6] Tool filtering (reputation + modulation)
    |
[7] LLM generation
    |
[8] Tool execution loop (up to 5 rounds)
    |
[9] Final response assembly
    |
[10] Token tracking
    |
[11] Quality estimation + surprise comparison
    |
[12] Plasticity rules
    |
[13] Goal alignment verification
    |
[14] Memory + learning consolidation
    |--- [14b] Calibration recording
    |--- [14c] Proactive trajectory learning
    |--- [14d] Cross-modal Hebbian binding
    |--- [14e] Metacognition cycle (every 10 turns)
    |--- [14f] Column outcome recording
    |--- [14g] Resource map learning
    |--- [14h] Attention change detection
    |--- [14j] Concept graph co-occurrence
    |--- [14k] Map reorganization tracking
    |--- [14i] Periodic maintenance (every 25 turns)
    |
    v
Response
```

---

## Step details

### Step 1: Feedback processing

The `FeedbackEngine` scans the user message for implicit signals -- satisfaction, frustration, confusion, topic shifts. Each signal is passed through the `AdaptationFilter`, which habituates to repetitive patterns and amplifies novel ones.

### Step 2: Goal initialization

On the first turn, the `GoalTracker` captures the user's intent from the message. The `CorticalContextEngine` records this as the session goal for drift detection.

### Step 3: Context integration

The message is added to conversation history and the context engine. Then five subsystems process it in parallel:

- **Attention (P2):** Classifies message priority -- subconscious (routine), normal, or critical.
- **Columns (P2):** Selects the best specialized column for the task type (coding, debugging, research, etc.).
- **Resources (P2):** Allocates compute budget based on learned task-type distributions.
- **Concepts (P3):** Activates related concepts via spreading activation and returns recommendations.
- **Cross-modal (P1):** Retrieves learned associations from co-occurrence history and injects them as context hints.

### Step 4: Prediction

The `PredictionEngine` predicts the outcome (success probability, expected latency, expected quality) for this turn. The `ProactivePredictionEngine` predicts what the user will do *next* and pre-warms resources.

### Step 5: Dual-process routing

The `DualProcessRouter` decides between System 1 (fast/worker model) and System 2 (slow/orchestrator model) based on:

- Surprise magnitude from the prediction engine
- Population quality agreement
- Task novelty
- Enterprise safety level
- Whether an error occurred in the previous step
- Goal drift score

### Step 6: Tool filtering

Quarantined tools (those that have failed repeatedly) are removed from the available tool set. The `TargetedModulator` can additionally force-activate or silence specific tools.

### Step 7: LLM generation

The `LLMRouter` sends the conversation history to the selected model. Role selection combines signals from attention priority, dual-process routing, resource allocation, column recommendation, and concept graph suggestions.

### Step 8: Tool execution

If the LLM returns tool calls, each is executed through the `ToolExecutor`. Results are fed back into the LLM for up to 5 rounds. Every tool call is recorded in:

- `WeightEngine` (Bayesian success tracking)
- `ReputationSystem` (trust scoring)
- `ContinuousCalibrationEngine` (predicted vs actual success)
- `CorticalContextEngine` (context history)

### Steps 9--10: Response assembly

The final LLM response is appended to conversation history. Token usage is recorded in the context engine.

### Step 11: Quality estimation

The `PopulationQualityEstimator` evaluates response quality from multiple perspectives. The `PredictionEngine` compares the actual outcome against its earlier prediction, generating a surprise signal.

### Step 12: Plasticity

The `PlasticityManager` applies learning rules based on model performance, quality, and surprise:

- **Hebbian:** Strengthen weights for successful model/tool combinations.
- **Homeostatic:** Pull extreme weights back toward baseline.
- **Metaplasticity:** Adjust learning rates based on recent volatility.

### Step 13: Goal alignment

The `GoalTracker` verifies that the response moves toward the user's goal. It updates progress, drift score, and loop detection state.

### Step 14: Consolidation

All subsystems record the outcome and learn:

- Memory fabric stores the turn in working memory.
- Calibration records prediction accuracy.
- Proactive engine records the turn for trajectory learning.
- Cross-modal associator binds co-occurring items (Hebbian).
- Columns, resources, attention, concepts, and map reorganizer update their internal models.
- Every 10 turns: metacognition cycle checks calibration health and adjusts learning rates.
- Every 25 turns: maintenance runs decay, pruning, and reorganization.

---

## LLM routing (the thalamus)

The `LLMRouter` acts as a thalamus -- a relay that routes requests to the right model. It supports:

- **Role-based routing:** `orchestrator` (complex) vs `worker` (routine).
- **Multi-provider failover:** If one provider is down, requests fall through to the next.
- **Model overrides:** Set `orchestrator_model` and `worker_model` at engine creation.

```python
engine = cortex.Engine(
    providers={
        "openai": {"api_key": "sk-..."},
        "gemini": {"api_key": "AIza..."},
    },
    orchestrator_model="gpt-4o",
    worker_model="gemini-2.0-flash",
)
```

---

## Learning loops

corteX has three learning loops operating at different timescales:

| Loop | Timescale | Mechanism |
|---|---|---|
| **Within-turn** | Milliseconds | Tool execution results update Bayesian priors and reputation scores immediately. |
| **Within-session** | Minutes | Plasticity rules adjust weights after each turn. Calibration corrects confidence. Concepts and columns specialize. |
| **Cross-session** | Hours/days | Memory consolidation moves important patterns to long-term storage. Map reorganization redistributes territory. |

Each loop feeds into the next. A tool that fails within a turn reduces its reputation. Repeated failures across turns trigger quarantine. Cross-session consolidation can permanently reassign a tool's territory to a more reliable alternative.
