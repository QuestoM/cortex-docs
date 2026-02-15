# Functional Columns

Functional Columns implement cortical column architecture for task specialization. Each column is a coherent processing unit that bundles tools, models, and weight configurations for a specific domain. Columns compete for activation using winner-take-all dynamics with lateral inhibition.

## What It Does

The column system routes tasks to specialized processing configurations. Instead of using the same tools and model for every task, the agent activates the column best suited for the current task type:

| Column | Tools | Model | Weight Overrides |
|--------|-------|-------|-----------------|
| **Coding** | code_interpreter, shell_exec | Quality model | code_density: 0.9 |
| **Research** | web_search, document_read | Quality model | detail_level: 0.8 |
| **Testing** | shell_exec, test_runner | Fast model | speed_vs_quality: 0.5 |
| **Conversation** | (none) | Fast model | verbosity: 0.3 |

## Why: The Neuroscience Inspiration

!!! note "Brain Science: Cortical Columns"
    "In the primary visual cortex there are functional columns, meaning 10,000 cells now fire when I show you this angle, and when these 10,000 cells fire in all of you, you necessarily know that there's such an angle in the world." -- Prof. Idan Segev

    A cortical column is a vertical group of approximately 10,000 neurons (0.5mm diameter) that responds selectively to a specific feature -- an orientation, a frequency, a texture. Key properties:

    - **Winner-take-all competition**: Only one column's interpretation dominates perception at a time
    - **Lateral inhibition**: Active columns suppress neighbors, sharpening selectivity
    - **Hebbian learning**: Columns that succeed get strengthened
    - **Recruitment**: Novel stimuli can recruit unused cortical territory (Merzenich's monkey experiments)
    - **Merging**: When two fingers are surgically joined, their cortical columns merge into one representation
    - **Pruning**: Unused columns lose territory over time ("use it or lose it")

## How It Works

### Column Definition

Each column has a Bayesian competence estimate tracked via a Beta distribution:

```python
from corteX.engine.columns import FunctionalColumn
from corteX.engine.bayesian import BetaDistribution

column = FunctionalColumn(
    column_id="col_coding",
    name="coding",
    specialization="coding",
    preferred_tools=["code_interpreter", "shell_exec"],
    preferred_model="gemini-3-flash-preview",
    weight_overrides={"code_density": 0.9, "detail_level": 0.7},
    competence=BetaDistribution(alpha=10.0, beta=2.0),  # ~83% success rate
)

# Activate the column
column.activate()  # Sets activation to 1.0
```

### Column Competition

Columns compete for activation using Thompson Sampling from their competence posteriors:

```python
from corteX.engine.columns import ColumnCompetition

competition = ColumnCompetition()

# Register columns
competition.register(coding_column)
competition.register(research_column)
competition.register(testing_column)

# Winner-take-all: sample from each column's Beta posterior
winner = competition.compete(task_type="coding")
# coding_column wins (highest sampled competence)
# Lateral inhibition: other columns' activation decays
```

### Task Classification

The `TaskClassifier` maps incoming messages to column specializations:

```python
from corteX.engine.columns import TaskClassifier

classifier = TaskClassifier()

# Uses keyword matching + regex + learned affinities
task_type = classifier.classify("Fix the bug in auth_handler.py")
# "coding"

task_type = classifier.classify("Research the latest OAuth2 best practices")
# "research"
```

### Column Manager

The `ColumnManager` handles the full lifecycle -- registration, learning, merging, and pruning:

```python
from corteX.engine.columns import ColumnManager

manager = ColumnManager()

# Default columns are registered automatically:
# coding, debugging, testing, research, conversation

# After a task completes, update competence
manager.record_outcome("coding", success=True)
# Beta posterior updated: alpha += 1

# Column recruitment for novel tasks
manager.recruit("data_analysis", tools=["python", "matplotlib"])

# Merging: if two columns are always co-activated
# (like Merzenich's joined fingers), merge them
manager.merge("coding", "debugging")

# Pruning: remove columns that have not been used
manager.prune(min_usage=5, min_age_hours=24)
```

### Co-activation Tracking

Columns track how often they are co-activated with other columns. High co-activation suggests the columns should merge (Merzenich-style):

```python
# After many interactions where coding and debugging co-activate:
column._coactivation_counts
# {"debugging": 42, "testing": 15, ...}

# Manager checks for merge candidates periodically
```

## When It Activates

- **On message receipt**: TaskClassifier identifies the task type
- **Before tool selection**: ColumnCompetition selects the winning column
- **During processing**: The winning column's tools, model, and weight overrides are applied
- **After task completion**: Competence posterior is updated
- **Periodically**: Merging and pruning maintain an efficient column set

## API Reference

```python
from corteX.engine.columns import (
    FunctionalColumn,
    TaskClassifier,
    ColumnCompetition,
    ColumnManager,
)
```
