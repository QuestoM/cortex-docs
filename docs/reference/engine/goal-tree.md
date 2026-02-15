# Goal Tree API Reference

## Module: `corteX.engine.goal_tree`

Hierarchical goal / sub-goal / step structure with weighted progress aggregation, dependency management, stuck detection, and budget enforcement. Three levels: Goal (root) -> SubGoals -> Steps (leaves).

Brain analogy: Prefrontal cortex (hierarchical planning), basal ganglia (sequencing), anterior cingulate cortex (stuck detection).

---

## Classes

### `NodeStatus`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `PENDING` | Not yet started |
| `ACTIVE` | Currently executing |
| `COMPLETE` | Successfully completed |
| `BLOCKED` | Waiting on dependencies |
| `FAILED` | Failed with error |
| `SKIPPED` | Intentionally skipped |

### `NodeLevel`

**Type**: `str, Enum`

| Value | Description |
|-------|-------------|
| `GOAL` | Root level (the overall goal) |
| `SUBGOAL` | Mid-level decomposition |
| `STEP` | Leaf-level executable action |

### `GoalNode`

**Type**: `@dataclass`

A node in the goal tree. Leaf nodes are steps; branch nodes are subgoals or goals.

| Attribute | Type | Description |
|-----------|------|-------------|
| `node_id` | `str` | Unique identifier |
| `description` | `str` | What this node represents |
| `level` | `NodeLevel` | `GOAL`, `SUBGOAL`, or `STEP` |
| `success_criteria` | `str` | How to measure success |
| `estimated_effort` | `int` | Effort weight [1-5] |
| `priority` | `int` | Execution priority (default: 1) |
| `status` | `NodeStatus` | Current status |
| `children` | `List[GoalNode]` | Child nodes |
| `dependencies` | `List[str]` | Node IDs that must complete first |
| `step_budget` | `int` | Maximum steps allowed (default: 20) |
| `steps_taken` | `int` | Steps consumed so far |
| `budget_tokens` | `int` | Token budget |
| `artifacts` | `List[str]` | Produced artifacts |
| `created_at` | `float` | Creation timestamp |
| `completed_at` | `Optional[float]` | Completion timestamp |
| `error` | `Optional[str]` | Error message if failed |

**Properties**:

| Property | Type | Description |
|----------|------|-------------|
| `progress` | `float` | Weighted progress [0.0, 1.0] across children |
| `is_stuck` | `bool` | `steps_taken >= step_budget and progress < 0.8` |
| `is_leaf` | `bool` | No children |
| `is_terminal` | `bool` | COMPLETE, FAILED, or SKIPPED |

**Methods**: `to_dict()` -- serializes node state.

---

### `GoalTree`

Hierarchical goal tracking with weighted progress.

#### Constructor

```python
GoalTree(
    goal: str,
    success_criteria: str = "",
    step_budget: int = 20,
    stuck_threshold: int = 5,
)
```

**Parameters**:

- `goal` (str): The top-level goal description
- `success_criteria` (str): How to measure overall success
- `step_budget` (int): Maximum steps for the root. Default: 20
- `stuck_threshold` (int): Steps without progress to declare stuck. Default: 5

#### Properties

| Property | Type | Description |
|----------|------|-------------|
| `root` | `GoalNode` | The root goal node |
| `total_nodes` | `int` | Total nodes in the tree |

#### Methods

##### `add_subgoal`

```python
def add_subgoal(
    self, description: str, success_criteria: str = "",
    effort: int = 1, priority: int = 1,
    step_budget: int = 20, dependencies: Optional[List[str]] = None,
) -> str
```

Add a sub-goal under the root. Returns the node ID. Effort is clamped to [1, 5].

##### `add_step`

```python
def add_step(
    self, parent_id: str, description: str,
    success_criteria: str = "", effort: int = 1,
    dependencies: Optional[List[str]] = None,
) -> str
```

Add a step under a sub-goal. Returns node ID. Raises `KeyError` if parent not found.

##### `activate_node` / `complete_node` / `fail_node` / `skip_node`

State transition methods. `complete_node` auto-propagates completion to parents when all children are terminal. `activate_node` checks dependencies and sets BLOCKED if unmet.

##### `record_step`

```python
def record_step(self, node_id: str) -> None
```

Increment the step counter for a node (for stuck detection).

##### `get_active_subgoal`

```python
def get_active_subgoal(self) -> Optional[GoalNode]
```

First ACTIVE child of root, or first PENDING if none active.

##### `get_next_pending_step`

```python
def get_next_pending_step(self) -> Optional[GoalNode]
```

Next PENDING step with met dependencies, sorted by parent priority (highest first).

##### `get_stuck_nodes` / `get_blocked_nodes`

```python
def get_stuck_nodes(self) -> List[GoalNode]
def get_blocked_nodes(self) -> List[GoalNode]
```

Detect stuck nodes (over budget, low progress) and blocked nodes (unmet dependencies).

##### `get_summary` / `to_display`

Summary dict and visual tree display (using `[ ]`, `[>]`, `[x]`, `[B]`, `[!]`, `[-]` icons).

---

## Progress Computation

Progress is computed as **weighted average** of children:

```
parent.progress = sum(child.progress * child.effort) / sum(child.effort)
```

Leaf nodes: `1.0` if COMPLETE or SKIPPED, `0.0` otherwise.

---

## Usage Example

```python
from corteX.engine.goal_tree import GoalTree

tree = GoalTree("Build REST API", success_criteria="All endpoints working")

# Decompose into sub-goals
auth_id = tree.add_subgoal("Implement authentication", effort=3, priority=2)
crud_id = tree.add_subgoal("Build CRUD endpoints", effort=2, priority=1)

# Add steps under sub-goals
login_id = tree.add_step(auth_id, "Create login endpoint", effort=2)
jwt_id = tree.add_step(auth_id, "Add JWT token generation", effort=1, dependencies=[login_id])

# Execute
tree.activate_node(login_id)
tree.record_step(login_id)
tree.complete_node(login_id, artifact="auth/login.py")

# Check progress
print(f"Overall: {tree.root.progress:.0%}")
print(tree.to_display())
# [>] Build REST API (33%)
#   [>] Implement authentication (50%)
#     [x] Create login endpoint
#     [ ] Add JWT token generation
#   [ ] Build CRUD endpoints (0%)
```

---

## See Also

- [Goal DNA API](./goal-dna.md)
- [Goal Reminder API](./goal-reminder.md)
- [Planning Engine API](./planner.md)
