# IntegrationBrain API

`IntegrationBrain` is the core class that powers all framework integrations. If you want to add corteX intelligence to a framework that doesn't have a built-in wrapper (e.g., AutoGen, Haystack, your own custom framework), use `IntegrationBrain` directly.

## Installation

```bash
pip install cortex-ai
```

No extra dependencies needed - `IntegrationBrain` is part of the core package.

## Quick Start

```python
from corteX.integrations.brain import IntegrationBrain

# Create a brain (defaults to full mode with 30+ components)
brain = IntegrationBrain(goal="Process customer refund requests")

# Feed events from your framework
await brain.on_step("Look up customer account", "Found account #456")
brain.on_tool_call("search_db", {"query": "customer"}, "Found 1 result", success=True)
brain.on_feedback("Thanks, that was helpful!")

# Read brain state
state = brain.get_brain_state()
print(state["goal_progress"])
print(state["drift_score"])
print(state["full_components"])  # List of 30+ active components
```

## Architecture

`IntegrationBrain` sits between your framework and the corteX brain engine:

```
Your Framework
     |
     | on_step(), on_tool_call(), on_feedback(), on_error()
     v
IntegrationBrain
     |
     +-- WeightEngine (synaptic weights, tool preferences)
     +-- GoalTracker (goal DNA, drift detection, loop detection)
     +-- FeedbackEngine (implicit signal detection)
     +-- [Full mode] 30+ additional brain components
```

### Lite vs Full Mode

```python
# Full mode (default) - all 30+ brain components
brain = IntegrationBrain(goal="...", mode="full")

# Lite mode - 3 core components only (faster startup)
brain = IntegrationBrain(goal="...", mode="lite")
```

Both modes are free. Full mode lazily imports additional components on first use.

## Event Handlers

These are the methods you call to feed events from your framework into the brain.

### `on_step(description, output)` - async

Record a step (action + result) from your framework.

```python
metrics = await brain.on_step(
    description="Search knowledge base for billing FAQ",
    output="Found 3 relevant articles about refund policies",
)

print(metrics["goal_progress"])   # Updated progress
print(metrics["drift_score"])     # Updated drift
print(metrics["step"])            # Step number (1, 2, 3...)
```

In full mode, this feeds the step through plasticity, concept graphs, context pinning, memory, and attention components.

### `on_tool_call(tool_name, args, result, *, success, latency_ms)`

Record a tool invocation.

```python
brain.on_tool_call(
    tool_name="search_db",
    args={"query": "customer 456"},
    result="Found: John Doe, Premium plan",
    success=True,
    latency_ms=45.2,
)
```

This updates:

- **Weight engine** - registers the tool and records success/failure for Bayesian learning
- **Bayesian selector** (full mode) - updates Thompson Sampling posteriors
- **Tool reputation** (full mode) - tracks reliability scores
- **Plasticity** (full mode) - adjusts learning rate

### `on_feedback(message)`

Record user feedback. The FeedbackEngine detects implicit signals (praise, frustration, correction) and updates weights automatically.

```python
brain.on_feedback("Thanks, that solved my issue!")
# Detects satisfaction signal, reinforces current behavior

brain.on_feedback("No, that's wrong. I asked about billing, not shipping.")
# Detects correction signal, adjusts weights
```

### `on_error(error)`

Record an error from your framework.

```python
brain.on_error("Connection refused: database timeout")
```

In full mode, this feeds into the degradation chain for graceful fallback tracking.

### `reset()`

Reset per-run tracking (step count, tool calls, errors) while keeping learned state (weights, memory).

```python
brain.reset()
# Step count, tool calls, and errors are cleared
# Weights, goal tracker state, and memory are preserved
```

## State Accessors

### `get_brain_state()` - Full Snapshot

Returns a comprehensive dict with all brain metrics:

```python
state = brain.get_brain_state()

# Always available (lite + full):
state["mode"]              # "lite" or "full"
state["weights"]           # WeightEngine snapshot
state["goal_progress"]     # 0.0 to 1.0
state["drift_score"]       # 0.0 to 1.0
state["drift_exceeded"]    # True if drift > threshold
state["loop_count"]        # Number of detected loops
state["loop_detected"]     # True if any loops detected
state["steps_completed"]   # Total steps this run
state["tool_calls"]        # Total tool invocations this run
state["errors"]            # Total errors this run
state["feedback_summary"]  # Signal summary from FeedbackEngine

# Full mode only:
state["full_components"]   # List of active component names
state["prediction"]        # Prediction engine state
state["plasticity"]        # Plasticity metrics
state["reputation"]        # Tool reputation scores
state["calibration"]       # Metacognitive calibration
state["degradation"]       # Degradation chain state
# ... and more
```

### `get_predictions()` - Prediction Engine

```python
preds = brain.get_predictions()

if preds["available"]:  # True in full mode
    print(preds["next_action"])   # Predicted next step
    print(preds["confidence"])    # Prediction confidence
```

### `get_component(name)` - Direct Access

Access any brain component by name:

```python
# Core components
brain.get_component("weights")        # WeightEngine
brain.get_component("goal_tracker")   # GoalTracker
brain.get_component("feedback")       # FeedbackEngine

# Full mode components
brain.get_component("prediction")     # PredictionEngine
brain.get_component("reputation")     # ToolReputation
brain.get_component("bayesian_selector")  # BayesianToolSelector
brain.get_component("plasticity")     # PlasticityManager
brain.get_component("calibration")    # CalibrationTracker
brain.get_component("bias_detector")  # BiasDetector
brain.get_component("context_pinner") # ContextPinner
brain.get_component("degradation")    # DegradationChain

# Returns None if component doesn't exist
brain.get_component("nonexistent")    # None
```

### Properties

```python
brain.mode              # "lite" or "full"
brain.is_full           # True if full mode active
brain.goal              # The goal string
brain.loop_detected     # Whether loops were detected
brain.drift_score       # Current drift (0.0 to 1.0)
brain.drift_exceeded    # True if drift > threshold
brain.goal_progress     # Progress toward goal (0.0 to 1.0)

# Direct access to core components
brain.weights           # WeightEngine instance
brain.goal_tracker      # GoalTracker instance
brain.feedback          # FeedbackEngine instance
```

## Building a Custom Integration

Here's a template for wrapping any AI framework with corteX:

```python
from corteX.integrations.brain import IntegrationBrain

class CortexMyFrameworkBrain:
    def __init__(self, framework_instance, goal: str, brain_mode: str = "full"):
        self._framework = framework_instance
        self._brain = IntegrationBrain(goal=goal, mode=brain_mode)

    async def run(self, input_text: str):
        self._brain.reset()

        # 1. Register goal
        self._brain.goal_tracker.set_plan([input_text])

        # 2. Run your framework (capture events)
        result = await self._framework.execute(input_text)

        # 3. Feed events into the brain
        for step in result.steps:
            await self._brain.on_step(step.description, step.output)

        for tool_call in result.tool_calls:
            self._brain.on_tool_call(
                tool_call.name, tool_call.args, tool_call.result,
                success=tool_call.success,
                latency_ms=tool_call.latency_ms,
            )

        # 4. Verify against goal
        await self._brain.goal_tracker.verify_step(
            step_description=input_text,
            step_output=str(result.output)[:1000],
        )

        # 5. Return enriched result
        return {
            "output": result.output,
            "brain_state": self._brain.get_brain_state(),
            "goal_progress": self._brain.goal_progress,
            "drift_score": self._brain.drift_score,
            "loop_detected": self._brain.loop_detected,
        }

    @property
    def brain(self):
        return self._brain
```

## Full Component List (Full Mode)

| Component | Key | Purpose |
|-----------|-----|---------|
| Prediction Engine | `prediction` | Anticipates next steps, detects surprises |
| Plasticity Manager | `plasticity` | Adjusts learning rate based on performance |
| Adaptation Engine | `adaptation` | Sensory adaptation to input patterns |
| Memory Fabric | `memory` | Cross-session learning and recall |
| Quality Estimator | `quality_estimator` | Estimates output quality |
| Dual Process | `dual_process` | Fast/slow thinking routing |
| Tool Reputation | `reputation` | Per-tool reliability tracking |
| Bayesian Selector | `bayesian_selector` | Thompson Sampling tool selection |
| Anchor Manager | `anchor_manager` | Anchoring bias management |
| Nash Optimizer | `nash_optimizer` | Game-theoretic optimization |
| Shapley Attributor | `shapley_attributor` | Contribution attribution |
| Attention Filter | `attention` | Context relevance filtering |
| Calibration Tracker | `calibration` | Metacognitive confidence |
| Content Predictor | `content_predictor` | Content-aware predictions |
| Proactive Engine | `proactive` | Proactive suggestion generation |
| Signal Injector | `signal_injector` | Brain signal injection |
| Signal Parser | `signal_parser` | Response signal extraction |
| Signal Aggregator | `signal_aggregator` | Multi-signal aggregation |
| Cortical Columns | `columns` | Hierarchical feature processing |
| Concept Graphs | `concepts` | Concept relationship mapping |
| Context Engine | `context_engine` | Cortical context management |
| Context Summarizer | `summarizer` | Conversation summarization |
| Cross-Modal | `cross_modal` | Cross-modal association |
| Resource Map | `resource_map` | Resource allocation |
| Map Reorganizer | `reorganizer` | Dynamic map reorganization |
| Modulator | `modulator` | Targeted behavior modulation |
| Simulator | `simulator` | What-if simulation |
| Semantic Scorer | `semantic_scorer` | Semantic similarity scoring |
| Context Pinner | `context_pinner` | Critical instruction reinforcement |
| Degradation Chain | `degradation` | Graceful fallback on errors |
| Span Tracker | `span_tracker` | Context utilization tracking |
| Bias Detector | `bias_detector` | LLM default override detection |
| Brain Injector | `brain_injector` | Brain state injection into prompts |
| Audit Logger | `audit` | Action audit trail |
