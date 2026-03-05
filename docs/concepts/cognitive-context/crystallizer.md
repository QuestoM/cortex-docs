# Memory Crystallizer

The `MemoryCrystallizer` extracts reusable cognitive patterns - called Crystals - from successful task executions. When the agent completes a task well, the crystallizer distills the goal template, decision chain, tool sequence, and error-resolution pairs into a generalized pattern that can be matched and applied to future similar goals.

## How It Works

### Crystallization Process

When a task completes with a success score above the minimum threshold (default 0.7), the crystallizer:

1. **Generalizes the goal** - Replaces specific values with type placeholders:
    - File paths become `{file_path}`
    - Quoted strings become `"{value}"`
    - Numbers become `{N}`
    - API endpoints become `{api_endpoint}`
    - URLs become `{url}`

2. **Extracts the decision chain** - Pulls all decisions from the execution trace (up to 20)

3. **Extracts the tool sequence** - Collects unique tools used, preserving order (up to 15)

4. **Captures error-resolution pairs** - Only keeps errors that were actually resolved

5. **Detects entity types** - Scans goal and decisions for domain markers: `python_file`, `javascript_file`, `api_endpoint`, `test`, `database`, `deployment`, `authentication`

6. **Novelty check** - Compares against existing crystals using Jaccard similarity on tool sequences (>0.8) and goal templates (>0.7). Duplicates are rejected.

### Query and Application

When a new goal arrives, `query()` finds matching crystals by keyword similarity between the new goal and stored goal templates. Matches are boosted by each crystal's tracked effectiveness score. The `apply()` method formats a matched crystal as a few-shot prompt for context injection, including the approach chain, tools used, and known pitfalls.

### Effectiveness Tracking

Each time a crystal is reused, `record_reuse()` updates its running effectiveness average. This creates a feedback loop: effective crystals get boosted in future queries, while ineffective ones gradually lose prominence and eventually get trimmed.

## Key Features

- **Automatic goal generalization** - Regex-based replacement of specifics with typed placeholders
- **Novelty detection** - Prevents storing near-duplicate crystals (Jaccard similarity check)
- **Effectiveness tracking** - Running average updated after each reuse
- **Entity type detection** - Automatically tags crystals with domain categories (database, deployment, auth, etc.)
- **Capacity management** - Trims least effective crystals when exceeding `max_crystals` (default 200), sorted by effectiveness, success score, and age
- **Few-shot prompt generation** via `apply()` - Formats crystals for direct context injection
- **Zero external dependencies** - All matching is keyword-based, no embeddings or LLM calls

## Integration

The `MemoryCrystallizer` bridges the gap between working memory (what the agent knows during a single task) and long-term memory (what the agent retains across tasks). After a successful execution, the `CognitiveContextPipeline` calls `crystallize()` with the execution trace. Before starting a new task, `query()` finds relevant past patterns and `apply()` injects them into the persistent context zone.

The crystallizer works closely with the `StateFileManager`'s insight layer, using decision logs and error journals as input for the execution trace. The resulting few-shot prompts are injected alongside the state file's persistent context.

## Usage Example

```python
from corteX.engine.cognitive import MemoryCrystallizer

crystallizer = MemoryCrystallizer(
    max_crystals=200,
    min_success_score=0.7,
)

# After a successful task, extract a crystal
crystal = crystallizer.crystallize(
    goal="Deploy user_service.py to production on port 8080",
    execution_trace=[
        {"decision": "Run tests first", "tool": "pytest"},
        {"decision": "Build Docker image", "tool": "docker_build"},
        {"decision": "Blue-green deploy", "tool": "kubectl"},
        {"error": "Health check failed", "resolution": "Increase timeout",
         "tool": "kubectl"},
    ],
    success_score=0.85,
    session_id="task-42",
)
# crystal.goal_template = "Deploy {file_path} to prod on port {N}"
# crystal.entity_types = ["python_file", "deployment"]

# Later, match against a new goal
matches = crystallizer.query("Deploy auth_service.py to staging", top_k=3)
for match in matches:
    prompt = crystallizer.apply(match)
    print(prompt)
    # ## Similar Past Success (0% effective, used 0x)
    # Goal pattern: Deploy {file_path} to prod on port {N}
    # Approach: Run tests first -> Build Docker image -> Blue-green deploy
    # Tools used: pytest, docker_build, kubectl
    # Known pitfalls:
    #   - Health check failed: Increase timeout

# Track effectiveness when reused
crystallizer.record_reuse(crystal.crystal_id, success=True)

# Statistics
stats = crystallizer.get_stats()
# {"total_crystals": 1, "effective_crystals": 1,
#  "total_reuses": 1, "avg_effectiveness": 1.0}
```

## See Also

- [State File Manager](state-file-manager.md) - Provides decision logs and error journals for crystallization input
- [Active Forgetting](active-forgetting.md) - Handles cleanup of crystals that prove ineffective
- [Context Quality Engine](quality-engine.md) - Decision preservation rate reflects crystal-injected knowledge
- [Context Versioner](versioner.md) - Tracks whether crystal-informed decisions lead to better outcomes
