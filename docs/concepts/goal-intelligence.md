# Goal Intelligence

Goal Intelligence ensures agents stay aligned with user objectives across thousands of execution steps. It combines O(1) drift detection, adaptive reminders, and hierarchical goal decomposition to prevent agents from going off-track, getting stuck, or forgetting their original purpose.

## What It Does

The Goal Intelligence system provides three core capabilities:

1. **GoalDNA**: Token-set fingerprinting for instant drift detection without LLM calls
2. **GoalReminderInjector**: Adaptive goal reminders that evolve based on conversation progress
3. **GoalTree**: Hierarchical goal decomposition with dependency tracking and stuck detection

Together, these components ensure that even in ultra-long workflows (10,000+ steps), the agent maintains perfect alignment with the user's original intent.

---

## GoalDNA

**corteX Innovation**: O(1) goal drift detection using token-set fingerprinting instead of expensive LLM comparisons.

GoalDNA creates a lightweight "genetic fingerprint" of the user's goal using token sets and character n-grams. This enables sub-millisecond drift detection without any LLM calls.

### Why: The Drift Detection Problem

Traditional goal drift detection uses LLM-based semantic similarity:

```python
# Slow, expensive, requires LLM call
drift_score = llm.compare_similarity(original_goal, current_activity)
```

For a 10,000-step workflow, this would require 10,000 LLM calls just for drift detection. At $0.01 per call, that's $100 in drift detection costs alone.

GoalDNA replaces this with a **deterministic token-set comparison** that runs in <1ms with zero API cost.

### How It Works

GoalDNA uses a dual-layer fingerprint:

1. **Token Layer (70% weight)**: Normalized word tokens → Jaccard similarity
2. **Trigram Layer (30% weight)**: Character 3-grams → Jaccard similarity

```python
from corteX.engine.goal_dna import GoalDNA

# Create DNA from original goal
dna = GoalDNA.from_goal("Build a REST API with JWT authentication and PostgreSQL database")

# Extract fingerprint
print(dna.token_set)
# {"build", "rest", "api", "jwt", "authentication", "postgresql", "database"}

print(dna.trigram_set)
# {"bui", "uil", "ild", "res", "est", "api", "jwt", "aut", "uth", ...}

# Check drift against current activity
activity = "Implementing user login endpoint with bcrypt password hashing"
drift_score = dna.calculate_drift(activity)
print(f"Drift: {drift_score:.3f}")
# 0.623 (some drift - "bcrypt" and "password" not in original goal)

# Check if activity aligns with goal
if drift_score < 0.5:
    print("Activity is aligned with goal")
else:
    print("WARNING: Activity drifting from original goal")
```

### Drift Calculation Formula

```
token_similarity = |goal_tokens ∩ activity_tokens| / |goal_tokens ∪ activity_tokens|
trigram_similarity = |goal_trigrams ∩ activity_trigrams| / |goal_trigrams ∪ activity_trigrams|

drift_score = 1.0 - (0.7 * token_similarity + 0.3 * trigram_similarity)
```

Drift ranges from 0.0 (perfect alignment) to 1.0 (complete divergence).

### Drift Trends

GoalDNA tracks drift history to detect gradual divergence:

```python
# Record drift over time
dna.record_drift(turn=1, drift=0.15)
dna.record_drift(turn=2, drift=0.18)
dna.record_drift(turn=3, drift=0.21)
dna.record_drift(turn=4, drift=0.24)

# Analyze trend
trend = dna.get_drift_trend(window=4)
print(trend)
# {
#   "average": 0.195,
#   "std_dev": 0.032,
#   "slope": 0.030,  # Increasing by 0.03 per turn
#   "is_increasing": True,
#   "velocity": 0.030
# }

# Check if drift is accelerating
if trend["is_increasing"] and trend["velocity"] > 0.02:
    print("WARNING: Drift is accelerating - agent going off-track")
```

### When to Use GoalDNA

- **Every turn**: Calculate drift before executing the next step
- **Before tool execution**: Verify the planned tool call aligns with the goal
- **In drift recovery**: Compare recovery plan against original goal
- **For sub-goals**: Create child GoalDNA for decomposed sub-tasks

---

## GoalReminderInjector

**corteX Innovation**: Adaptive goal reminders that evolve from verbose to compact as the conversation progresses.

The GoalReminderInjector injects goal reminders into every LLM call, with progressive detail reduction based on turn number. This prevents the agent from forgetting its purpose while minimizing context window waste.

### Why: The Memory Fade Problem

In long conversations, agents forget their original goal:

| Turn | Without Reminders | With Static Reminders |
|------|-------------------|----------------------|
| 1-10 | Perfect alignment | Perfect alignment, but wastes tokens |
| 50-100 | Minor drift | Perfect alignment, wastes 50 tokens/turn |
| 500+ | Severe drift | Perfect alignment, wastes 100 tokens/turn |

GoalReminderInjector balances alignment and efficiency with **adaptive detail levels**.

### How It Works

The injector uses three detail levels based on conversation progress:

| Detail Level | Turns | Token Budget | Content |
|--------------|-------|--------------|---------|
| **Full** | 1-5 | ~100 tokens | Full goal text + context + sub-goals + pitfalls |
| **Compact** | 6-15 | ~50 tokens | Goal text + active sub-goal |
| **Ultra-Compact** | 16+ | ~20 tokens | "GOAL: [short summary]" |

```python
from corteX.engine.goal_reminder import GoalReminderInjector

# Initialize injector
injector = GoalReminderInjector(
    full_cutoff=5,
    compact_cutoff=15,
    max_pitfalls=5,
    max_tried=5
)

# Set the goal
injector.set_goal(
    goal_text="Build a REST API with JWT authentication and PostgreSQL database",
    context={
        "language": "Python",
        "framework": "FastAPI",
        "auth_method": "JWT with RS256"
    }
)

# Add sub-goals
injector.add_subgoal("Set up FastAPI project structure")
injector.add_subgoal("Implement user model with SQLAlchemy")
injector.add_subgoal("Create login endpoint with JWT")

# Add known pitfalls
injector.add_pitfall("Don't hardcode JWT secret key")
injector.add_pitfall("Always hash passwords with bcrypt")

# Inject reminder at different turns
reminder_turn_1 = injector.get_reminder(turn=1)
print(reminder_turn_1)
# """
# ═══ PRIMARY GOAL ═══
# Build a REST API with JWT authentication and PostgreSQL database
#
# Context:
# - Language: Python
# - Framework: FastAPI
# - Auth method: JWT with RS256
#
# Sub-goals:
# 1. Set up FastAPI project structure
# 2. Implement user model with SQLAlchemy
# 3. Create login endpoint with JWT
#
# Critical pitfalls to avoid:
# - Don't hardcode JWT secret key
# - Always hash passwords with bcrypt
# """

reminder_turn_10 = injector.get_reminder(turn=10)
print(reminder_turn_10)
# """
# GOAL: Build a REST API with JWT authentication and PostgreSQL database
# Active sub-goal: Create login endpoint with JWT
# """

reminder_turn_50 = injector.get_reminder(turn=50)
print(reminder_turn_50)
# """
# GOAL: Build REST API (JWT + PostgreSQL)
# """
```

### Reminder Placement

Reminders are injected in two locations:

1. **System prompt header**: Before all other instructions
2. **User message footer**: After the user's message (for critical turns)

```python
# Inject into system prompt
system_prompt = injector.inject_system(
    base_prompt="You are a helpful AI assistant...",
    turn=1
)
# "═══ PRIMARY GOAL ═══\n...\n\nYou are a helpful AI assistant..."

# Inject into user message (only for critical turns)
user_message = injector.inject_user(
    message="Implement the login endpoint",
    turn=1,
    force=False  # Only inject if turn is critical
)
```

### Pitfall Tracking

Record mistakes to prevent repetition:

```python
# Agent made a mistake
injector.record_pitfall(
    turn=15,
    pitfall="Hardcoded database password in config.py",
    fix="Moved password to environment variable",
)

# On future turns, reminder includes:
# "Pitfall avoided: Hardcoded database password (turn 15)"
```

### When to Use GoalReminderInjector

- **Every LLM call**: Inject reminder to maintain alignment
- **After compression**: Re-inject full reminder to restore context
- **On drift detection**: Escalate to full reminder temporarily
- **For sub-agents**: Create child injector with sub-goal

---

## GoalTree

**corteX Innovation**: Hierarchical goal decomposition with weighted progress, stuck detection, and dependency tracking.

GoalTree breaks down complex goals into manageable sub-goals and steps, tracks progress with weighted completion, and detects when the agent is stuck.

### Why: The Complexity Management Problem

Complex goals like "Build a production-ready SaaS application" cannot be approached monolithically. GoalTree provides:

1. **Decomposition**: Break big goals into small, achievable steps
2. **Progress Tracking**: Weighted completion percentage
3. **Stuck Detection**: Identify when a sub-goal isn't progressing
4. **Dependency Management**: Ensure prerequisites are met
5. **Priority Scheduling**: Focus on high-value, unblocked tasks

### How It Works

GoalTree is a 3-level hierarchy:

```
Root Goal (weight: 1.0)
├─ Sub-Goal 1 (weight: 0.4)
│  ├─ Step 1.1 (weight: 0.6)
│  └─ Step 1.2 (weight: 0.4)
├─ Sub-Goal 2 (weight: 0.3)
│  ├─ Step 2.1 (weight: 0.5)
│  └─ Step 2.2 (weight: 0.5)
└─ Sub-Goal 3 (weight: 0.3)
   └─ Step 3.1 (weight: 1.0)
```

```python
from corteX.engine.goal_tree import GoalTree, GoalNode, NodeStatus

# Create root goal
tree = GoalTree(
    root_goal="Build a REST API with JWT authentication"
)

# Add sub-goals with weights
api_structure = tree.add_subgoal(
    "Set up FastAPI project structure",
    weight=0.2
)
user_model = tree.add_subgoal(
    "Implement user model with PostgreSQL",
    weight=0.3
)
auth_endpoint = tree.add_subgoal(
    "Create authentication endpoints",
    weight=0.5
)

# Add steps to sub-goals
tree.add_step(
    parent_id=auth_endpoint,
    description="Build /login endpoint",
    weight=0.6
)
tree.add_step(
    parent_id=auth_endpoint,
    description="Build /refresh endpoint",
    weight=0.4
)

# Mark a step as complete
tree.complete_step(step_id=login_step)

# Check overall progress
progress = tree.get_progress()
print(f"Overall: {progress.overall:.1%}")
# 30.0% (login endpoint is 60% of auth, auth is 50% of root)

print(f"Current sub-goal: {progress.current_subgoal}")
# "Create authentication endpoints (60% done)"

print(f"Next step: {progress.next_step}")
# "Build /refresh endpoint"
```

### Stuck Detection

GoalTree detects when progress has stalled:

```python
# Record activity on a step
tree.record_activity(
    step_id=login_step,
    turn=10,
    description="Debugging JWT signature validation"
)
tree.record_activity(
    step_id=login_step,
    turn=11,
    description="Still debugging JWT signature validation"
)
tree.record_activity(
    step_id=login_step,
    turn=12,
    description="Still debugging JWT signature validation"
)

# Detect stuck state
stuck_items = tree.get_stuck_items(
    stuck_threshold_turns=3  # No progress for 3 turns
)

for item in stuck_items:
    print(f"STUCK: {item.description}")
    print(f"  Turns without progress: {item.turns_without_progress}")
    print(f"  Last activity: {item.last_activity}")
    print(f"  Suggestion: {item.suggestion}")

# Output:
# STUCK: Build /login endpoint
#   Turns without progress: 3
#   Last activity: Still debugging JWT signature validation
#   Suggestion: Consider asking user for help or trying a different approach
```

### Dependency Tracking

Ensure prerequisites are met before starting a step:

```python
# Add dependency: can't build /refresh until /login is done
tree.add_dependency(
    step_id=refresh_step,
    depends_on=login_step
)

# Get next available step (considers dependencies)
next_step = tree.get_next_step()
# Returns None if login_step isn't complete yet

# Check if a step is blocked
is_blocked = tree.is_blocked(refresh_step)
# True (waiting for login_step)
```

### Priority Scheduling

GoalTree prioritizes steps by weight × (1 - progress):

```python
# Get prioritized task list
tasks = tree.get_prioritized_tasks(max_tasks=5)

for task in tasks:
    print(f"{task.priority:.2f}: {task.description}")
    print(f"  Blocked: {task.is_blocked}")
    print(f"  Progress: {task.progress:.1%}")

# Output:
# 0.50: Build /login endpoint
#   Blocked: False
#   Progress: 0%
# 0.20: Set up FastAPI project structure
#   Blocked: False
#   Progress: 0%
# 0.00: Build /refresh endpoint
#   Blocked: True (waiting for /login)
#   Progress: 0%
```

### When to Use GoalTree

- **Session initialization**: Decompose the user's goal into sub-goals and steps
- **Every 10 turns**: Check progress and detect stuck items
- **After completing a step**: Update progress and get next task
- **On drift detection**: Re-align current activity with goal tree
- **For planning**: Generate execution plan from goal tree

---

## Integration with Agent Loop

All three components integrate seamlessly into the agent loop:

```python
# In Session.__init__()
self.goal_dna = GoalDNA.from_goal(user_goal)
self.goal_reminder = GoalReminderInjector()
self.goal_tree = GoalTree(root_goal=user_goal)

# In Session.run() or Session.run_agentic()

# 1. Check drift before executing
drift_score = self.goal_dna.calculate_drift(planned_action)
if drift_score > 0.5:
    logger.warning(f"High drift detected: {drift_score:.3f}")
    # Escalate to System 2, inject full reminder

# 2. Inject goal reminder into LLM call
reminder = self.goal_reminder.get_reminder(turn=self.turn_number)
system_prompt = self.goal_reminder.inject_system(base_prompt, turn)
user_message = self.goal_reminder.inject_user(message, turn)

# 3. Update goal tree progress
self.goal_tree.record_activity(
    step_id=current_step,
    turn=self.turn_number,
    description=action_taken
)

# 4. Check for stuck state every 10 turns
if self.turn_number % 10 == 0:
    stuck_items = self.goal_tree.get_stuck_items()
    if stuck_items:
        # Trigger recovery: ask user, try different approach, etc.
        pass
```

---

## Configuration

```python
from corteX.engine.goal_dna import GoalDNA
from corteX.engine.goal_reminder import GoalReminderInjector
from corteX.engine.goal_tree import GoalTree

# GoalDNA configuration
dna = GoalDNA(
    goal="Build REST API...",
    drift_threshold=0.15,
    consecutive_limit=3,
    history_size=200
)

# GoalReminderInjector configuration
injector = GoalReminderInjector(
    full_cutoff=5,
    compact_cutoff=15,
    max_pitfalls=5,
    max_tried=5,
    drift_warning_threshold=0.3
)

# GoalTree configuration
tree = GoalTree(
    goal="Build REST API...",
    success_criteria="All endpoints functional",
    step_budget=50,
    stuck_threshold=5
)
```

---

## API Reference

```python
from corteX.engine.goal_dna import (
    GoalDNA,
    DriftEvent,
    DriftTrend,
    DriftSeverity,
)
from corteX.engine.goal_reminder import (
    GoalReminderInjector,
    GoalReminder,
    GoalProgress,
    ReminderContext,
)
from corteX.engine.goal_tree import (
    GoalTree,
    GoalNode,
    NodeStatus,
    NodeLevel,
)
```

See also:
- [Cognitive Context](cognitive-context.md) - StateFileManager, ContextQualityEngine, CognitiveContextPipeline
- [Anti-Drift](anti-drift.md) - LoopDetector, DriftEngine, AdaptiveBudget
- [Goal Tracking](brain/goal-tracking.md) - Original goal tracking system
