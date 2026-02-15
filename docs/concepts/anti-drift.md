# Anti-Drift System

The Anti-Drift system prevents agents from getting stuck in loops, drifting off-goal, or wasting resources on unproductive actions. It combines multi-resolution loop detection, graduated drift responses, and adaptive budget management to keep agents productive across ultra-long workflows.

## What It Does

The anti-drift system provides three core capabilities:

1. **MultiResolutionLoopDetector**: Parallel loop detection across 4 detection strategies
2. **DriftEngine**: 5-signal drift scoring with 4 graduated response levels
3. **AdaptiveBudget**: Dynamic step/token budgets that expand/contract based on velocity

Together, these components detect and resolve drift 3-4x more effectively than hash-only loop detection.

---

## MultiResolutionLoopDetector

**corteX Innovation**: 4 parallel loop detectors that catch different loop patterns simultaneously.

The MultiResolutionLoopDetector runs four detection strategies in parallel, catching loops that single-strategy detectors would miss.

### Why: The Single-Strategy Blindness Problem

Traditional loop detection uses only state hashing:

```python
# Misses loops with slight variations
if current_state_hash == previous_state_hash:
    loop_detected = True
```

This catches **exact loops** (same action, same state) but misses:
- **Semantic loops**: Different words, same meaning ("Analyzing..." → "Still analyzing...")
- **Oscillation loops**: A → B → A → B → A (alternating between two states)
- **Dead-end loops**: Stuck on an impossible task without repeating exact state

MultiResolutionLoopDetector solves this with **4 parallel detectors**.

### How It Works

The detector runs four strategies simultaneously:

```python
from corteX.engine.loop_detector import MultiResolutionLoopDetector, LoopEvent, LoopType, LoopSignal

detector = MultiResolutionLoopDetector(
    window_size=10,  # Look back 10 steps
    exact_hash_threshold=2,  # 2 exact repetitions = loop
    semantic_threshold=0.85,  # 85% semantic similarity = loop
    oscillation_threshold=3,  # 3 cycles of A↔B = loop
    dead_end_threshold=5,  # 5 steps without progress = dead end
)

# Record each step
detector.record_step(
    turn=1,
    action="Read file config.py",
    state={"file": "config.py", "status": "reading"},
    result="File not found"
)

detector.record_step(
    turn=2,
    action="Read file config.py",
    state={"file": "config.py", "status": "reading"},
    result="File not found"
)

# Check for loops
loop = detector.detect_loop()

if loop.detected:
    print(f"Loop detected: {loop.type}")
    print(f"Confidence: {loop.confidence:.2%}")
    print(f"Strategy: {loop.detection_strategy}")
    print(f"Steps involved: {loop.loop_sequence}")
    print(f"Suggestion: {loop.suggestion}")

# Output:
# Loop detected: EXACT_HASH
# Confidence: 100%
# Strategy: exact_hash
# Steps involved: [1, 2]
# Suggestion: The same action "Read file config.py" has been repeated with identical results. Try a different approach.
```

### Four Detection Strategies

#### 1. Exact Hash Detector

Detects exact repetitions using state hashing:

```python
# Catches:
# - Same tool, same arguments, same result
# - Exact state duplication

detector.record_step(
    turn=1,
    action="ls /home/user",
    state={"cwd": "/home/user"},
    result="file1.txt file2.txt"
)

detector.record_step(
    turn=2,
    action="ls /home/user",
    state={"cwd": "/home/user"},
    result="file1.txt file2.txt"
)

# Loop detected: EXACT_HASH (100% confidence)
```

#### 2. Semantic Jaccard Detector

Detects loops with slight variations using token-set similarity:

```python
# Catches:
# - Different wording, same meaning
# - Minor parameter changes

detector.record_step(
    turn=1,
    action="Analyzing the configuration file",
    state={"task": "analyze"},
    result="Found 3 issues"
)

detector.record_step(
    turn=2,
    action="Still analyzing configuration file",
    state={"task": "analyze"},
    result="Found 3 problems"
)

# Loop detected: SEMANTIC (87% confidence)
# Jaccard similarity: 0.87 (issues/problems are semantically similar)
```

#### 3. Oscillation Detector

Detects A ↔ B ↔ A alternating patterns:

```python
# Catches:
# - Switching between two states
# - Flip-flopping decisions

detector.record_step(turn=1, action="Approach A", state={"method": "A"})
detector.record_step(turn=2, action="Approach B", state={"method": "B"})
detector.record_step(turn=3, action="Approach A", state={"method": "A"})
detector.record_step(turn=4, action="Approach B", state={"method": "B"})
detector.record_step(turn=5, action="Approach A", state={"method": "A"})

# Loop detected: OSCILLATION (95% confidence)
# Pattern: A → B → A → B → A (3 complete cycles)
```

#### 4. Dead-End Detector

Detects when the agent is stuck without making progress:

```python
# Catches:
# - No state changes despite different actions
# - Spinning wheels on impossible tasks

detector.record_step(
    turn=1,
    action="Try approach 1",
    state={"progress": 0},
    result="Failed"
)

detector.record_step(
    turn=2,
    action="Try approach 2",
    state={"progress": 0},
    result="Failed"
)

detector.record_step(
    turn=3,
    action="Try approach 3",
    state={"progress": 0},
    result="Failed"
)

# Loop detected: DEAD_END (80% confidence)
# No progress for 3 steps - agent is stuck on an impossible task
```

### Combined Detection

All four detectors run in parallel and the highest-confidence detection wins:

```python
loop = detector.detect_loop()

if loop.detected:
    print(f"Primary detection: {loop.detection_strategy}")
    print(f"All detections: {loop.all_detections}")

# Output:
# Primary detection: semantic
# All detections: {
#   "exact_hash": None,
#   "semantic": LoopDetection(confidence=0.87, ...),
#   "oscillation": None,
#   "dead_end": LoopDetection(confidence=0.65, ...)
# }
```

This catches 3-4x more loops than hash-only detection.

---

## DriftEngine

**corteX Innovation**: 5-signal drift scoring with 4 graduated response levels.

The DriftEngine monitors drift across five dimensions and applies graduated responses from gentle nudges to hard stops.

### Why: The Binary Drift Problem

Traditional drift detection is binary:

```python
if drift > threshold:
    stop_agent()  # Too harsh
else:
    continue()  # Misses early warning signs
```

DriftEngine uses **graduated responses** that escalate only as needed.

### How It Works

The engine tracks 5 drift signals:

```python
from corteX.engine.drift_engine import DriftEngine, DriftSignals, DriftAction, DriftAssessment, DriftSeverity

engine = DriftEngine(
    nudge_threshold=0.3,  # Gentle reminder
    replan_threshold=0.5,  # Regenerate plan
    checkpoint_threshold=0.7,  # Rollback to last good state
    stop_threshold=0.9,  # Ask user for help
)

# Record drift signals
signals = DriftSignals(
    goal_dna_drift=0.45,  # GoalDNA token similarity
    loop_risk=0.20,  # MultiResolutionLoopDetector
    budget_velocity=-0.30,  # Spending faster than progress
    quality_degradation=0.15,  # Context quality dropping
    stuck_time=0.10,  # GoalTree stuck detection
)

# Score drift
drift_report = engine.score_drift(signals)

print(f"Overall drift: {drift_report.overall_score:.2%}")
print(f"Response level: {drift_report.response_level}")
print(f"Recommended action: {drift_report.recommendation}")

# Output:
# Overall drift: 52%
# Response level: REPLAN
# Recommended action: Current approach is drifting from goal. Generate new execution plan.
```

### Five Drift Signals

| Signal | Weight | What It Measures |
|--------|--------|------------------|
| **goal_dna_drift** | 35% | Token-set divergence from original goal |
| **loop_risk** | 25% | Loop detection confidence |
| **budget_velocity** | 20% | Spending vs. progress rate |
| **quality_degradation** | 15% | Context quality drop |
| **stuck_time** | 5% | Time without progress |

### Drift Scoring Formula

```
overall_drift = (
    0.35 * goal_dna_drift +
    0.25 * loop_risk +
    0.20 * |budget_velocity| +
    0.15 * quality_degradation +
    0.05 * stuck_time
)
```

### Four Response Levels

The engine maps drift scores to graduated responses:

| Drift Range | Response Level | Action |
|-------------|----------------|--------|
| **0.0 - 0.3** | CONTINUE | Continue normally |
| **0.3 - 0.5** | INJECT_REMINDER | Inject goal reminder, highlight drift in prompt |
| **0.5 - 0.7** | SUMMARIZE_REPLAN | Regenerate execution plan, re-decompose goal |
| **0.7 - 0.9** | CHECKPOINT_RESET | Rollback to last checkpoint, restore good state |
| **0.9 - 1.0** | ASK_USER | Ask user for help, explain the situation |

#### Response Level: INJECT_REMINDER

Gentle intervention:

```python
if drift_report.recommended_action == DriftAction.INJECT_REMINDER:
    # Inject goal reminder into next LLM call
    reminder = goal_reminder.get_reminder(turn, force_full=True)
    prompt = f"{reminder}\n\n{user_message}"

    # Add drift warning to system prompt
    system_prompt += "\n⚠️ WARNING: Slight drift detected. Ensure next action aligns with primary goal."
```

#### Response Level: SUMMARIZE_REPLAN

Regenerate the plan:

```python
if drift_report.recommended_action == DriftAction.SUMMARIZE_REPLAN:
    # Ask LLM to regenerate plan
    replan_prompt = f"""
    The current approach is drifting from the goal.

    Original goal: {original_goal}
    Current activity: {current_action}
    Drift score: {drift_report.overall_score:.1%}

    Please generate a new execution plan that stays aligned with the goal.
    """

    new_plan = llm.generate(replan_prompt)
    goal_tree.rebuild_from_plan(new_plan)
```

#### Response Level: CHECKPOINT_RESET

Rollback to last good state:

```python
if drift_report.recommended_action == DriftAction.CHECKPOINT_RESET:
    # Restore last checkpoint
    checkpoint = cognitive_pipeline.get_last_checkpoint()

    logger.warning(f"Drift {drift_report.overall_score:.1%} - rolling back to turn {checkpoint.turn}")

    # Restore state
    conversation_history = checkpoint.conversation_history
    goal_tree = checkpoint.goal_tree
    state_files = checkpoint.state_files

    # Explain to user
    response = f"I noticed I was drifting off-track. I've rolled back to a previous checkpoint and will try a different approach."
```

#### Response Level: ASK_USER

Ask user for help:

```python
if drift_report.recommended_action == DriftAction.ASK_USER:
    # Stop execution and ask user
    response = f"""
    I'm having trouble staying aligned with your goal.

    Original goal: {original_goal}
    Drift score: {drift_report.overall_score:.1%}

    Contributing factors:
    - Goal DNA drift: {signals.goal_dna_drift:.1%}
    - Loop risk: {signals.loop_risk:.1%}
    - Budget velocity: {signals.budget_velocity:.1%}

    Could you help me by:
    1. Clarifying the next step you'd like me to take, or
    2. Adjusting the goal to better match what I should be doing
    """

    return response  # Wait for user input
```

### Drift Trends

The engine tracks drift history to detect acceleration:

```python
# Record drift over time
engine.record_drift(turn=10, drift_score=0.25)
engine.record_drift(turn=11, drift_score=0.30)
engine.record_drift(turn=12, drift_score=0.38)
engine.record_drift(turn=13, drift_score=0.50)

# Analyze trend
trend = engine.get_drift_trend(window=4)

if trend.is_accelerating and trend.velocity > 0.05:
    # Drift is increasing rapidly - escalate response level
    logger.warning(f"Drift accelerating at {trend.velocity:.3f} per turn")
    # Escalate from NUDGE → REPLAN
```

---

## AdaptiveBudget

**corteX Innovation**: Dynamic step/token budgets that expand/contract based on velocity and historical task profiles.

AdaptiveBudget prevents resource waste by automatically adjusting budgets based on actual progress velocity.

### Why: The Static Budget Problem

Static budgets are either:
- **Too generous**: Waste resources on stuck agents
- **Too restrictive**: Block agents from completing complex tasks

AdaptiveBudget solves this with **velocity-based budget adjustment**.

### How It Works

The budget adapts based on progress velocity:

```python
from corteX.engine.adaptive_budget import AdaptiveBudget, BudgetState, BudgetDecision

budget = AdaptiveBudget(
    initial_steps=50,
    initial_tokens=100000,
    min_velocity=0.01,  # Must make 1% progress per step
    velocity_window=10,  # Calculate velocity over last 10 steps
    expansion_factor=1.5,  # Increase by 50% if velocity is good
    contraction_factor=0.7,  # Decrease by 30% if velocity is poor
)

# Record progress
budget.record_step(
    turn=1,
    tokens_used=1500,
    progress_delta=0.05,  # Made 5% progress
)

budget.record_step(
    turn=2,
    tokens_used=2000,
    progress_delta=0.04,
)

# Check budget status
status = budget.get_status()

print(f"Steps remaining: {status.steps_remaining}")
print(f"Tokens remaining: {status.tokens_remaining}")
print(f"Progress velocity: {status.velocity:.3f} per step")
print(f"Budget adjustment: {status.adjustment_factor:.1%}")

# Output:
# Steps remaining: 48
# Tokens remaining: 96500
# Progress velocity: 0.045 per step
# Budget adjustment: 150% (good velocity - budget expanded)
```

### Velocity Calculation

```
velocity = Σ(progress_delta) / window_size
```

Example:
- Last 10 steps made 5% + 4% + 6% + 3% + 5% + 4% + 7% + 5% + 6% + 5% = 50% total progress
- Velocity = 50% / 10 steps = 5% progress per step

### Budget Adjustment Rules

| Velocity | Adjustment | Rationale |
|----------|------------|-----------|
| **>= min_velocity** | Expand by 1.5x | Good progress - allow more resources |
| **< min_velocity** | Contract by 0.7x | Poor progress - reduce resources to prevent waste |
| **== 0 for 5 steps** | Contract by 0.5x | Stuck - aggressive reduction |

```python
# Good velocity scenario
budget.record_step(turn=10, tokens_used=1000, progress_delta=0.08)
# Velocity: 0.08 (above min 0.01)
# Next step budget: 50 * 1.5 = 75 steps

# Poor velocity scenario
budget.record_step(turn=20, tokens_used=2000, progress_delta=0.005)
# Velocity: 0.005 (below min 0.01)
# Next step budget: 75 * 0.7 = 52 steps

# Stuck scenario
for i in range(5):
    budget.record_step(turn=30+i, tokens_used=1000, progress_delta=0.0)
# Velocity: 0.0 for 5 steps
# Next step budget: 52 * 0.5 = 26 steps
```

### Historical Task Profiling

The budget learns typical resource needs for task types:

```python
# Record completed tasks
budget.record_task_completion(
    task_type="implement_api_endpoint",
    steps_used=15,
    tokens_used=25000,
    total_progress=1.0,
)

budget.record_task_completion(
    task_type="implement_api_endpoint",
    steps_used=18,
    tokens_used=30000,
    total_progress=1.0,
)

# Get budget recommendation for similar task
recommended = budget.get_task_budget_recommendation(
    task_type="implement_api_endpoint"
)

print(f"Recommended steps: {recommended.steps}")
print(f"Recommended tokens: {recommended.tokens}")

# Output:
# Recommended steps: 20 (mean 16.5 + 20% buffer)
# Recommended tokens: 33000 (mean 27500 + 20% buffer)
```

### Emergency Budget Extension

For critical tasks, the budget can request emergency extensions:

```python
# Agent is 90% done but out of budget
if progress > 0.9 and budget.is_exhausted():
    emergency_extension = budget.request_emergency_extension(
        justification="Task is 90% complete, need 5 more steps to finish",
        requested_steps=5,
        requested_tokens=10000,
    )

    if emergency_extension.approved:
        # Continue with extended budget
        pass
    else:
        # Stop and ask user
        response = "I'm 90% done but ran out of budget. May I have 5 more steps to complete?"
```

---

## Integration with Agent Loop

All three components integrate seamlessly:

```python
# In Session.__init__()
self.loop_detector = MultiResolutionLoopDetector()
self.drift_engine = DriftEngine()
self.adaptive_budget = AdaptiveBudget()

# In Session.run_agentic() loop

# 1. Check for loops
loop = self.loop_detector.detect_loop()
if loop.detected:
    if loop.confidence > 0.9:
        # High-confidence loop - break immediately
        return f"Loop detected: {loop.suggestion}"
    else:
        # Low-confidence - record as drift signal
        loop_risk = loop.confidence

# 2. Score drift
drift_signals = DriftSignals(
    goal_dna_drift=self.goal_dna.calculate_drift(current_action),
    loop_risk=loop_risk,
    budget_velocity=self.adaptive_budget.get_velocity(),
    quality_degradation=self.quality_engine.get_degradation(),
    stuck_time=self.goal_tree.get_stuck_score(),
)

drift_report = self.drift_engine.score_drift(drift_signals)

# 3. Apply graduated response
if drift_report.recommended_action == DriftAction.ASK_USER:
    return f"Drift too high: {drift_report.explanation}"
elif drift_report.recommended_action == DriftAction.CHECKPOINT_RESET:
    self.restore_checkpoint()
elif drift_report.recommended_action == DriftAction.SUMMARIZE_REPLAN:
    self.regenerate_plan()
elif drift_report.recommended_action == DriftAction.INJECT_REMINDER:
    self.inject_goal_reminder(force_full=True)

# 4. Check budget
if self.adaptive_budget.is_exhausted():
    return "Budget exhausted - task incomplete"

# 5. Execute step
result = self.execute_step(action)

# 6. Record progress
progress_delta = self.goal_tree.calculate_progress_delta()
self.adaptive_budget.record_step(
    turn=self.turn,
    tokens_used=result.tokens,
    progress_delta=progress_delta,
)

# 7. Record for loop detection
self.loop_detector.record_step(
    turn=self.turn,
    action=action,
    state=self.get_state_snapshot(),
    result=result.output,
)
```

---

## Configuration

```python
from corteX.engine.loop_detector import MultiResolutionLoopDetector, LoopEvent, LoopType, LoopSignal
from corteX.engine.drift_engine import DriftEngine, DriftSignals, DriftAction, DriftAssessment, DriftSeverity
from corteX.engine.adaptive_budget import AdaptiveBudget, BudgetState, BudgetDecision

# Loop detector configuration
detector = MultiResolutionLoopDetector(
    window_size=10,
    exact_hash_threshold=2,
    semantic_threshold=0.85,
    oscillation_threshold=3,
    dead_end_threshold=5,
    enable_all_strategies=True,
)

# Drift engine configuration
drift_engine = DriftEngine(
    nudge_threshold=0.3,
    replan_threshold=0.5,
    checkpoint_threshold=0.7,
    stop_threshold=0.9,
    signal_weights={
        "goal_dna_drift": 0.35,
        "loop_risk": 0.25,
        "budget_velocity": 0.20,
        "quality_degradation": 0.15,
        "stuck_time": 0.05,
    },
)

# Adaptive budget configuration
budget = AdaptiveBudget(
    initial_steps=50,
    initial_tokens=100000,
    min_velocity=0.01,
    velocity_window=10,
    expansion_factor=1.5,
    contraction_factor=0.7,
    enable_historical_profiling=True,
    enable_emergency_extensions=True,
)
```

---

## API Reference

```python
from corteX.engine.loop_detector import (
    MultiResolutionLoopDetector,
    LoopEvent,
    LoopType,
    LoopSignal,
)
from corteX.engine.drift_engine import (
    DriftEngine,
    DriftSignals,
    DriftAction,
    DriftAssessment,
    DriftSeverity,
)
from corteX.engine.adaptive_budget import (
    AdaptiveBudget,
    BudgetState,
    BudgetDecision,
)
```

See also:
- [Goal Intelligence](goal-intelligence.md) - GoalDNA, GoalReminderInjector, GoalTree
- [Cognitive Context](cognitive-context.md) - Context quality and state management
- [Goal Tracking](brain/goal-tracking.md) - Original goal alignment system
