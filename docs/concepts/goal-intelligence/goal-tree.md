# Goal Tree

The Goal Tree provides hierarchical goal decomposition with three levels: Goal (root), SubGoals (branches), and Steps (leaves). Each node carries effort weights, priority, budget, dependency tracking, and status. Parent progress is computed as the weighted average of children, giving the agent and developer a clear picture of what is done, what is next, and what is stuck.

## How It Works

A `GoalTree` is created with a root goal description. Sub-goals are added under the root, and steps are added under sub-goals. Each node is a `GoalNode` dataclass with:

- **Status** - PENDING, ACTIVE, COMPLETE, BLOCKED, FAILED, or SKIPPED
- **Effort weight** - integer 1-5 indicating relative difficulty. Parent progress = weighted average of children.
- **Priority** - integer for ordering. Higher priority sub-goals have their steps processed first.
- **Dependencies** - list of node IDs that must be COMPLETE before this node can activate
- **Step budget** - maximum steps allowed before the node is considered stuck
- **Artifacts** - output artifacts produced by the step

### Progress Computation

For leaf nodes (steps): progress is 1.0 if COMPLETE or SKIPPED, 0.0 otherwise.

For branch nodes (sub-goals, root): progress is the effort-weighted average of children:
```
progress = sum(child.progress * child.effort) / sum(child.effort)
```

This means a sub-goal with two children of effort 1 and effort 3 weights the harder child 3x more in the progress calculation.

### Dependency Management

`activate_node()` checks whether all dependencies are met (all dependency nodes COMPLETE). If not, the node transitions to BLOCKED. `get_blocked_nodes()` returns all nodes waiting on unmet dependencies.

### Stuck Detection

A node is stuck when `steps_taken >= step_budget` and `progress < 80%`. `get_stuck_nodes()` returns all non-terminal stuck nodes for the agent to escalate or replan.

### Completion Propagation

When all children of a node become terminal (COMPLETE, FAILED, or SKIPPED) and at least one is COMPLETE, the parent automatically transitions to COMPLETE.

## Key Features

- **Three-level hierarchy** via `NodeLevel` enum: `GOAL`, `SUBGOAL`, `STEP`
- **Six status states** via `NodeStatus` enum: `PENDING`, `ACTIVE`, `COMPLETE`, `BLOCKED`, `FAILED`, `SKIPPED`
- **Weighted progress aggregation** - effort weights control how much each child contributes to parent progress
- **Priority-based step ordering** - `get_next_pending_step()` iterates sub-goals in priority order
- **Dependency tracking** - nodes can depend on other nodes being COMPLETE
- **Stuck detection** - budget-based detection of nodes making insufficient progress
- **Automatic completion propagation** - parent auto-completes when all children are terminal
- **Display rendering** - `to_display()` produces a human-readable tree with status icons: `[ ]` pending, `[>]` active, `[x]` complete, `[B]` blocked, `[!]` failed, `[-]` skipped
- **Summary statistics** - `get_summary()` returns progress, node counts by status, stuck count, blocked count

## Integration

The Goal Tree connects with the corteX planning and tracking infrastructure:

- **Goal Tracker** - wraps the GoalTree and feeds its progress into the broader brain system
- **Adaptive Budget** - step budget per node provides fine-grained resource control
- **Goal Reminder** - current sub-goal from `get_active_subgoal()` populates the `current_sub_goal` field in reminders
- **Progress Estimator** - tree progress contributes to overall velocity computation
- **Drift Engine** - tree progress drives the goal progress signal

## Usage Example

```python
from corteX.engine.goal_tree import GoalTree

tree = GoalTree(
    goal="Build user management API",
    success_criteria="All CRUD endpoints working with JWT auth",
    step_budget=30,
)

# Add sub-goals with effort weights
sg1 = tree.add_subgoal("Database schema", effort=2, priority=3)
sg2 = tree.add_subgoal("JWT authentication", effort=3, priority=2)
sg3 = tree.add_subgoal("REST endpoints", effort=4, priority=1,
                        dependencies=[sg1, sg2])

# Add steps under sub-goals
s1 = tree.add_step(sg1, "Create users table migration", effort=1)
s2 = tree.add_step(sg1, "Create roles table migration", effort=1)
s3 = tree.add_step(sg2, "Implement token generation", effort=2)

# Work through steps
tree.activate_node(s1)
tree.record_step(s1)
tree.complete_node(s1, artifact="migrations/001_users.sql")

# Check progress
print(f"Overall: {tree.root.progress:.0%}")
print(f"Next step: {tree.get_next_pending_step().description}")

# Display tree
print(tree.to_display())
# [>] Build user management API (33%)
#   [x] Database schema (50%)
#     [x] Create users table migration
#     [ ] Create roles table migration
#   [ ] JWT authentication (0%)
#     [ ] Implement token generation
#   [B] REST endpoints (0%)

# Check for stuck nodes
for node in tree.get_stuck_nodes():
    print(f"STUCK: {node.description}")
```

## See Also

- [Goal DNA](goal-dna.md) - token fingerprinting for drift detection against the goal
- [Goal Reminder](goal-reminder.md) - injects current sub-goal into LLM context
- [Progress Estimator](../agent-intelligence/progress-estimator.md) - velocity-based completion prediction
- [Adaptive Budget](../anti-drift/adaptive-budget.md) - dynamic resource management per step
